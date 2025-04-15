---
author: "pshegger"
title: "My First Gamejam"
date: "2023-05-02"
description: "My thought and experiences about my first gamejam (Ludum Dare 53)"
tags: ["gamedev", "gamejam", "ludum dare", "godot"]
categories: ["gamedev"]
---

> If you're only here to try the game, you can do it on [itch.io](https://harmlesshawk.itch.io/wheres-my-lunch)

I cannot even remember when was the first time I heard about [Ludum Dare](https://ldjam.com/), but it was at least 5 years ago. Even back then, I was fascinated by the idea of creating a game in such a short time, but even though I wanted to try it, I had no game development experience at that time, so it always remained just a dream.

Up until now. Ludum Dare 53 was organized last weekend, and I finally decided to try it myself by participating. The last big push I needed was the introduction of the extra format, where you had three weeks to submit your game, instead of the usual 2/3 days. As you'll see, this wasn't required in the end, but it gave me the confidence to start.

## So, what is Ludum Dare?

Ludum Dare is one of the biggest (and probably the most well-known) game jams in the world.

The concept of game jams is simple. You have some kind of restriction (usually a theme), and you have a limited time to create a game. Once you're done, you share your game with the public, and they can give feedback and rate it in different categories.

This time, Ludum Dare had 3 formats: Compo, Jam, and Extra.

Compo is the most restricted and the hardest. Only solo developers are allowed and they have 48 hours to write the game and create all required assets for it.

Jam is a bit more relaxed, both solo developers and teams can participate, the time limit is 72 hours, and you can use sounds/graphics that were created outside of the time frame. You're also not required to create these assets by yourself, you can use outside resources.

Extra is a fairly new format, mostly the same as jam, but you have 3 weeks. The biggest difference is, that in the end your work will not be scored.

## Preparations

Even though you have to create the game itself in the given timeframe, there are a few things you can prepare for. For me, it came down to 2 things: come up with ideas for the possible themes, and decide what game engine to use.

The first one is probably the harder task of the two. Luckily, the theme of the jam was decided by public voting, so a few days before the start we already had a list of 15 possible ones. I went through them one by one and tried to come up with some concepts the day before the event. Unfortunately, I couldn't figure out concepts for all the themes, but fortunately, the chosen theme was one that I had a concept for.

The second thing was to decide what game engine to use. Even though I don't have a huge game development experience, I played around with a few of them in the last year. The 2 main competitor was [Bevy](https://bevyengine.org/) and [Godot](https://godotengine.org/). One of my main requirements was to be able to create HTML5 builds with it, but since both engines have support, I couldn't decide based on that.

Bevy is an awesome open-source engine, written in Rust, and has a really interesting (and performant) [ECS](https://en.wikipedia.org/wiki/Entity_component_system)-based architecture. It's early in development but has great potential in my opinion.

Godot is a more well-known and also open-source engine, that has a great graphical editor interface but uses its own language (GDScript). It is possible to use it with C#, but the documentation is lacking, and WebGL support is also missing.

In the end, I decided to use Godot, purely because the editor makes development quicker in my opinion, but if I ever start working on a bigger project, I'll consider Bevy.

## The First Day

Since the start of the event was at 3 AM for me, I couldn't start right away when the theme was announced. I only learned what the theme was when I woke up on Saturday morning. I was a bit sad that it wasn't the one I'd preferred, but at the same time, I was happy that it was one that I had an idea for. It was *delivery* and I already decided on the previous day, that I wanted to create a top-down driving game, where your goal is to deliver lunch for hungry customers. I know it's not the most original or exciting idea, but hey, it's a start.

I started to work on it at around 7 AM. Since I'm not a graphics designer, my first task was to find some assets that could be used for this concept. Luckily, my go-to place for these kinds of things (thanks [Kenney](https://kenney.nl/), you're awesome) had an asset pack that was perfect for this game.

As the next step, I started to create the map itself. I had some previous experience with tile maps in Godot, but only small trials, so this was the first time I had to really use its capabilities. The process itself was fairly easy, but it took a lot of time. Looking back, I shouldn't have spent that much time on small details, the fast-paced nature of the game makes it almost impossible to notice them. All-in-all, it took me until early afternoon (~2-3 PM) to have a map that I liked enough to move on to the next task.

My next goal was to create the player car and add controls for its movement. Creating the car itself wasn't too big of a deal, I already had all the assets, I just had to assemble them. Then came the movement. My first idea was to use a rigid body, that way it will handle collisions with other objects (once I added the collision boxes to the tile set). Adding some simple controls was easy, just apply the correct force when the input is pressed. I quickly implemented it, and it was awesome to drive around the map. The only issue was the missing collision boxes for the tiles, so I also added those, and that's when the issues began. Driving was still fine, but as soon as the car collided with the tiles, it started to spin around uncontrollably. This of course was annoying, so I had to find a good solution. I tried a lot of things, but in the end, all I had to do was change the rigid body to a character controller (and reimplement the input handling, since character controllers have different physics).

At this point, I could drive around the city, but it was a bit empty, so I decided to add some AI cars that drive around too. I thought it would be easy, since all I had to do was to define a route for them and make them drive forward, with the occasional turning when it's required. Well, the turning part wasn't as easy as I expected it to be (quelle surprise), but I managed to make it work after some tinkering. It's still not the best, it can get stuck easily, but I wasn't expecting more in such a short time. (Also, the mandatory, it's not a bug, it's a feature since them getting stuck means the player has less space to navigate, making the game harder as time progresses)

I spent the remainder of the day (until around 10 PM) setting up a looped route for the cars, which took a bit of time since I had to manually find the correct target coordinates, which minimized the chance of them getting stuck.

## Day Two

I started this day where I finished the previous one. Added two new loops for two new groups of cars, to have more movement around the map. Even though I wanted to set up the loops without overlaps, I managed to somehow create one, and since I didn't want to spend more time on this part of the game, I decided to leave it. Luckily the timings turned out pretty well, the cars avoid each other most of the time, and it actually looks cool.

After finishing that part of the game, I realized that I should add some gameplay too. At this point, I had to make a decision. Which format do I want to participate in? My original intention was to have a game for the extra format, but the progress I made on the first day meant that I could finish the (somewhat limited version of the) game in time for the jam category. Since I wanted to get as much feedback as possible, I decided to participate in the jam category and finish the game as soon as possible.

My original idea was to have specific pickup locations (the restaurants) and deliver the food to random buildings starting at those locations. I also wanted to create a system to accept/decline orders, so the players have to make decisions about how to best organize their time. I was also thinking about splitting the game into days and having the player pay for the gas they used during the day. In the end, I had to abandon these ideas, because I thought it would take too much time to implement them. 

Instead, I chose a simplified version, where the player has to visit locations one after the other, each delivery resulting in a salary (based on distance) and a rating (based on speed). The game ends when the player's rating drops below a threshold. This was definitely easier (and with that quicker) to implement.

After I implemented these mechanics, I added some way to display the earned salary and the current rating of the player. I also added a simple animation that shows the delivery results once it's done. With these done, the game was in a playable (and somewhat enjoyable) state.

Of course, there were still a lot of things to do. The next task on my list was to create a game over screen and a simple menu screen. After that, I started to look into music and sound effects. I already had a few sound effects from Kenney, but I needed one for the delivery completed notification, so I went to itch.io and found a good one from [Shapeforms Audio](https://shapeforms.itch.io/). I also needed some good music, and after some search, I found an awesome song by [David KBD](https://davidkbd.itch.io/). After adding them to the game, only testing left.

This is the part, where I have to mention an issue I had during the two days. The game ran fine (500+ FPS), but occasionally it had huge FPS drops, which was not only annoying but also threw off the NPC car algorithm, resulting in more collisions than intended. Until this point, I dismissed this issue, since I thought it happens only because I'm using a debug build. As soon as I started testing the release build, I had to realize that this is not a debug issue, so I had to find a solution for it. I did a lot of research, but couldn't find the issue. I used the built-in profiler, but all I could find was that my code wasn't responsible for the longer frame times (the max time it required was less than 1 ms) and that the issue was with `Process Time`. This one took more than 500ms sometimes, and I had no idea why. After some research, I tried one of the suggested solutions of setting a max FPS. After that, I could still see some long frame times, but it happens rarely, so it seems to have solved the issue.

At this point of time, it was already after 10 PM on Sunday, so I decided the take a break and continue in the morning. Fortunately, Monday was a public holiday, so I had time to finish everything.

## Final Touches

Monday morning I felt completely drained, so it took some time to start doing the last remaining tasks. It also didn't help that these tasks were the ones that excited me the least.

I had to create a cover image for the game. As I already mentioned, I'm not at graphics design, so I decided to just put the name of the game on a semy transparent background, with a screenshot behind that. I also made a really basic icon, with just the name of the game on it. Not too exciting, but still better than using the default one.

The next step was to create the builds. In theory, this should be easy, just select one of the templates, and press export. Naturally, this was not the case.

MacOS apps need to be signed and since I don't have an Apple developer account, I had to use the default ad-hoc signing. This means, that when someone wants to start the game the OS will prevent it, and they have to make some adjustments to allow it anyway.

On Windows, I had issues with setting up the correct icon. It requires some tool that is Windows only, and since I'm working on Mac, it would require Wine. When I found this issue, I just decided not to do it, because it doesn't worth the hassle. When I tried testing the build on my Windows machine I also had to realize, that Windows proudly "defends" me from running the game. I didn't look into solving this issue, since I thought most people will not use the native builds anyway.

The thought behind that of course was that my main target was the HTML5 build. It is the easiest for testers, and it also poses the least risk. Of course, creating this build was harder than expected as well. Godot 4 has made changes to its WebGL exports and now requires some headers that you cannot control most of the time. I found a plugin in the asset store that solved one of the issues, but even when I was using that, the Ludum Dare side still couldn't properly embed the game. I managed to upload it to GitHub Pages, where it fortunately works. Later, I also uploaded it to itch.io, and that also works fine.

As for the last platform, I had no issues with the Linux build, although the truth is I didn't test that version.

With the cover image and the builds ready, all that was left was creating the submission page and writing a few words about the game. I quickly managed to do both, and with that, the hard part of the jam was over.

## What's Next?

Now, that the submission period is over, there are only two things to do: play and rate the games of other participants, and read the feedback on my game.

I already played some of the games, but I'm planning on trying as many of them as possible in the following days. Some of the ones I've already played were awesome, it's hard to believe that they were developed in only a few days (but of course, I know that that's the case).

As for the feedback, I already got a few comments, most people think it's a fun game with sometimes frustrating controls. I have to agree with both of these points. I hope I'll get some more comments and ratings, but since the rating period has just started, I'm sure more people will try it (especially when they had some rest after this long weekend). I really hope to get at least 20 ratings, since that's the minimum to see them.

The future of the game is also clear to me: even though it's a fun little game, I don't think it's unique enough to work on it any further. It is available on itch.io so anyone can try it, but no more development should be expected.

So, what about my future as a game developer? I'm still only at the beginning of this journey, but I'll continue creating games in the future, that's for sure. The next Ludum Dare will be at the end of September and I'd love to participate. I'd also like to take part in a GMTK jam since that was one of my inspirations to start this whole journey. Other than jams, it would be great to start working on some kind of longer game, but I first need a good idea for that.

## Thoughts About Godot

Even though I tinkered with Godot in the past, it was my first time using it for something more than a learning project, so I thought I'll share a few thoughts about it.

The editor is fairly straightforward, I could easily understand how it works and where to search for the things I'm looking for. The "everything is a scene" concept was a bit strange at first but quickly got used to it. The code editor integration is also nice, the suggestions were really useful and made development quicker. I also like that the documentation is quickly available right from the editor.

On the other hand, I also had a few issues. It stopped responding multiple times during the 2 days. I'm using an older MacBook Pro, so I expect slowdowns, but completely stopping is something that never happens with other applications. I also don't like the code editor itself. As I already mentioned, it has great integration, but I always had trouble navigating between open files. I got used to my shortcuts, which are set up by default in every other application (for example Ctrl+Tab to cycle between tabs), but here I couldn't find the settings to have something similar. It might be just me not paying enough attention, but it was annoying for sure. I also tried setting up VSCode as an external editor, but whatever I did, I couldn't make it connect to the language server, even though I was using the official plugin.

My other issue was GDScript itself. It's not a bad language at all, really easy to learn if you have some Python knowledge, but as an Android developer, I'm spoiled with Kotlin. I hope the C# documentation gets improved over time and it will get support for WebGL exports.

These are of course just minor annoyances, and I got used to most of them after a while, I just wanted to mention them. It's also worth mentioning, that even with these issues, the experience was great and far faster than working with only a code editor, and having no graphical editor interface.

## Where can I play the game?

It is available on [itch.io](https://harmlesshawk.itch.io/wheres-my-lunch). The HTML5 version is embedded into the page, but you can also download it for Mac, Windows, and Linux. Please keep in mind, that the Linux version is not tested.

You can also find the source code on [GitHub](https://github.com/HarmlessHawk/wheres-my-lunch), feel free to check it out, but don't forget, that it was created in 2 days. Don't expect the best code quality.