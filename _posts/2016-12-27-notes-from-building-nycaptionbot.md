---
layout: post
title: Notes from building @NewYorkerCaptionBot
archive: true
---

<img class="framed" src="/img/2016/newyorker.jpg" alt="A New Yorker cartoon with the caption 'How soon until the trolls arrive?'">
_<a href="https://twitter.com/NYCaptionBot/status/813678246873202688">Caption by @jtag</a>_

Over the holiday weekend I put some free time towards building my first Twitter bot: [@NYCaptionBot](https://twitter.com/NYCaptionBot/): a bot that takes an @-message and turns it into a caption for a random New Yorker cartoon.

All-in-all, it took maybe 12 hours scattered over a few days. This was my first project using the Twitter API and using Node, so there was a bit of a learning curve, but development was ultimately pretty straightforward. Now that it’s unleashed into the world I wanted to jot down some thoughts and things I learned in the hope that they might be useful to someone else down the road.

<!--more-->

## How I got started
I started with the [Twitter bot starter project on Gomix](https://gomix.com/#!/project/twitterbot). If you’re not familiar, Gomix is a new site that makes it super easy to tinker with web apps without having to worry about setting up an environment, dealing with deployment, etc. I had been wanting to take it for a spin, so this seemed like a good opportunity.

This starter project had everything I needed to get started, but first I had to create a new Twitter account for the bot, register a new app, and add the credentials to the .env file.

**A quick note on setting up a new Twitter account:** Twitter only lets you have one account per email address. I didn’t want to make a whole new email account for this, so instead I used a little known Gmail trick and added a `+` modifier to my email address like `noah.manger+somethingelse@gmail.com` The emails still go to my main account, but it registers as a new email address to Twitter. Just wanted to note that because if you’re setting up a bot this could save you a step.

Anyway, once I added the credentials things basically worked as expected and I could start tinkering. I could have kept going, but I thought I would need some things that would require developing locally, so I just downloaded my Gomix project to my computer and continued there.

## How it works
Here’s the basic way that the bot works (for more details, you can check out [the actual code](https://github.com/noahmanger/newyorkercaptionbot/) ).:

### 1. Get the tweet
The tiny expressjs app uses [Twit](https://www.npmjs.com/package/twit) , a Node wrapper for Twitter’s API to set up a new [https://dev.twitter.com/streaming/userstreams](https://dev.twitter.com/streaming/userstreams) to Twitter. This shows you all Tweets that this user would normally see. I set `replies: ‘all’` to show all @-replies, regardless of whether the bot is following an account and then set it to filter for messages that included the bot’s handle.

When a tweet pops into the stream, the bot first makes sure that it didn’t come from itself and then makes sure that the @nycaptionbot is the first word in the tweet (so that if people are just talking about it, those tweets don’t trigger a post).  It then parses everything after the @nycaptionbot handle and saves it as the caption to add to the image.

### 2. Get the image
Then things get interesting. First it needs to get a random New Yorker cartoon. I initially saw a couple ways to do this:
- I could go and download a bunch of cartoons and store them in a folder on the server
- Or I could hopefully find a site to scrape a random cartoon

After going with the first method for prototyping purposes, I decided to just google “random new yorker cartoon” which amazingly turned up [newyorker.com/cartoons/random/](newyorker.com/cartoons/random/): a site that gives you a random cartoon on each visit.

"This was perfect!" I thought. I would just have to set up a simple scraper to go to the site and get the `src` of whichever image was showing. Well, because the site relies on AJAX, scraping would be a little more difficult because it would need to wait for the image to be loaded before it could get the data. I had never done anything like that, so after some googling I landed on a technique to use [Nightmare](http://www.nightmarejs.org/).  I actually got this to work exactly like I needed with minimal effort, but when it came time to deploy to Heroku I learned that getting Nightmare to run on Heroku is actually quite a pain. So I banged my head against the wall for a couple hours before looking for alternatives.

Fortunately for me, the alternative was 100x easier. On a whim, I opened up the network tab in Chrome on the random cartoon page and lo and behold, a call to an open, undocumented API that returns a new URL to a cartoon every time:

![Chrome network tab](/img/2016/network.png)

So I ripped out the scraper code and replaced it with a call to newyorker.com/cartoons/random/randomAPI and it worked flawlessly.

### 3.Add the caption
Next up, it adds the caption. After poking around a little I settled on [gm](https://www.npmjs.com/package/gm), a GraphicsMagick library for Node. With this I was able to:

- Resize the cartoon to a uniform width
- Append a blank white rectangle below jpeg, which would hold the caption
- Draw the text on the image (using the same font that they use at the magazine, of course)

Because gm doesn’t support line breaks the bot splits up text after about 70 characters and turns it into two separate lines.

### Post the image
Once the image is made all that’s left is to post it. Tweeting an image is a three step process (after first encoding the image as a base64 string):

1. First, upload the image using `Twit.post(‘media/upload’)` which then gives you a `media_id`
2. Add metadata using `Twit.post(‘media/metadata/create’)` which lets you add alt text
3. Post the tweet using `Twit.post(’statuses/update)`, passing along the `media_id` of the image and the tweet text.

And that’s it! The bot posts a new tweet, @-ing the original user by giving them credit for their caption.

### Debugging
The only really annoying part about this whole thing was debugging. There might be a better solution, but I ultimately came up with a pretty cumbersome rig.

I made another twitter account specifically for testing (so that my personal timeline wouldn’t be full of gibberish).  I then kept this testing account open in a different browser and used it to send test messages. And I added lots of `console.log()` messages so I could tell where things were breaking, which is how I was able to figure out that Nightmare was just silently failing.

Of course, it wouldn’t be software if it didn’t break as soon as you showed it to people, so after sharing it late last night and the first users started to interact, I quickly realized there was a problem with it sending out multiple tweets with the same caption, and sometimes attributed to the wrong users. I won’t get into the details here; just sharing because hey, things always go wrong.

---

Again, you can check out the full repo and fork it / make a pull request / do whatever over [here](https://github.com/noahmanger/newyorkercaptionbot/). Go follow and interact with the bot at [@NYCaptionBot](https://twitter.com/NYCaptionBot) and me at [@noahmanger](https://twitter.com/noahmanger).
