---
title: "How web apps can watch your every move"
date: 2016-10-03
draft: true
---

To serve your customers, you should understand them. And to understand them, you need data. To that end, developers strive to collect a mountain of data and then mine it to extract meaning and derive hypotheses about their customers.

While evaluating analytics tools for [Bold](https://bold.io), a communication platform for companies, I tested some of the latest and greatest analytics services available. A few of them stuck out to me as incredibly powerful, perhaps a little too much so. The three services I tested were [Fullstory](http://fullstory.com), [inspeclet](http://inspectlet.com), and [hotjar](http://www.hotjar.com); all with the goal of providing real-time recordings of user activity with DVR-like capabilities.

For example, Fullstory’s home page says:
> **See what your users see. **Fullstory lets your company easily record, replay, search, and analyze each user’s actual experience with your website. Think of it as your team’s super-searchable DVR for all customer interactions.

And here’s a screenshot from inspectlet:

![Hmm…](https://cdn-images-1.medium.com/max/2000/1*3-UVazES-AMMsJtbOmC4lw.png)*Hmm…*

In other words, developers can watch your every mouse twitch and keypress right on their screen; without you ever knowing. From the developer’s perspective (myself included), this is the holy grail of raw user behavior analytics. But from the customer’s vantage point, this can feel a little creepy. Both sides are important to consider.

(Feel free to jump to the bottom where I offer a small form of respite in the form of a [Chrome extension named Snoopie](https://chrome.google.com/webstore/detail/snoopie/ickbbjgidjpmiggaclheacnffnpghpbn)).

### How easy is it for a developer to setup one of these?

Very. For example, with only a few lines of copy/paste Javascript, Fullstory and inspectlet will record every activity your user takes and allow you to play it back. The technology, albeit straightforward, is quite robust. It works by capturing the what the browser is rendering (DOM) and mouse events over time and uploads snapshots up to their servers. The web view is then reconstructed and made available for playback (just like a DVR). *Note: this clever trick is not possible on native Android and iOS apps.*

To be crystal clear, I’m not calling out Fullstory, inspectlet or any others as evil by any means, they all provide great technology and are incredibly useful. But, I’m using these as a specific example as they were the ones we took for a test drive. Upon using the software, all of our engineers had the same sentiment, “this is kinda creepy” or “I don’t feel right watching this.” That was their gut instinct (which is what ultimately lead me to writing this).

You might say it’s analogous to a security camera in a retail store or airport, so what’s the big deal? Perhaps, but those are for security purposes and we can usually see the cameras or the signs (that we all tend to ignore).

![](https://cdn-images-1.medium.com/max/2000/1*HjKQqgsGm5YiidlsE4aVHw.png)

On the web, however, that just doesn’t exist. And there are a couple more more problems to point out:

1. The developer might also know a lot more about you (all of the other information you’ve given them) if you’re logged in or providing any private user data.

1. Virtual everything is recorded by default.

In general this sort of tracking isn’t rooted in malicious intent on the developer’s part, but the opportunity remains to do real damage (purposefully or not). For example, imagine typing your password into the wrong field. Fullstory sees this and plays it back for developers, password in plain sight.

Here’s a a screenshot of onboarding captured with Fullstory (Sadly, animated GIFs aren’t uploading properly on Medium right now)

![Mouse movements, what I clicked, what I typed, and other data automatically scooped up, such as my (rough) location.](https://cdn-images-1.medium.com/max/2000/1*oxeb9YnFFhybSOWDmcqbmQ.png)*Mouse movements, what I clicked, what I typed, and other data automatically scooped up, such as my (rough) location.*

To be fair, Fullstory and others offer some ability to “exclude” elements from being recorded, but this takes work (typically by specifying [CSS class selectors](https://developer.mozilla.org/en-US/docs/Web/CSS/Class_selectors)) and is easily overlooked until someone realizes they shouldn’t be collecting it. In Fullstory’s case, the only default exclusion is for password input fields (which are censored in browsers by default, but can still reveal the length of a password).

### What should developers do?

To reiterate, I believe these types of tools are extremely useful in the right hands. As developers, I believe that customers should be made aware that they are being recorded if they’re logged in or providing private data. It’s one thing to record anonymous user sessions on your marketing pages to detect inefficiencies and optimize click-through rates, but an entirely other to track everything going on in your service.

For example, a service might provide a disclaimer upfront (we all know that nobody reads the ToS) or anonymize user data whenever possible.

Most importantly though, you must consider what’s worth tracking and what’s not, based on the type of software you’re providing. For example, go ahead and record how anonymous users respond to the redesign of your new home page, but not how easy it is to fill out your shiny new credit card form.

**Still not convinced because you want this treasure trove of data?
**Consider the fact that these services often use their own service. To put it another way: Fullstory uses Fullstory itself to record you on Fullstory. So they could watch you watch your users on your service — mind blown.

### What should the rest of us do?

As a customer, unfortunately, there’s not much we can do short of inspecting the loaded Javascript (too hard on desktop and forget about doing it on your mobile browser).

So, as a Sunday afternoon project, I built a [simple Chrome extension named Snoopie](https://chrome.google.com/webstore/detail/snoopie/ickbbjgidjpmiggaclheacnffnpghpbn), that will monitor the sites you visit and let you know when it detects one of the popular tracking services. It’s free to use, open-sourced and currently looks for [Fullstory](http://fullstory.com), [inspectlet](http://inspectlet.com), [hotjar](http://hotjar.com), [mouseflow](http://mouseflow.com), [hoverowl](http://hoverowl.com), [Lucky Orange](http://luckyorange.com), [Ptengine](http://ptengine.com), and [SessionCam](http://sessioncam.com).

![Snoopie smells something.](https://cdn-images-1.medium.com/max/2000/1*bVR2igjczZXUs5BF1vLGZw.png)*Snoopie smells something.*

Good luck out there!

Feel free to follow me on Twitter [@davidbyttow](http://twitter.com/davidbyttow)
