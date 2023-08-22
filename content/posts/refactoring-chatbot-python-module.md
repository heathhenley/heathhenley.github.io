---
title: "Refactoring my Simple Python Chatbot Module"
date: 2023-08-17T23:40:00-04:00
draft: false
description: "I wrote a chatbot in python to impersonate one of my friends on Slack. It was a fun project, but originally I set it up a pretty hacky way. Recently, I decided to refactor it to make it more maintainable and easier to add new features. There's no real reason to use this module instead of LangChain, but it's a simple example that I hope will help somewhere learn."
tags: ["python", "software"]
categories: ["software"]
keywords: ["python", "chatbot", "refactoring", "openai", "solid"]
---

*TL;DR: cleaned up a simple chatbot module that I wrote in Python, originally
as a way to impersonate a friend in my Slack chat.*

## How it started
I wrote a chatbot in python to impersonate one of my friends on Slack. It was a
fun project, but originally I set it up a pretty hacky way.  Here's an example
of what it looks like integrated into slack:

![Example of chatbot in slack](/chatbot/slack.png)

Ok so that's a pretty goofy impersonation of a friend of mine (littered with
inside jokes) - but it can actually be useful. For example, I used the same 
basic module to create a chatbot that acts as an expert on 3D forward looking
sonar data in my work Google Chat. It uses the
[FarSounder Tech Blog](https://www.farsounder.com/blog) as a knowledge base. Here's an example of that:

![Example of chatbot in google chat](/chatbot/gchat.jpg)

## Refactoring
### Why?
Recently, I decided to refactor it to make it more maintainable, readible, and
easier to add new features. You can see what it looked like before refactoring 
in the git history [here](https://github.com/heathhenley/ChatGPTBot/blob/671551e85270a786e3cf396d8ee88db7a529bd46/chatbot/chatbot.py#L45).

### What was the problem?
Basically, in my opinion, the biggest problem was that there was a lot going on 
in that ChatBot class. It was responsible for taking the users prompt, looking
for similar data in the database using vector similarity, storing the chat
message history, and calling the OpenAI API to generate a response. This
made it really hard to add new features, as it required a lot of understanding
of how everything worked together, and a lot of modification. More or less this
violated the Open/Closed principle of SOLID design - it was hard to extend the
functionality without modifying the existing code. Instead, I wanted to make it
open for extension, but closed for modification.

### The approach to fix it
To clean it up, I took the approach of breaking the class up into smaller
classes that each have a single responsibility, and passing them into the
chatbot class as dependencies (dependency injection). So that meant splitting
out the "chat history storage" functionality and the "knowledge base"
functionality into their own classes.

However, to make it even easier to extend
the functionality of the chatbot, I introduced base classes for each of those
that define their required interface. So the chatbot knows that it's getting a 
class for message history, and a class for managing the knowledge base, but it
doesn't care exactly what they do as long as they implement the required 
interface. With this approach, it's really easy to add new functionality to the
chatbot, just implement your own custom child class of either the message 
history class or the knowledge base class to add your own functionality, and
then pass it into the chatbot class when you instantiate it.

For example, if you wanted to store the chat history in a Postgres database so
that it could be persisted, instead of stored in memory, you could create a
class that inherits `BaseMessageMemory` class and implements the required 
interface, and then pass that into the chatbot class when you instantiate it,
and no other changes would be required: open for
extension, closed for modification.

You can see refactored version [here](https://github.com/heathhenley/ChatGPTBot/blob/main/chatbot/chatbot.py).

If you want to try it out, clone the repo and run:
```python
>>> from chatbot import chatbot
>>> import os
>>> bot = chatbot.ChatBot(api_key=os.getenv("OPENAI_API_KEY"))
>>> bot.get_reply("What's up there chatbot?")
"Hello! I'm here to help you with any questions or problems you may have. How can I assist you today?"
```

With a default prompt and no knowledge base, it's not very interesting - more
or less it's just a wrapper around the OpenAI API with a history of the last 5
messages stored memory. But, you can easily add your own knowledge base
and customize the prompt to make it more interesting. There's an example in the
repo of how to use it with a knowledge base of this blog, a custom prompt,
and FastAPI to create a rest API that will let you ask questions about this
blog. You can see that example [here in the repo](https://github.com/heathhenley/ChatGPTBot/tree/main/examples/fast_api) and there's a version
of the API [here](https://heathblogbot.up.railway.app/docs) that you can try
out.

## That's it
I mentioned before, if you're going to implement anything based on using a
LLM in python, you really should probably use
[LangChain](https://github.com/langchain-ai/langchain) - but hopefully this simple example helps someone.
