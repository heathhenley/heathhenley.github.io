---
title: "My notes from implementing two similar apps with dissimilar tech stacks"
date: 2024-08-21T06:12:27-07:00
draft: false
description: "Notes on implementing two similar apps that interact with NOAA
bathymetry data endpoints using dissimilar tech stacks - the apps used FastAPI/HTMX and Next.js, respectively."
tags: ["software", "notes", "web development", "full stack"]
categories: ["software", "notes", "web development", "full stack"]
keywords: ["nextjs", "fastapi", "htmx", "web development", "full stack", 
"typescript", "notes"]
---

**TLDR** - My general takeaways two similar apps. One the OG 
server-returns-html way with FastAPI/HTMX and one with Next.js using the app 
router / RSC / SSR. These are my quick take aways from the point of view of 
working on them from the perspective of a non-web dev.

## Intro
I am currently working on two somewhat similar web apps, one using
[FastAPI](https://fastapi.tiangolo.com/) with [HTMX](https://htmx.org/) and one
using [Next.js](https://nextjs.org/). The first
([https://newdepths.xyz](https://newdepths.xyz)) is an app to receive email
notifications when new geographic data is available in a user provided area at
some public endpoints (bathymetric data hosted by NOAA), and the second ([CSB
Data Explorer](https://mycsb.farsounder.com)) displays subsets of similar
geographic NOAA data on a map viewer with some stats. The notification app uses
a FastAPI server with Jinja2 templates and HTMX, and the viewer app uses Next.js
app router.

## Why I chose these stacks?
**Next** - I wanted to play around more with Next and force myself to learn more
about React server components. I like how easy it is to deploy to triangle
company and this app will never have enough traffic to really cost anything.

**FastAPI/HTMX** - I wanted to play around with HTMX and how it works as it has
gained fame via excellent meme skills over the last years. I used FastAPI not
really because I needed any of the async stuff, but because I wanted to roll
everything (auth / db / worker, etc) for funsies (yes I know that other
frameworks have *more* batteries included). I was also originally planning to
make a standalone api for this app but I've cut that out for now as I don't need
it anymore.


## Quick Takeaways

> **Note** - *There is not complete feature parity between the two apps, these
were two separate projects that I wanted to build for their own purposes. I only
chose to use different stacks for my own learning and curiosity. Someone more
interested in this comparison might want to implement the exact same app using
the each stack for a more direct comparison.*

My biggest takeaways so far:
* **Server / client boundary is not always clear in Next.js** - The
client / server boundary in a full stack next app is just not always immediately
clear, especially to a non-web developer. It’s not that bad and the [docs are
good](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns)
at explaining how to compose server rendered components (to fetch data for
example) with client components (for user interaction). Still, it’s not as
obvious as working on the FastAPI/HTMX app (or a JSON HTTP API server / react
client, or django, laravel, rails, etc) set up where your server code just
defines handlers that return rendered html for your specific endpoints.
* **Updating different areas of the page when user interacts is harder in the FastAPI/HTMX app** -
This got me while working with the FastAPI/HTMX app. Doing something /
re-rendering something somewhere else on the page when the user did something is
not as obvious. This is straightforward in a react app - move state up to the
common parent of the components that need to reflect the change and pass to them
as props (or use context). To handle this in FastAPI/HTMX, I am only aware of
using an [out of band swap](https://htmx.org/attributes/hx-swap-oob/) or using a
custom event that you trigger with [response
headers](https://htmx.org/headers/hx-trigger/). For example, I used a custom
event to delete orders for a user selected area from the UI when that area was
deleted, and to show toasts on certain user actions. Maybe a skill issue, but
also not something the dev even needs to really think about in a react app.  
* **Need to check headers and set vary header for any endpoints that return
partials for HTMX** - Any endpoints that you use to return partials for HTMX to
swap in should also be able to return the full page (or a full page) in case
it’s requested by the browser navigating there instead of by HTMX. This isn’t
difficult or anything, but something to consider and it was unexpected dealing
with it as noob. You can check the headers to the “hx-request” header to
determine if it’s a request coming from HTMX or not, and add a “vary” header to
the response so that the browser won’t cache partials and full page responses
the same (which could cause it to accidentally show a partial on a back/forward
navigation, for example). Again, both easy fixes, but gotchas.
* **Next caching defaults** - I don't think I've totally figured this out yet,
but I had to play with the default caching setup for the data that my server
components were fetching, as it's a regular get request but the data returned
updates periodically. Not a big deal, but something that had me scratching my
heading for a bit as I was trying to figure out why my data wasn't updating.

My preference so far - it depends 🤣. I like the control and simplicity of a
classic server-renders-html app using HTMX for some interactions. I personally
like the very clear client/server boundary in these cases too. On the other
hand, using Typescript/Javascript for the whole stack is nice, no real context
switching there and it's definitely easier if there's a 'lot' of complex user
interaction. 

I feel like to use HTMX and a server-rendered app effectively, you need to have
a good understanding of web dev in general, while to use Next.js effectively,
you need to have a good understanding Next.js specifically. I realize I also
wasn't really using a framework like Django, Rails or Laravel for the FastAPI /
HTMX app, so that might have shifted the scales a bit too.