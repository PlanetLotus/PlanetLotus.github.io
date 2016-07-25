---
layout: post
title: "Why I decided to ditch AutoMapper"
description: "Mapping objects is painful. AutoMapper makes many aspects of mapping less painful, but is the cost worth it?"
---

### Background

First off, I have no desire to bash a free tool that does its job very well.
Some people love AutoMapper. I really think some devs will love what it has to
offer and others won't. For what it sets out to do, it does a great job, and
since I haven't filed any bugs or feature requests with it, it wouldn't be fair
for me to publicly bash it for what it's supposed to do. Keep that in mind for
the remainder of the post. I appreciate all the work that's gone into it.

That said, I started using AutoMapper because a coworker wanted to. After
investing several hours into using it over several months, I'm ready to report
why I choose to write new code without AutoMapper, and why I wouldn't want to
work on a team that insisted on using AutoMapper.

It's worth noting that I use AutoMapper in an MVC project, so all of my
examples will be based in an MVC project.

### What AutoMapper Does

AutoMapper is a .NET tool that seeks to make mapping objects between formats
less tedious and less time consuming without sacrificing performance. As any
MVC developer will tell you, mapping objects from database, to business layer,
to view is really tedious and seems like a huge waste of time.

What AutoMapper is very good at is mapping two objects without any
customization. If `DbFoo` (database object) and `Foo` (business object) have
exactly the same properties on them but are "duplicated" (not really duplicate,
but seems that way) to maintain separation of layers, then using
AutoMapper is a huge win.

Let's say the classes look like this:

{% highlight csharp %}
// The class representing the db table
public class DbUser
{
    public int UserId { get; set; }
    public string UserName { get; set; }
    public string Email { get; set; }
}

// The "business logic" class
public class User
{
    public int UserId { get; set; }
    public string UserName { get; set; }
    public string Email { get; set; }
}
{% endhighlight %}

In this case, mapping with AutoMapper just looks like this:

{% highlight csharp %}
Mapper.CreateMap<DbUser, User>();
{% endhighlight %}

Whereas without AutoMapper you'd have something like this:

{% highlight csharp %}
public static User MapUser(DbUser user)
{
    return new User
    {
        UserId = user.UserId,
        UserName = user.UserName,
        Email = user.Email
    };
}
{% endhighlight %}

If all of my models matched the other layers this directly, I'd probably use
AutoMapper.  Unfortunately a non-trivial app is unlikely to be this
straightforward, which in my opinion, is where AutoMapper *can* work, but
starts losing its value.

### Problem 1: Losing Compile-time Safety

One of my biggest gripes with AutoMapper is that it takes a strongly-typed
language (C#) and makes all mappings computed at runtime. In other words, you
don't know if your mappings work until you run them. This is a problem even in
the simple case mentioned above. Let's say a particularly picky developer
decided to rename the `Email` column on the `DbUser` class to `EmailAddress`,
but either forgets or can't update the `User` object due to its
numerous uses. Now your AutoMapper mapping just broke but you have no
idea. Not only will it not tell you at compile time, but this sort of
change will silently fail. The property that doesn't map will simply be
ignored. As far as I'm aware, this is by design.

Compare to the manual mapper and simply building will reveal this bug.

### Problem 2: No Static Analysis

I use Visual Studio's and ReSharper's static analysis tools frequently, such as
Find Usages. While refactoring, I need to make sure all uses of something were
updated correctly. If I make a change like in the last example but I'm using
AutoMapper, it's now a lot harder to exhaustively find all uses of a class or
one of its properties. I've found this not only leads to bugs, but also extra
mappers that should be deleted but no one is confident enough that they can be
safely removed. The problem is, the only way you can test an AutoMapper mapping
is by hitting each use of it at runtime which usually isn't practical in a
large codebase.

### Problem 3: Debugging

There's probably better debugging methods than what I've done in the past, but
after using AutoMapper for several months I have spent cumulative *hours*
debugging mappings that don't work. These were usually very simple bugs that
were rather embarrassing, but the bottom line is that it would've been caught
at compile time if it were a manual mapper. This of course assumes that I
actually know there's a bug and the code isn't suffering from Problem 1 above.

### Problem 4: Customization = Lots of Syntax

AutoMapper can handle more complex mappings, but this comes at a cost of either
writing these mappings in lambda expressions, or factoring them out into custom
mappers. The former works for awhile, but if your logic isn't incredibly
straightforward then the expression will become very difficult to read. The
latter I haven't personally explored, because I feel that if I have to add that
much code for an AutoMapper mapping, then I've completely defeated the point
and should just write a regular mapper method.

### Problem 5: Only 1:1 Mappings Supported

As far as I know, a mapping can only take one object and spit out a result. It
can't map two objects into a third object. I run into this case pretty
frequently. Here's an example when querying two different data sources:

{% highlight csharp %}
FooApiDto fooApiDto = await fooApiService.GetFooApiDto();
FooApiDto2 fooApiDto2 = await fooApiService2.GetFooApiDto2();

// Can't do this; wish I could
Foo foo = Mapper.Map<Foo>(fooApiDto, fooApiDto2);
{% endhighlight %}

Another common example is while taking in form input:

{% highlight csharp %}
public async Task<ActionResult> Index(FooProfile fooProfile)
{
    string userId = this.User.Identity.GetUserId();

    // Can't do this :(
    FooUserDto = Mapper.Map<FooUserDto>(userId, fooProfile);
}
{% endhighlight %}

### Alternative?

I think mapping between object formats is the most tedious part of
programming (in MVC anyway) when it comes to adding features or refactoring and
it seems like such a waste of time. I've yet to find a viable alternative to
writing manual mappers as shown above, but this seems like the simplest and
least error prone method, even if it does add a ton of code and possibly
require more time up front. It really doesn't take long to write mappers and I
like not running into mapper bugs. That said, it does feel pretty lame to write
`UserId = user.UserId`...it just seems so obvious that a computer should be
able to do it for me. But that's AutoMapper, and I don't think the advantages
are worth the disadvantages it brings.

If you've solved this problem a different way, please let me know in the
comments below. There's got to be a million dollar bounty on that one.
