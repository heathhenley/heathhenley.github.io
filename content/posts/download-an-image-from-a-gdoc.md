---
title: "How to Download an Image from a Google Doc"
date: 2023-03-19T23:20:05-04:00
draft: false
tags: ["notes", "google docs", "random", "tips"]
keywords: ["random", "google docs", "google", "image", "download"]
---
*TL;DR: Either "File --> Download" the Google Doc as a web page, or open developer tools and watch for the browser to request the image from Google's file servers.*

# The deets
Here are two simple ways to get an image out of a Google Doc, at least until Google (maybe inevitably?) adds a “right click-> save image as” feature. 

The first way is pretty straightforward, download the Google Doc containing the image files you would like to extract as a web page (File → Download → Web page (.html, zipped)). The image files will be contained in the downloaded .zip file. Extract the zip and the images will be in an “images” directory.

The second way to grab the image is to open developer tools in your browser of choice (Chrome and Chromium based browsers: `CTRL + SHIFT + I`), and navigate to the “Network” tab. We’re going to watch for the browser to request the image, and then we can use that link to download the image directly.

![Network Tab in Dev Tools](/gdoc/gdoc_1.png)

What you’re seeing here are all the requests the browser is making the backend(s) associated with the app. To sync our Google Doc with the server and any other connected clients (using [Operational Transformation](https://operational-transformation.github.io/) = which is pretty interesting to read about but not relevant here at all 🤓). 

While you have this panel open, just refresh the page - you will see the requests get fired off to the server, one of those will be a GET request for your image - you. It might help to sort by type - here’s an example where the .png screenshot above was included: 

![Network Tab in Dev Tools](/gdoc/gdoc_2.png)

There’s going to be a bunch of image requests, for your image but also all the icons, your avatar, etc, so click around until you find the request that corresponds to the image you want (use “Preview” after clicking the link with the request to check). Once you’ve found it, right click and open that link in a new tab and you can download your image!

# Is there a better way?
Let me know if I missed a simple way to download an image from a GDOC - I’m sure there is one and I was surprised that "right click -> save image as" isn't an option. 
An even better solution might be to keep the raw images handy in the same folder where you keep your Google Docs, but just in case that doesn't happen, I hope this helps someone out there save some time!