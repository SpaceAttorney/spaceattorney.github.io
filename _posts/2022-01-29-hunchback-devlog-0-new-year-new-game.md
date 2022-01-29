---
layout: post
title: 'Hunchback Devlog #0: New Year, New Game'
categories:
- Gamedev
- Hunchback
tags:
- intro
- blogging
- adhd
img_path: "/assets/img/post/2022/jan/hunch0/"
date: 2022-01-29 02:41 -0600
---
Hello, and welcome to the official blog of Intermittent Games. This is the first in a series of posts detailing my development of deductive adventure game _Hunchback_. I'm calling this post "#0" because it's mostly an introduction to the project on a conceptual level, and to the blog as a whole. In the next post, I'll go over all the work I did before starting this blog, and from that point on it will be live.

## The Story So Far
This isn't the first time I've started this project. At the start of 2021 I made a resolution to start work on my first game, with the goal of delivering a playable **vertical slice**[^1] by the end of the year. It didn't go great: according to Sourcetree, my last commit was on 12 June. This year, I made the same resolution. How can I expect different results?

Well, mostly it's because I have Adderall now. I spent a good chunk of college being unable to get proper work done and never quite being sure why. I certainly never suspected any flavor of ADD - even though I'd taken a fair few psychology courses, my image of the disorder was mostly informed by jokes in my childhood cartoons. It wasn't until well _after_ college that I'd see people on a Discord server I frequent connecting it to the general listlessness and willpower issues I was experiencing.

![Anonymized post from Discord, stating "my idea of what ADHD was was 'ooh shiny'. then I read the DSM symptom list and was like 'shit.'"](adhd.jpg){: w="482", h="158" }
_Couldn't have put it better myself._

The tipping point was when the administrator of that server said
> Arguably the most important thing was to get help from a psychiatrist for my previously-untreated ADHD. This whole process would have been impossible to navigate while untreated, and ADHD is more common than people think. If you think you might have ADHD (I am not a medical professional, but you can find a questionnaire to screen you here), you should go see a medical professional about it.

If that doesn't sound suitably impressive, he said this about [teaching himself to code at a professional level in a year](https://the-skew.com/2021/07/27/how-i-went-from-college-dropout-to-google-software-engineer-in-about-a-year/).

So I went and got treated myself, and in about a week I'd remade everything from that first attempt, much of it improved. At present I feel quite encouraged about the project going forward. Hopefully this remains.

## Why the Blog?
I started this blog to help me in the following areas:
1. **Work** - Committing to regularly posting about the progress on my game necessarily means committing to regularly _making_ progress on my game. The threat of this blog publicly gathering dust as testament to my failure should help pressure me into better habits.
2. **Play** - Blogging is way easier than gamedev and provides significantly more visible output per unit of effort. Being able to go back and forth between the two should make the project more interesting and less stressful, even though it is like, objectively double the work.
3. **Teaching** - Poring through documentation is an aggravating way to learn, and I'd be glad if me doing it meant you didn't have to. [Narratives are a good way to learn](https://slimemoldtimemold.com/2022/01/08/the-didactic-novel/), and I hope a full account of my struggles as I create will serve you better than a standard plug-n'-play tutorial[^2].
4. **Learning** - I used to run a Let's Play channel with a friend of mine, and one of the things I most liked about it was the _mindset_ it put me in: trying to make regular, entertaining observations of a game meant engaging with it on a much deeper level than regular play. Similarly, when I write about my work I have to not only _review_ the topics I've learned but _reorganize_ them into a sensible, linear format, which helps a lot with both retention and overall understanding.

In addition to _Hunchback_, I'll be posting my progress on other projects as they crop up, as well as trying to review every game I play this year. Look forward to it!

## Why the game?
I play a lot of mystery games, and it's interesting to see how each one of them handles the process of playing detective. _Ace Attorney_ and _Danganronpa_ are drama series first and foremost, and they need to keep a tight grip on the pace at which you're allowed to collect and collate information for the sake of their narratives. _Her Story_ and _A Hand With Many Fingers_ go for total open-endedness, hardly gating the player's access to clues at all, but without a way to keep tabs on the player's progress, they end up having to choose arbitrary and unsatisfying endpoints. _Paradise Killer_ and the _Sherlock Holmes_ games are somewhere in the middle: they allow the player to make any of several possible conclusions according to their own logic, and while they technically don't require you to obtain every clue, it's easy to tell from the UI whether you're missing information.

![A screenshot of Sherlock Holmes: Crimes and Punishments, showing a list of specific tasks in the game's UI](sherlock.jpg){: w="1920" h="1080" }
_Image taken from Softpedia's [review](https://www.softpedia.com/reviews/games/pc/Sherlock-Holmes-Crimes-and-Punishments-Review-472425.shtml) of Sherlock Holmes: Crimes and Punishments._

There's a delicate balance to strike when it comes to making a deductive puzzler. Too much structure and you never allow the player to solve the mystery; too little, and you don't reward them for it. It's unfair for a player to miss a clue because of poor mechanics, but if you lean too far in the direction of legibility you can easily end up giving them more information than they asked for. Whether a game solves this problem satisfactorily is a matter of taste, but in my experience almost nothing does. So I'm taking a crack at it myself.

## What's the big idea?
_Hunchback_ is the working title for the deductive adventure game I'm currently making in Unity. In it, the player controls Constance, who winds up stuck in the small New England town of Widowlake due to severe weather only for a murder to occur that very night.

Planned key features include:

### Open-ended deduction
I said earlier that _almost_ nothing strikes the ideal balance of a mystery game for me. The cause of the qualifier, and my primary mechanical inspiration, is _Return of the Obra Dinn_, a game about solving the deaths or disappearances of a 60-person ship's crew. Like _Obra Dinn_, _Hunchback_ will feature a large number of intersecting cases, each of which are represented by a small set of questions that are simple but still require a complete understanding of the mystery to answer correctly.

To compensate for its more simplistic art style, _Hunchback_ places less focus on the observation side of detective work in favor of deduction. Like in _Ace Attorney_, clues you gather will be represented by text representing the relevant details about it. In _Ace Attorney_, evidence is described with a simplicity that sometimes colors the player's expectations in unfair ways - for example, if somebody says that a car alarm went off three hours before the murder and this gets added to your record, you will know that it turns out to be relevant somehow even if an in-universe investigator would rightly consider it innocuous. To get around this, I plan to introduce an _evidence-sorting_ mechanic. In _Hunchback_, when you pick up a piece of evidence it gets added to a base "Unsorted" folder, and it is up to the player to assign it to the correct case. Because the evidence for a particular case might be mixed in with evidence from other cases intially, the player must construct a model of the mystery themselves instead of relying on the game to tell them which information will be important beforehand.

Each case will be visually represented using a "conspiracy board" motif: a case consists of multiple questions, written on cards and connected with string on the board. Players pin the evidence from that case's folder onto the question that it answers or helps to answer. Similarly to _Obra Dinn_, answers are not verified as correct until a case has been completely solved.

### Fully explorable town
Inspired by the dangerous-yet-cozy locales and memorable ensemble casts of works like _Twin Peaks_, _Gravity Falls_, and _Deadly Premonition_, _Hunchback_ takes place in Widowlake, a small hamlet in Maine. I'm targeting a time period of somewhere in the late 90s or early 2000s, though there will be some deliberate anachronisms - mostly leaning on the region's colonial history - to make the setting feel more eerie, and allow it to be smaller and more remote than it could be more realistically.

The game relies on having a large number of cases to make its evidence-sorting an engaging challenge, so I plan for each resident of the town to have an associated sidequest. This will be a lot of work no matter how it's implemented, though since each case is based on only a few questions they do not need to be particularly complex to fit the structure.

### Time-bending puzzles
Though Widowlake is, in principle, fully explorable, there will be areas and clues that the townsfolk will, understandably, be reluctant to allow access to. By solving a case, however, you will gain a **favor**, which can help you progress other cases. Different NPCs can help you in different ways, e.g. by letting you into a private area, by giving previously withheld information, or by using their expertise to analyze a piece of evidence. For each case, the player can gain a favor from either its client (as a token of gratitude) or its culprit (as hush money), though not both. This choice will help allow players multiple routes to progress.

However, access to areas and clues can also be gated by **time**: each NPC has a set schedule over the course of the day, and certain scripted events might happen that change what the player is capable of. In some instances, clues will seem completely unavailable because they get locked off before the player has a chance to solve any cases. This is where the **Hunch** system comes into play. On each loop, the player can re-solve any previously solved case in a fraction of the time, simply by selecting it from a menu. This allows players to easily see alternate resolutions to a case, and it allows me as a developer to control the pacing of the game without detracting too much from the "open-world" feel it needs to have.

## Conclusion
I think that'll do for a mission statement. I plan to go into more detail on the mechanics when I get around to implementing, in large part because I expect they'll change significantly with playtesting. Next time on the devblog, I'll give a quick rundown on all the work I did prior to getting distracted by the process of setting up this website. Thanks for reading!

---
[^1]: A playable demo incorporating every planned feature of the game.
[^2]: One of my personal favorite Unity tutorials is _Gunpoint_ developer Tom Francis' [Make a Game in Unity With No Experience](https://www.youtube.com/playlist?list=PLUtKzyIe0aB3TZfe2wsIgJgGZW5G_NAxa). Tom basically just records himself adding features to a game, and leaves in every weird bug and experimental idea that crops up. As a result, it feels a lot more like actual gamedev than most tutorials.
