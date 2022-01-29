---
layout: post
title: 'Side Projects: Dungeon Upkeeper'
categories: [Gamedev, Side Projects]
tags: [game jam, minijam, minijam 98, dungeon upkeeper]
img_path: "/assets/img/post/2022/"
---
A Discord server I frequent has a lovely little channel called #workshop, in which people can post progress on, or ask for help with, any projects they're working on. A few months ago, a fellow amateur dev there mentioned wanting to try a game jam at some point, and I (having at this point been several months removed from any work on _Hunchback_) asked him to let me know if he did. Cut to the weekend of 21 January, and [the 98th rendition](https://itch.io/jam/mini-jam-98-empty) of the biweekly 72-hour game jam [Mini Jam](https://minijamofficial.com/) begins. I and this person I've never met have three days to make a game. How do we do it?

## The Idea
Each Mini Jam features a **theme**, announced alongside the jam, and a **limitation**, which is kept hidden until the start of the 72 hours. The former (in this case, **Empty**) is optional and primarily there to spur information, but the latter is mandatory, and adherence to it is one of the axes each entry is graded on. Although the intention is for everyone to have the same amount of time to work around the limitation, there's a page where you can vote on submissions, so I knew what the possibilities were ahead of time, and tried to have an idea ready for each one. For the limitations "Status effects must be a main mechanic" and "You may not kill", for instance, I thought we could make a game about a witch who has to inflict "empty" as a status effect to make her enemies too depressed to fight. When the start rolled around, I had ideas fitting all of the possible limitations except one: under "Make it a roguelike", my notes only said "IT BETTER NOT BE THIS ONE".

You'll never guess what it turned out to be.

### Making it a Roguelike
What makes this theme scary to amateurs like us is that it necessitates procedural level generation, which neither of us had experience with. This meant a _lot_ of coding work before we even had a skeleton to build a game around. But before we could do that, we had to know what we were making.

Although the limitation went to great pains to make it clear that it wouldn't be anal about exactly what separates a roguelike from a "rogue-lite", we decided to start by going for something more traditional - a turn-based, grid-based, dungeon crawler. This was mostly for efficiency reasons - we wanted the movement system to be as simple as possible because we knew the procedural generation was going to take a good fraction of our time to hammer out. In keeping with the theme, my partner (serp) suggested that the dungeon you were crawling would be empty, and I elaborated with the idea of playing as a minion who was responsible for resetting the traps after somebody ran the dungeon. We liked this idea because it presented a wide range of scope - we would start by having a simple trap that activated as you walked over it, and a treasure chest that you could refill, but if time allowed we could add extra elements like other trap types or adventurers' bodies to clean up.

The jam started at a poor time for me - 2200 Thursday, my time, which was less than halfway through a shift at work - so for a while it was just serp trying to work out the level generation algorithm without a Unity project attached to it. It wouldn't be until the next evening that I'd get to start.

## Level Generation
I wasted a bunch of time trying some fancy tricks I'd picked up from tutorials in making the levels, but the final version I eventually slapped together follows this logic:

### 1. Generate Seed
For some reason I got really hung up on the difference between "randomly generated" and "procedurally generated" over the course of this project, and that wound up teaching me a lot about memory even if it probably didn't earn us any points.

The number generator we used is a `System.Random` type, the "System" denoting that it comes not from the Unity engine but from C# in general. I declared the variable as **static**, which elevates it to global status and ensures that I'm always reading or setting the original variable (as opposed to a copy) whenever I reference it. This is important because System.Random produces a set of numbers from a seed, and for the procedural generation to be robust it needs to read these numbers in an order independent of which script is asking for it.

### 2. Generate Features
One thing I wanted when generating the maps was fine-grained control over the level's complexity as a puzzle, so I made the first step in the map generation the placement of **features**, which is a catchall term for traps, treasure chests, and any other interactive elements we managed to squeeze in there. I added variables for the number of features to throw in, as well as the map size, and made them **serializable** so they would show up in the Unity editor. In general throughout this project I erred on the side of making as much stuff as possible easy to edit this way, as I figured a procedurally generated puzzle game would take a lot of fine-tuning parameters to get to a point where it was enjoyable.

Elsewhere in the project, we have a publicly accessible list of every tile type in the game. At first, I had a complicated system in for making sure the code would always grab the right one by checking its filename, but in the end we just decided to remember which was which when referring to them. When you're working on a 72-hour deadline, it turns out there is such a thing as too elegant. In any event, the map generator creates an empty list of features, populating it first with a single start and exit tile and then appending it randomly with as many other features as is specified.

### 3. Place features
I initially envisioned the map generation as **node-and-branch-based**. What I mean by this is that the game would start by determining which features were reachable from each other first, placing them on the map in workable locations, and then drawing pathways between them. This idea was based on a tutorial for [a much more complex map generator](https://learn.unity.com/tutorial/generating-content?projectId=5c514ac8edbc2a0020694815#), and involved all kinds of craziness like a dedicated Hallway class that kept track of which coordinates were meant to be connected to each other. In the end, this idea didn't work out, due to a bug I was never able to pin down (until, uh, just now when I'm reviewing the code for this post, I think) that caused a particular room to be connected to everything. Drawing the hallways would have been a headache, too, though, so I'm ultimately glad I switched.

```c#
do
       {
           minDistance = mapSize.magnitude;
           // Debug.Log("New Start Point");
           // Find nearest unconnected neighbor of start tile
           foreach (Feature neighbor in featureList)
           {
               // Check if the features are the same
               if (startTile.position == neighbor.position)
               {
                   continue;
               }

               // Check if the hallway already exists
               if (startTile.IsConnected(neighbor))
               {
                   continue;
               }

               // Compare distance to current nearest neighbor
               float distance = Vector2Int.Distance(startTile.position, neighbor.position);
               if (distance < minDistance)
               {
                   minDistance = distance;
                   endTile = neighbor;
               }
           }
           // Add hallway to winning neighbor pair
           hallwayList.Add(new Hallway(startTile, endTile));
           Feature.AddConnection(startTile, endTile);

           // Next path will start from the newly reached tile
           startTile = endTile;

       } while (endTile.tile.name.ToLower() != "exit");
```
{: file="MapController.cs"}
_Exact nature of the error left as an exercise for the reader, but it is noticeable just from this snippet._

Funnily enough, I had a simpler version of this idea in my head, then got distracted and completely forgot about it until I looked up and noticed the method I was writing in was called `PlaceFeaturesInGrid()`. I immediately moved on to creating this simpler version, which subdivides the grid into sets of 3x3 tiles, and places features randomly in the center of these "super-tiles". In addition to the constraint that features could not be in the same spot, I eventually added another stipulation that the start and end tiles had to be a minimum distance from each other.

### 4. Connect Features
The map starts by finding the "neighbors" of each feature: that is, which other features, if any, are exactly three tiles away from the given feature in any cardinal direction. This still uses a simpler version of the **Connections** attribute from the earlier tutorial, as well as its associated methods.

Once each connection is drawn, the game determines whether every tile on the map is reachable from the entrance. It does this by repeatedly going through the list of features and setting their boolean `Reachability` attribute to **true** if any of their connections also had it as **true**.
```c#
public static void DetermineReachability(List<Feature> featureList)
{
    bool stagnant;
    do
    {
        stagnant = true;
        foreach (Feature feature in featureList.FindAll(x => !x.reachable))
        {
            foreach (Feature neighbor in feature.connections)
            {
                if (neighbor.reachable && neighbor.tile.name.ToLower() != "exit")
                {
                    feature.reachable = true;
                    stagnant = false;
                }
            }
        }
    } while (!stagnant);
}
```
{: file="feature.cs"}
_The "stagnant" bool keeps the do-loop running until the foreach-loop ceases to cause any updates._

The entire map is unlikely to be reachable at the start, so
