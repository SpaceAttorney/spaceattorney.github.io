---
layout: post
title: "Hunchback Devlog #2: Who's Graphing Now?"
categories:
- Gamedev
- Hunchback
tags: []
img_path: "/assets/img/post/2022/apr/hunch2"
---
Hello again! [Last time]({% post_url 2022-02-09-hunchback-devlog-1-the-old-to-the-new %}) on the devblog, I mentioned I'd be moving on to menu design. And, while I did do that, I quickly got sucked into a side distraction for about a month, so I'll be devoting this entry to that instead. And so it goes.

## What Happened?
I started by working on the main pause menu, the one that pops up when you press the Menu key in the overworld. I still plan on giving the menu design and code its own post later, but here it is for reference:

![The main "office" menu](office.png){: w="842" h="470"}
_The main pause menu. I've been referring to this internally as the "office menu" due to its stylization, as "main menu" makes me think of the menu that would appear when you first boot the game._

After setting the office menu up, I moved on to the Cases menu. This is where the action of _Hunchback_ is meant to happen, so I wanted to work on it as early as possible. In fact, I was so eager to get to working on the core mechanics that I even overcame my fear of default Unity assets:

![The Cases menu](cases.png){: w="835" h="471"}
_A case consists of up to four questions, plus identifying the culprit. Currently nothing here is functional except the return button._

With the mockup done, my next task was to create a sample Case for the menu to read and display. At this point, Cases were a purely theoretical construct, so I had to figure out how to translate them into readable data. This turned out to be a bit of a nightmare.

### Distraction 1: Case Data
One difference I didn't fully appreciate between _Hunchback_ and its chief inspiration, _Return of the Obra Dinn_, is that my game is supposed to be filled with living NPCs that you have a reason to talk to. This required complicating the question/answer dynamic a bit. In _Hunchback_, you can interview an NPC about a **question** tied to a case, a piece of **evidence**, or another **suspect**[^1]. If they have information about one of these things, a **detail** will be added to it. I flip-flopped a lot on how important Details should be to solving a case. In general, I want to keep the mechanics as light as possible to keep from leading players by the nose, but having a structure in place means less variables to account for when writing. It's a balance I still haven't struck, and likely won't until well into the playtesting phase.

---
[^1]: I mean, you can't. NPCs won't be implemented for a while. Man, it feels weird to talk about planned features in the present tense. But it also feels weird not to! Argh!
