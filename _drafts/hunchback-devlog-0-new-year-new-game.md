---
layout: post
title: 'Hunchback Devlog #0: New Year, New Game'
categories: [Gamedev, Hunchback]
tags: [intro, blogging, adhd]
img_path: "/assets/img/post/2022/jan/hunch0/"
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
I mentioned before that "almost nothing" strikes the ideal balance of a mystery game in my experience. The best exception to this statement, IMO, is _Return of the Obra Dinn_, which is why it's my primary inspiration. For those who don't know, _Obra Dinn_ features sixty cases - one for each person on board the titular ship. For each case, you must match three things: their name, their cause of death, and the responsible party. The first is chosen from the ship's manifest; the other two are given specific drop-down menus with an exhaustive list of possible answers. Whenever you get three sets of three correct, those answers are locked in.

So what about this works so well? I think I'm going to call it _strict simplicity_. Every case in the game is comprised of the same three questions, which are answered via the same three drop-down menus.
---
[^1]: A playable demo incorporating every planned feature of the game.
[^2]: One of my personal favorite Unity tutorials is _Gunpoint_ developer Tom Francis' [Make a Game in Unity With No Experience](https://www.youtube.com/playlist?list=PLUtKzyIe0aB3TZfe2wsIgJgGZW5G_NAxa). Tom basically just records himself adding features to a game, and leaves in every weird bug and experimental idea that crops up. As a result, it feels a lot more like actual gamedev than most tutorials.
