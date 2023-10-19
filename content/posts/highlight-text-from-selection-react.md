---
title: "Highlight Text From Multiple User Selections in React"
date: 2023-10-19T18:06:57-04:00
draft: false
metathumbnail: "/writescribir/highlights.png"
description: "A simple React component that allows you to highlight text from multiple user selections."
tags: ["react", "typescript", "javascript"]
categories: ["react", "typescript", "javascript", "programming", "web development", "software development"]
keywords: ["react", "typescript", "javascript", "programming", "web development", "software development", "highlight selected text", "highlight multiple user selections in react"]
---

**TL;DR** - This is a simple React component that allows you to make multiple
highlights in a text from user selections.

## Why?
Working on a [side project](https://writescribir.com) I wanted a way for users
(and me as the current only user) to save words that they don't know from
a text. I found this [great example blog](https://medium.com/unprogrammer/a-simple-text-highlighting-component-with-react-e9f7a3c1791a) that demonstrated 
one way to do this in React, but it didn't allow for multiple highlights. So I 
took the proposed approach, generalized it a bit for multiple highlights, and 
the added it into my project.

In the same spirit as the original poster, I wanted to share it here in case it's useful to anyone else.

## How?

The idea is simple, and taken from the article linked above:
- Get the text that the user selected
- Wrap the selected text in a span, and apply styles that highlight the text

The approach in the linked article fell short for my application because it
assumed that there would only be one highlight. It gets a little bit trickier
dynamically adding spans because the selected text offsets for a given selection
need to be adjusted for each highlight that came before that selection.

The component is used like this:

``` typescript
import HighlightableText from "./components/HighlightableText";

const dummyText =
  "This is a bunch of dummy text to test highlighting! This is a bunch of dummy text to test highlighting. Test text to highlight. This is a bunch of fake test to test highlighting. Test text to highlight. This is a bunch of fake test to test highlighting.";

function App() {
  return (
    <div className="App">
      <HighlightableText text={dummyText} />
    </div>
  );
}
```

The component itself is a little longer, but it's available on [GitHub](https://github.com/heathhenley/HighlightableText) wrapped in 
a barebones Vite project that just loads the component as above, so that you
can spin it up, test it out, and start highlighting text.

## What's Next?

It's still got a problem - I would like to be able to handle the case where a
selection is made that overlaps with a previous selection. Right now, it just
doesn't allow it. In my app for now, I just show a toast that says "Overlapping
selections are not allowed" - but it would be nice to be able to handle that
case.

It would be nice to get it to work on Mobile too - I tried adding the `onTouchEnd` event handler and calling the same handler function, and it does
work in general - but it's not quite right, at least the way I want to use it.
When you select on mobile, you need to hold and select the text, and you can then drag the selection handles to adjust the selection. The problem is that the first word that you touch when you touch to start the selection will also be highlighted. Which actually is ok, but it's annoying when you want to call a
handler with the word that is selected, you'll now have two separate 
highlights (on from the initial onTouchEnd and one from the second). I guess 
adjacent highlights could be merged in the backend...I plan to try some things 
and see how it goes!

## That's it!

Let me know if you have any feedback or suggestions!