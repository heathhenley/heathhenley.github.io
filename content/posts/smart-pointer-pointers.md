---
title: "Smart Pointer Pointers"
date: 2022-05-04T10:05:29-04:00
draft: false
---
*TL;DR - Switch raw heap pointers to `unique_ptr`s when possible. If your heap
allocated resource needs multiple owners, use a `shared_ptr` instead.*

I find myself reviewing C++ smart pointer types over and over again. I guess for
me, it’s just been one of those things that doesn’t stick, or perhaps I’ve only
needed to use them infrequently enough to forget about them. So, here’s a note
to myself, and anyone else dying to read about smart pointers in C++. This is
not something that a veteran "hardcore" C++ programmer will gain anything from
reading. Though if you’re a a bit of a generalist like I am and you find
yourself switching between projects and languages, maybe you’ll find this
useful. Honestly, I’m hoping that through writing it up, I’ll never have to look
it up again :)

I must say, that all of the information below, including the examples, is
summarized from the Microsoft C++ docs, in particular, the section on [Smart
Pointers](https://docs.microsoft.com/en-us/cpp/cpp/smart-pointers-modern-cpp?view=msvc-170),
so if you need more detail or examples, please check that out.

Having full control over whether resources are allocated on the stack or heap is
a big advantage but also a can be a huge pitfall for programmers writing C++
code. Heap allocated resources need to be explicitly deleted to release the
underlying memory, so the programmer needs make sure that happens in the right
place, and when it’s appropriate, and also under any failure cases. In modern
C++, smart pointers can be used to help avoid any headaches associated with
managing raw pointers to heap allocated objects or variables. They basically
help make sure that the resource is always deleted / freed and or cleaned up
when necessary. They are stack allocated wrappers around the raw pointer to the
heap allocated resource in question, and there are a few different types with
different use cases. 

The available types of smart pointers in the Standard Library are:
  - `unique_ptr`: allows exactly one owner of the underlying heap allocated resource. The
underlying resource is cleaned up when the (stack allocated) `unique_ptr` goes out
of scope.
  - `shared_ptr`: kind of like `unique_ptr` but reference counted, so
the underlying resource is not cleaned up until all of the references are out of
scope.
  - `weak_ptr`: allows access to resource pointed at by a `shared_ptr` but
doesn’t increase the reference counter, so it won’t cause it to be kept alive if
it goes out of scope everywhere else. I’ve never needed to use this one, but
it's good to know about.

The following example from the docs demonstrates the difference
between allocating a raw pointer and smart pointer.

```cpp
void UseRawPointer()
{
    // Using a raw pointer -- not recommended.
    Song* pSong = new Song(L"Nothing on You", L"Bruno Mars"); 

    // Use pSong...

    // Don't forget to delete!
    delete pSong;   
}


void UseSmartPointer()
{
    // Declare a smart pointer on stack and pass it the raw pointer.
    unique_ptr<Song> song2(new Song(L"Nothing on You", L"Bruno Mars"));

    // Use song2...
    wstring s = song2->duration_;
    //...

} // song2 is deleted automatically here.

```

In the `UseSmartPointer` snippet, the `song2` variable is stack allocated and
manages access to the heap allocated `Song` object. In this smart pointer case,
when `song2` goes out of scope, its destructor handles the clean up of the heap
allocated `Song` object. If we were using a raw pointer, as in the UseRawPointer
case, we would be on the hook for managing this cleanup. The `unique_ptr`  is
the smart pointer I end up needing most of the time.

Initialization of a `shared_ptr` can look very similar (note the docs do
recommend using
[`make_unique`](https://docs.microsoft.com/en-us/cpp/standard-library/unique-ptr-class?view=msvc-170)
or
[`make_shared`](https://docs.microsoft.com/en-us/cpp/standard-library/memory-functions?view=msvc-170#make_shared)
for initialization instead when possible) This container holds a reference
counter along with the raw pointer to the heap allocated resource. When
`shared_ptr`s that share ownership with an underlying resource go out of scope,
the reference count will be decreased. When no other references to the
underlaying remain, so the reference count gets to zero, the resource will
finally get deleted. So if we switched the `unique_ptr` for a `shared_ptr` in
the example above, the behavior would be the same. There is only one reference
so our heap object gets cleaned up when the smart pointer goes out of scope.

When `shared_ptr`s are passed to functions or methods, they behave differently
depending on whether they are passed by value or reference. This is important to
keep in mind! If passed by value, the copy constructor is called and the
reference count is increased, so the method or function receiving the
`shared_ptr` is now an ower also. If passed by reference, instead the reference
count is not incremented and the “callee” is not an owner. So in this case
access to the underlying object is only available when the caller is in scope. 

Summary In my experience, when updating or interacting with legacy code, the
most common and straightforward switch is using `unique_ptr`s instead of raw
ones, which makes clean up logic super simple. The `unique_ptr` can only have a
single owner, and can’t even be passed by reference in a method or function
(this would call the copy constructor, and it cannot be copied). Of course,
having one owner of a resource is good practice, so the docs recommend using
this option whenever it’s possible.  The `shared_ptr` use cases can be a little
more complex, because if a `shared_ptr` is required, you’ve necessarily decided
that multiple components of the system need to own the underlying resource.
Essentially asking whether both (or multiple) users of the resource need
guaranteed access to it in order to perform their function (eg should they
become owners?) helps to determine which smart pointer to choose (and how we
should pass it if it’s a `shared_ptr`).

The docs on these smart pointer types for Visual Studio C++ are extensive and
contain a ton of examples. Check
[them](https://docs.microsoft.com/en-us/cpp/cpp/smart-pointers-modern-cpp?view=msvc-170)
out if you're ready to dig deeper. Hopefully now that I’ve written this up, I
won’t find myself digging through them next time!
