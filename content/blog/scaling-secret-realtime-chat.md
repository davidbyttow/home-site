---
title: "Scaling Secret: Real-time Chat"
date: 2015-05-12
draft: true
---

*(This is the first of a series of technical posts I intend to write, and my goal is to help other engineers learn from the experience of building and scaling Secret.)*

I try a lot of iOS apps and many offer chat features (especially social networking and dating apps). Almost all are unique in their own ways and nearly all of them are broken in one way or another, usually they’re either too slow (not real-time) or too unreliable (messages fail to send or are duped), which is surprising given that chat has been around since the early days of the Internet.

Chat should be one of the *best* experiences in an app because it attempts to mimic a real-time conversation. That sets some basic requirements:

* The experience should be fast, reliable and simple

* The user shouldn’t need to “pull-to-refresh” to get new messages

Then, there are application-specific questions to answer:

* How many users can chat together?

* Can the user send photos or videos?

* Are the chats permanent or ephemeral?

* Should the user know when the recipient has read the message (read receipts)?

* Should the user know when the other person is typing a message?

* Should the user know when the other person is currently viewing the chat (presence)?

### Requirements

In Secret, real-time chat was a long-time discussion and something we had talked about since day one. We refrained from implementing it for various reasons:

1. We felt the core experience needed to be strong and we didn’t want to add too much too fast

1. We didn’t want private conversations to cannibalize public discussions

1. We didn’t have a strong case for how it should work or why it was important to the overall product

When our redesign came around late December 2014, and we added the ability for communities and strangers to connect, it made a lot more sense.

The requirements for Secret’s chat were:

* Both users should be completely anonymous

* Any user can chat with either the author of a post, or any other commenter

* The chat should be fast, reliable and clean (much to our designers’ chagrin, I was against ‘chat bubbles’ and wanted a much more linear, simple chat similar to Snapchat, IRC, Slack)

* Chat should be ephemeral. We didn’t want people feeling like their private discussions lasted forever (they lasted 24 hours since the last exchange)

* Users should be able to send (ephemeral) photos

* It should feel like a conversation and mimic the real-world as much as possible. Indicate the person is present (someone looking at you when you talk), indicate the person has read your message (acknowledgment or nodding along), indicate the person is typing (someone is talking and you should pay attention)

![](https://cdn-images-1.medium.com/max/2000/1*rUX4PYmgb5pyZxHX9yNnIw.png)

We knew chat would be a hot feature in the product. And quick back-of-the-envelope math concluded, assuming 5% of our simultaneous actives were chatting, that we needed to be ready to support up to 50,000 users chatting at once (50,000 persistent connections) from day one.

Note: We also held a global per-user private connection per session for in-app notifications (push notifications aren’t reliable or fast). We also required over 1,000,000 connections for that.

All of this was a tall order for not only a complete product redesign, but an entirely new chat feature in the product.

I challenged the team with building and shipping it in less than three weeks on both iOS and Android. Special thanks to our client team (Amol Jain, Jason Byttow, Safeer Jiwan, and Sara Haider) and our design team (Ben Lee, Cecile Parker, Chrys Bader) we pulled it off. Here’s how we built it.

### Architecture

When we talked about implementing chat, we usually talked about three options:

1. Use a 3rd-party chat service (e.g., [Layer](https://layer.com/))

1. Use a 3rd-party websocket implementation (e.g., [Pusher](https://pusher.com/))

1. Roll our own websocket-based protocol backed by our servers running on AWS or GCE (we were almost entirely hosted in App Engine at the time)

Chat needs to be fast and it’s not good enough to do polling (kill your servers with requests and your users with slowness), so we needed persistent connections (e.g., [websocket](http://en.wikipedia.org/wiki/WebSocket)).

We quickly discarded #1 as an option (Layer), because we wanted to control our own user data and we wanted full control over the stack and the experience (admittedly, we didn’t dive much into Layer’s entire offering, but we felt that rolling our own would the be the fastest path).

While some felt #3 was the best option, I believed #2 might actually work out best. I threw together a web-based, internal-only prototype over a weekend based on Pusher.

The architecture was simple. Here’s a (crude) image of a simple request flow.

![Wow, I have bad handwriting.](https://cdn-images-1.medium.com/max/2632/1*LVlDbUvLeVyf75fqqe8gjg.png)*Wow, I have bad handwriting.*

When a user entered a chat room a [private, presence channel](https://pusher.com/docs/client_api_guide/client_presence_channels) was created and connection established with pusher. This let the user receive notifications from Pusher for the duration of being in the chat room. It was destroyed when the user left or backgrounded the app. Luckily, there were 3rd party Pusher protocol libraries (albeit we had to modify them) available for iOS and Android, which sped up development.

In this image, there are two users (A and B) present in the same chat room.

1. User A presses send on a message and a POST request is made to the Secret frontend with the chat id

1. Server retrieves the chat session data, adds the message, and writes it back

1. Server makes a POST request to Pusher with a payload intended for the client

1. Pusher routes the message to User B via the websocket connection

Typing notifications and delivery receipts were very similar.

In this model, although not perfect from a latency perspective, it was good enough and simple because requests only flowed in one direction: Client -> Server -> Pusher -> Client

It’s also important to note that Pusher was only there for real-time notifications, not for canonical data. If at any time the user came back to the chat room, it would refresh the most recent state from the server and accept any new notifications from there. Analogous to rendering a single-page application and delivering changes to the client via AJAX or websocket.

### Code and Model

The stored data model in the backend is simple. Secret’s canonical datastore was Google App Engine’s High-Replication Datastore (more on that in other blog posts). Essentially, it’s a schema-less, NoSQL datastore built on top of BigTable and Megastore. Entities are document structure and stored in rows by a given key. There are no JOINs and query semantics are very limited, but it allows for very high read-throughput and wide scaling you’d expect from a NoSQL offering.

![](https://cdn-images-1.medium.com/max/2068/1*BTzkP0kzQYJzTXylZl6CCw.png)

When the user enters a chat room, an idempotent ID is created on the server that is effectively “<user1_id>:<user2_id>:<secret_id>”, also known as the chat session id. Important note: user ids in this key were always sorted (partial-ordering), guaranteeing idempotency. That way, given a pair of users and a secret, we can always generate the single ID for that secret.

Chat sessions are keyed by simple ids and contain the data above in a single row. Each time a chat is mutated, the server performs a transactional read-modify-write on the row. The transaction is fine so long as write throughput is kept to <= 1 write/sec per entity.

For example, when a user left a chat, we wanted to alert the recipient that they had done so. The server-side code looked like this:

![](https://cdn-images-1.medium.com/max/2164/1*48ZaRW6LFADOftTt2m2zIQ.png)

Pretty simple.

For fast queries for things like showing the user all of their ongoing or previous chats, we created indexes on the participant and created time properties, allowing fast answers for things like “Fetch chats user X is a participant in sorted time in descending order” and locally sorted by last update time. Because chats only lasted 24 hours since the last message exchange, we knew the number of chats would be a reasonably small number to fetch and sort locally on the server (e.g., < 100 in almost every case). If they exceeded that, chances are the user was a bad actor and we could drop some on the floor. That code was simple:

![](https://cdn-images-1.medium.com/max/2404/1*RTObyxGygknNNI-h-991jA.png)

Here’s a small library I just open-sourced for talking directly to Pusher via Go (in App Engine environment). You can easily tweak this, the only tricky part of the entire process was authenticating the requests for private and presence channels. [https://github.com/guitardave24/pusher-go-appengine/blob/master/pusher.go](https://github.com/guitardave24/pusher-go-appengine/blob/master/pusher.go)

### Launch

When we launched the redesign, chat was an instant hit. It grew week-over-week to well over 1,000,000 concurrent connections (chat + notifications). Luckily, Pusher worked well and was fairly inexpensive and our scaled linearly to a point where, assuming Pusher was able to meet our demand, we would have no issues for the foreseeable future.

The key takeaways and reinforcements for me in this experience were:

* Start small and be ready to build a throw-away prototype to help force a decision

* It’s often unwise to roll your own implementation, no matter how fun it might be (obviously, but we all keep doing it!)

If I were to implement real-time chat in an app again, I’d strongly consider using a simple model like this again. The downside of the above architecture is that it’s not as fast as it could be (notably, because messages are routed through the Secret frontend to Pusher), but the end result was just fast, reliable and simple enough.

Hopefully you found this interesting and/or useful, if you have any questions, please don’t hesitate to email me at [d@secret.ly](mailto:d@secret.ly) or follow me on Twitter [@davidbyttow](http://twitter.com/davidbyttow)

David Byttow
