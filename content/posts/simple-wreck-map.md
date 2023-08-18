---
title: "New England Shipwreck Map"
date: 2022-12-22T15:28:45-05:00
draft: false
tags: ["python3", "python", "django", "project", "leafletjs"]
keywords: ["shipwrecks", "new england shipwrecks", "map", "sonar"]
description: "A map of the shipwrecks in Narragansett Bay, Rhode Island, and really the greater New England area in general. The data is aggregated from two sources, and plotted on an interactive map using Leaflet.js."
---
**TL;DR** - Here's a list of shipwrecks plotted on an [interactive map](https://www.neshipwrecksmap.com), all over the Narragansett Bay and greater New England area.

## Motivation
At [FarSounder](https://www.farsounder.com) I work on the development software of 3D Forward Looking Sonar products. Part of that development process of course includes collecting data from a lot of different situations and running it through some processing algorithms to evaluate performance and make improvements. To that end, we're often out on the Narragansett Bay collecting data with the system, and always looking for areas in the bay with interesting features that we can use to test and benchmark our algorithms (pilings, super steep shoals, rock piles, piers, etc).
Some of the most historically interesting features that you can find in the bay are shipwrecks - they are also pretty nice to use as sonar targets in the development of processing algorithms.

There's a pretty well known wreck in the bay, just south of Bull Point. It's the 1920 wreck of the 267 foot freighter vessel "Cape Fear". It's a large and mostly intact wreck, but is a difficult dive due to vessel traffic, tide, and depth. At least that's what I hear, I only investigated it from the surface.

Below is a pretty cool side scan sonar image of the wreck (copyright Mark Munro, I retrieved this image from Wreckhunter).

![Sidescan sonar image of the wreck of the Cape Fear](/wreck_app/capefear_ss_mark_munro.jpg)

Still, we've been there a number of times and collected some really valuable and interesting data sonar data, so I went searching around for a map of what other wrecks might be within range our engineering tests. While I was able to find some resources that extensively documented the wrecks in the Narragansett Bay area (and beyond), there was not a great of searching them visually. I've worked with a lot of GIS data, so I pulled in the wreck info from the two most comprehensive resources I could find.

## Where's the data from?
All of the wreck data came from two sources: (1) the diving focused website [WreckHunter](https://www.wreckhunter.net) and (2) the comprehensive historical database of shipwrecks compiled by the [Beavertail Lighthouse Museum](https://www.beavertaillight.org/) and maritime historian Jim Jenney. Both of these sources have all the rights to the data, so if you're interested in using their data please check with them individually concerning any limitations on use.

## How is it set up?
It's open source - so you can check it out for yourself on [GitHub](https://github.com/heathhenley/Shipwrecks) if you're interested in some more of the details. In brief, the site is just a single page [Django](https://www.djangoproject.com/) app running on [Heroku](https://www.heroku.com/), the backend is a simple little sqlite database at the moment, which is lightweight but makes it easy to use Django's ORM and migration utilities in case the project expands. The frontend is just mostly using [Leaflet.js](https://leafletjs.com/) to display the map, but I pulled some additional overlay and basemap layers from [NOAA](https://www.ncei.noaa.gov/maps/bathymetry/) and [FarSounder](https://www.farsounder.com/blog/expedition-sourced-data-collection-program-progress-update).

## What's Next?
The current app is simple, but I think it's a pretty cool way to visualize the wrecks in the Narragansett Bay (and greater New England to some extent). Of course it would be great to add more wreck data and expand to include data from wreck locations. For example, here's a screenshot zoomed in on the wreck of the Cape Fear: 

![Zoomed in on bathymetric layers new Cape Fear](/wreck_app/cape_fear_bathy_both.png)

This was a large vessel and you can see the depth difference in the middle of the channel due to the presence of the wreck in the NOAA MBES and FarSounder 3D FLS bathymetry layers. If you were to uncheck the bathymetry layers, you would also see the marking for the wreck on the ENC chart.

So from a sonar testing standpoint, I'll be my own first user 😀. We'll now be able to look at wreck locations, crosscheck them with NOAA MBES data in the area to see if there may be anything substantial still on the seafloor there, and then finally plan our engineering tests to go collect data in those areas.

As far as continued development of this app, I am also thinking about introducing some API endpoints that return wrecks, location, and links to their references as a JSON response, potentially based on a location passed in with the request. For example: a user could programmatically request all of the wrecks within some miles of a location. Django has a REST API framework, but this might also be a fun way to check out [FastAPI](https://fastapi.tiangolo.com/) - TBD.

I also think it would be cool to stop using Django template language with pure Javascript for the front end, and instead use Django (or something else eg FastAPI) only for the backend. Then the main frontend client could be written in React (which I've been hoping to jump into for a while) - and it could just use the mentioned REST API to get data.

Posting notes, comments, or warnings on a given wreck could be potentially interesting to divers, and maybe historians also. Also the ability to flag errors or suggests edits would be nice. It might be interesting to set up a system to notify any of the data sources of the suggested edit.

## Conclusion
This has been a fun project so far, it has already helped me to check out areas where we might be able to find some interesting sonar data. Hopefully it's useful to someone else out there as well!

Please reach out with any suggestions or idea for improvements on [Github](https://github.com/heathhenley/Shipwrecks).

If you find yourself in beautiful Beavertail in the summer, stop by the [Beavertail Lighthouse Museum](https://beavertaillight.org/); you might just see the [New England Shipwreck Map](https://www.neshipwrecksmap.com) in action on their 55" display!