---
title: "Switching to the Python Pathlib Module"
date: 2020-03-15T20:32:39-04:00
draft: false 
tags: ["python3"]
categories: ["software development"]
keywords: ["python", "python3", "pathlib", "os"]
description: "Using Python's pathlib module instead of os.path to manipulate file system paths can make your code more readable and easier to maintain, with a couple examples of how to use it."
---
It’s been about a year or so since we officially upgraded all of our tooling at
my job [at FarSounder](www.farsounder.com) from Python 2.7 to Python 3 (3.6 at
the moment). Aside from the syntactic changes, there have been a handful of
updates in Python 3 that I’ve found to really increase the readability of our
scripts. One of those updates (from back in Python 3.4) has been the
introduction of the [`pathlib`](https://docs.python.org/3/library/pathlib.html)
module.

The `pathlib` module contains high level classes that represent file system paths
- and I’ve mainly been in using it cases where I would have previously used the
[`os`](https://docs.python.org/3/library/os.html) module. Here are some examples
of the most common uses that I’ve seen occurring in the wild, and how they can
be implemented using both modules.

## Path Construction
In this case, the goal is to store a path to the `test_dir` directory, that lives
in the arbitrary hard-coded path in my system. An example of how to do this
using both modules is:

``` python
# Construct the paths
# 1. Using the os module and os.path.join
ospath = os.path.join("c:\\", "dev", "sandbox-heath", "pathlib_test", "test_dir")
# 2. Using the pathlib module and pathlib.Path (and the overloaded "/")
plpath = pathlib.Path("c:\\") / "dev" / "sandbox-heath" / "pathlib_test" / "test_dir"

```
Many of the scripts I work with day-to-day deal with opening, manipulating and
writing other files, so the `os` module is used a lot. Specifically, `os.path.join`,
used to create a valid path from the variable number of path components (given
as one or more arguments to the `os.path.join` method). It works great, but it’s a
little verbose, especially as the number of path components gets larger.

The `/` operator is implemented for `Path` objects, so might have noticed above,
that path components can be joined using a `/` in the `pathlib.Path` version. I
personally really like this syntax because of similarity to an actual file
system path - mentally it feels easier to parse. I would be interested to hear
what you think, especially if you haven’t seen it before.

## Iterating Over Directory Contents 
Another super common pattern when working with the file system is iterating over
files in a given directory. Here’s an example of how to do this using both
modules:

``` python
# Loop over files in a directory
for f in os.listdir(ospath):
  full_path = os.path.join(ospath, f)
  print(F"Filename: {f}")
  print(F"Fullpath: {full_path}")
 
for f in plpath.iterdir():
  print(F"Filename: {f.name}")
  print(F"Fullpath: {f}")
```

The example above iterates over files in the directory, and prints the filename,
along with the full path to the file - so both loops above output:

```
c:\dev\heath\pathlib_test>py -3.6 pathlib_test.py
Filename: a.dat
Fullpath: c:\dev\heath\pathlib_test\test_dir\a.dat
Filename: a.py
Fullpath: c:\dev\heath\pathlib_test\test_dir\a.py
Filename: b.dat
Fullpath: c:\dev\heath\pathlib_test\test_dir\b.dat
Filename: b.py
Fullpath: c:\dev\heath\pathlib_test\test_dir\b.py
```

Looping over the files in the directory is simple in both cases, however some of
utility of the Path object is starting to show through - first we can iterate
over the directory using the built-in `iterdir()` method and given that it is
yielding Path objects itself, we can use this objects `.name` to print the name of
the file, and it’s `str` representation to print the full path.

Often, there is a requirement to iterate only over a specific file type in a
directory. An option for this using each method is given below:

``` python
# Looping over specific files (.py for example)
# 1. Using os module
for f in os.listdir(ospath):
  full_path = os.path.join(ospath, f)
  name, ext = os.path.splitext(f)
  if f.endswith(".py"):
    print(F"Found a python file: {f}")
    print(F"Name: {name}, Ext: {ext}")
    print(F"Fullpath: {full_path}")
 
# 2. Using pathlib module
for f in plpath.glob("*.py"):
  print(F"Found a python file: {f.name}")
  print(F"Name: {f.stem}, Ext: {f.suffix}")
  print(F"Fullpath: {f}")
```
This results in the same output from each loop, and depending on the contents of
your `test_dir`, looks something like:

```
Found a python file: a.py
Name: a, Ext: .py
Fullpath: c:\dev\heath\pathlib_test\test_dir\a.py
Found a python file: b.py
Name: b, Ext: .py
Fullpath: c:\dev\heath\pathlib_test\test_dir\b.py
```
Again, both methods are pretty similar - but in the `pathlib` version, it’s
convenient to take advantage of `pathlib.Path`‘s `glob()` method to iterate over
only the files we want. Further, when we need to work with the file,
`pathlib.Path` offers a ton of useful properties that are a lot simpler than the
`os` version.

## Conclusion
There is a lot more you can do with this module, and you can read more about
this in the [Python docs](https://docs.python.org/3/library/pathlib.html). There
are a ton examples and a lot of useful information there.

I don’t think that there would be a huge upside to rewriting / refactoring
existing scripts to use `pathlib` instead of `os`, and I’m certainly not advocating
that. However, I know that due to the ease of manipulating file system paths
using `pathlib`, I am definitely going to default to using it whenever I’m
implementing a new script or adding a significant chunk of functionality to an
existing one.

What do you think?  Is it really cleaner and easier to read? Or is it just me?
Can you think of cases where you would still prefer to use the underlying os
module instead?
