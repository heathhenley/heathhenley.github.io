---
title: "Writing in Target Language Everyday and Writescribir"
date: 2023-10-12T08:00:26-04:00
draft: false
metathumbnail: "/writescribir/twitter-card.png"
description: "I think that writing in your target language everyday is a great way to improve your writing skills! So much so that I'm working on an app to help with that. Check it out at writescribir.com."
tags: ["language learning", "writescribir"]
categories: ["language learning", "writescribir"]
keywords: ["language learning", "ESOL", "writescribir", "writing", "spanish", "english"]
---

TL;DR: I think that writing in your target language everyday is a great way to 
improve your writing skills! So much so that I'm working on an app to help with
that. Check it out at [writescribir.com](https://writescribir.com/en/about).

I think that writing in your target language everyday is a great way to improve
and to keep your new language on your mind, especially when you don't live in a
country where your target language is spoken (or primarily spoken). You 
definitely don't need my app to do this! Check out any of the /r/WriteStreak 
subreddits for a community of people who are doing this (eg [r/WriteStreakES](https://www.reddit.com/r/WriteStreakES/)). So there's clearly at least some
people who agree with me that daily writing is worthwhile!

I've been teaching a free English class for Spanish speakers for a few years
now, and I've been trying to get my students to write more consistently, even
if it's only a few sentences a day. Of course there are a lot of ways to do
this, but I want to make something that will be easy to use, simple, and 
provide some provide some useful features. So I'm starting on [Writescribir](https://writescribir.com/en/about)!
The goal with Writescribir is to offer a simple prompt, everyday, to get you to 
produce some writing in your target language consistently. I added a few features, and I'm working on 
more, to make it more useful than just using Reddit, a prompt generator, or a 
journal. For example, I've already added automatic translation of your writing, 
and an automatic AI based correction bot to correct any new
answers as they are posted. I'm hoping to add a way to save words and phrases 
so they can be exported into a spaced repetition system like Anki, and some type
of text to speech so you can hear your writing read back to you / practice
pronunciation.

If you have any other ideas for features, please let me know!

## Technical Details

Writescribir is a [Next.js](https://nextjs.org/) app, which is a React framework
that I really like and have been learning. It's hosted on 
[Vercel](https://vercel.com/). The backend is [Convex.dev](https://www.convex.dev/), which is another new platform that I was excited to test out.
Convex provides a document database, and you can easily write serverless 
functions and actions in the same project that can access / modify the database. It's also possible to run serverless functions as cron jobs to run
actions on a schedule. Actually, I'm using that approach to generate the daily prompts for Writescribir! Everyday at 0:00 UTC, a cron job runs a serverless function to request a prompt from OpenAI. After a bit of tuning, I was able to get it to generate some pretty good prompts. I'm also using the OpenAI API to generate the automatic corrections for the user's writing - it works well but
it's still sometimes making unnecessary or wrong corrections. The automatic translations come from [DeepL API](https://www.deepl.com/docs-api/), which also runs as a serverless action. I'm using the free version with a limit of 500,000 characters per month - hopefully should be more than enough for my purposes!

Styling on the front end is something I'm still working on. I'm using [TailwindCSS](https://tailwindcss.com/) and [shadcn/ui](https://ui.shadcn.com/) for some components, but honestly I'm still learning design / UX / UI stuff.

## Conclusion
Check it out, and let me know if you have any suggestions. Either way, I hope you practice your English or Spanish everyday!