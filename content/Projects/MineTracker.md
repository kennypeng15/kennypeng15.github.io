---
title: MineTracker
---
Check out MineTracker [here!](https://kennypeng15.github.io/minetracker/)

{{< toc >}}

## Overview and Motivation
MineTracker is an application for collecting and displaying information about my _minesweeper_ games (played on
https://minesweeper.online), with the ultimate goal of _tracking_ performance over time.

I first started playing minesweeper as an undergraduate in college - the COVID pandemic hit, and I needed
something to keep me occupied! As I played more, I thought it'd be interesting to design a way to gauge 
if and how I was improving. 

The minesweeper.online platform offers some information towards this regard: for each user, it stores a table
of successful games (i.e., games where the user won), along with some statistics about those games 
(e.g., the time it took, how efficient the user was with clicks, a measure of how difficult
the game board was, etc.). However, I found this information to be lacking in two ways:
1. There's no option to display information graphically; information is only displayed as a table, and,
2. There's no information displayed about games that were not ultimately successful (i.e., games where the user lost).

The second point was particularly important. I've found that the majority of minesweeper games
I play are not successful, but they still may contain useful information (notably, minesweeper.online
has an additional "estimated time" for unsuccessful games), and thus tracking them could lead to insights.

If minesweeper.online had a table of a user's unsuccessful games 
in addition to successful ones, it would be fairly straightforward to design a web scraper to obtain that
information, which could then be displayed graphically. 

Since it doesn't, however, I designed MineTracker.

MineTracker relies on the fact that minesweeper.online is an in-web browser minesweeper platform,
so each game played leaves an entry in the history of whatever web browser is used (in my case, Chrome). 
Since web browser history is stored locally, it can be programatically accessed; MineTracker thus
scans through browser history, looking for any entries (i.e., URL + access time combinations)
corresponding to minesweeper.online games. MineTracker then scrapes the URL associated with the entries, 
obtaining information about the game regardless of if it was successful or not. This information is 
persisted to a database, which powers an API, which in turn powers a front-end to display the information
in a palatable manner.

(as an aside - check out https://minesweeper.online/help/guides for more information on minesweeper and game statistics)

---

## Architecture and Design Considerations
I originally designed MineTracker to work completely locally (i.e., entirely on my own laptop): the
browser history and web-scraping application ran locally and persisted data to a CSV file; another application
read that CSV file and stood up an API, exposing the data; and a frontend ran locally was used to 
look at the data.

This was (barely) functional, but had obvious downsides:
1. CSV storage, i.e., having to read from file every API call, was grossly inefficient
2. It was really annoying having to start the API server, then have to start the frontend
3. I frequently ran into problems (i.e., temporary IP bans, completely justified) from minesweeper.online when running the scraping job, since I was regularly making high volumes of requests from the same IP address, meaning grabbing any new data was a pain.

For that reason, I decided to migrate MineTracker over to the cloud, where possible.
I opted to use AWS technologies, due to my having some familiarity with them (through certifications)
and due to their wide offering of free tier services. In my usage of AWS and throughout the MineTracker
project in general, I made a point of using as much always-free stuff as possible (even if 
not necessarily ideal from a design perspective, as will be discussed later).

MineTracker now resembles a typical 3-tier web application, having a data layer, an API exposing that data layer,
and a presentation layer.

![MineTracker's architecture](/images/minetracker-arch.png)

### Data Layer
The data layer of MineTracker consists of two projects: [minetracker-lambda](https://github.com/kennypeng15/minetracker-lambda) 
and [minetracker-publisher](https://github.com/kennypeng15/minetracker-publisher).

[minetracker-publisher](https://github.com/kennypeng15/minetracker-publisher) is designed to be run locally; 
at its core, it is a python application that scans through a user's browser (Chrome) history, parses out minesweeper.online games 
(by looking for entries in history that have URLs beginning with minesweeper.online/game/), and sends those games and the times
they were played at to an SNS endpoint in AWS.

[minetracker-lambda](https://github.com/kennypeng15/minetracker-lambda) is intended to be used as an AWS Lambda function. 
It is invoked whenever a (game URL + timestamp) combination is published to the SNS endpoint that minetracker-publisher writes to.
minetracker-lambda uses selenium to scrape the game URL provided via SNS, parses out key information about
the minesweeper game (e.g., if it was a successful game or not, the game time, etc.), and 
writes that information to a DynamoDB database. 
Unsolved games without sufficient progress are not written to the database to avoid noise.
An SQS deadletter queue is configured for any failures.

[minetracker-lambda](https://github.com/kennypeng15/minetracker-lambda) circumvents the issue of IP bans since Lambda environments are ephemeral.
However, since a custom Lambda image must be used to have selenium, other necessary python libraries,
as well as a usable Chrome binary, Amazon ECR-Private must be used;
this, unfortunately, is the only AWS non-always-free service used.

The choice of DynamoDB for the database was one made for financial reasons: DynamoDB qualifies
for AWS always-free, while something like RDS is only free for 12 months.
In our case, the data we store is definitely more relational in nature, and RDS would be a better option
from a pure design standpoint.
This is made apparent when working with DynamoDB scans and queries:
- To get data from a DynamoDB database, you can either scan (search through the entire database), or query (look for items with a certain partition key).
- Partition keys must uniquely identify data; alternately, items can have the same partition key, but they must have a (partition key, sort key) combination that is unique.
- With DynamoDB, you cannot query for attributes that are not either the partition key or the sort key.
- The types of queries I want to run on my data, regardless of how I choose (partition key, sort key), would ultimately require querying on non-key attributes.
- It is thus impossible for me to utilize the more efficient query operation for my use case, and I thus have to use the less efficient and more expensive scan option.
- These queries would be very simple in SQL. (`TODO`: example?)

### API Layer
The API layer of MineTracker consists simply of [minetracker-api](https://github.com/kennypeng15/minetracker-api), 
a Flask API hosted on [pythonanywhere](https://www.pythonanywhere.com/) using their free tier.

The API exposes only basic GET endpoints, which return data (which can be filtered with optional query parameters)
and basic diagnostic information about MineTracker.

A naive and basic cache is used to circumvent potentially expensive repeated full-DynamoDB scans.

### Presentation Layer
The presentation layer of MineTracker consists of a React application, [minetracker](https://github.com/kennypeng15/minetracker), 
hosted on [GitHub pages](https://kennypeng15.github.io/minetracker/).

GitHub actions are used to automatically update the application whenever changes to the source
code are made.

### Auxiliary Helpers
I also created a small auxiliary application to assist in migrating old CSV data to DynamoDB, 
[minetracker-migrator](https://github.com/kennypeng15/minetracker-migrator).

It simply reads in a CSV file, converts column names and filters out extraneous data as necessary, 
and then writes to DynamoDB.

---

## Future Work and Considerations
Stay tuned!