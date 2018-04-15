---
layout: post
title: "How updating to Node 8 led to timestamps 7 hours into the future"
description: "A change in timezone semantics in ES2016 resulted in an obscure bug with huge consequences."
tags: [work]
---

# The Bug

We recently updated one of our APIs from Node 6 to Node 8. In hindsight, I think we did just about everything feasible to make sure we didn't ship bugs, but still ended up with a pretty obscure one. We deployed on a Friday (I'm not afraid of Friday deploys), and early Monday morning started receiving customer complaints about certain timestamps being 7 hours off. Oops.

Specifically, this issue occurs when sending a date string with no specified timezone. For example, the string `"2018-04-15T15:00:00"` is 3pm today but at no particular timezone. The lack of timezone, in our case, is intentional. We can't determine the timezone because it's coming from another source that doesn't give us this info. The behavior we want (and what was working before) is for the timezone to basically be ignored; don't mess with my timestamp, please! Just let me do basic logic on it (like see if it is today, again ignoring timezone) and then display it on a web page. This issue was further confused because it's coming from a .NET process. .NET has a `DateTime` class where you can specify an `Unspecified` timezone - this means don't translate the timezone! This value is sent to a NodeJS API without the luxury of those same semantics. I didn't need another reason to dislike JavaScript...

# Troubleshooting

Naturally, when the bug was first reported, I assumed it was something weird in our code changes that caused it, or maybe one of our servers' timezones changed. I spent the next few hours bugging other team members for anything that could possibly explain this. I also went over the list of changes between Node 6 and Node 8 (I did this before deploying, too) and didn't find anything relevant. At this point, I knew it wasn't going to be easy to figure out what happened.

I started looking at our data and confirmed that it hadn't always been happening. In fact I was able to confirm that the change started happening between roughly noon and 6pm the day of the deploy. I confirmed that we deployed in that time range. That made me reasonably certain the deploy was responsible.

My manager then led the effort to figure out what within the deployment caused the issue. He started with the most likely things: other libraries being upgraded in the process, other code changes, etc. After a few hours he narrowed it down strictly to the Node upgrade. Even keeping the packages at the same version as before...the bug still showed up.

So, we knew it was Node, but didn't know what specifically in Node caused the issue. Like I say, I looked at all of the changes and didn't find anything relevant. It dawned on me that maybe V8 updated too and this included a relevant change. We decided at this point that we weren't going to roll back our change; we'd just work within the changes now that we know it's probably not a "bug" but just a change in behavior. While I worked on a fix, another coworker found the [V8 commit responsible](https://github.com/v8/v8/commit/d31c5410c4fdfc5eb66582892d5e3ecd3706bd58). Turns out, ES2016 changed how default timezones are handled.

# Workaround

Thankfully the workaround was straightforward. In our .NET process, all I had to do was tell the JSON.NET library to default timezones to UTC if they're not specified otherwise. This adds a "Z" to the end of the string. Worth pointing out, this is obviously incorrect because we're saying this is in a certain timezone and it almost certainly isn't in that timezone. But, this is how we make JavaScript not mess with our timestamps.

It may make more sense in new code to not use a date field for this, but rather split it into two fields: a date, and a separate "hours" field, or something similar. This would allow similar logic to work without having to deal with odd parsing behavior.

# Reflections

It was validating to discover how far removed this change was from our deployment, but also concerning. How could we have known this? I looked at [breaking change lists](https://docs.google.com/document/d/1g8JFi8T_oAE_7uAri7Njtig7fKaPDfotU6huOa1alds/edit) for V8 and don't see this change mentioned anywhere. What else could we be missing that may lead to similar issues in the future?

Have you dealt with anything similar? If so, please leave a comment below. What did you do to prevent this from happening in the future?
