---
layout: post
title: 'Side Projects: Dungeon Upkeeper'
categories:
- Gamedev
- Side Projects
tags:
- game jam
- minijam
- minijam 98
- dungeon upkeeper
img_path: "/assets/img/post/2022/jan/mj98/"
image:
  src: title.png
  width: 982
  height: 556
  alt: Tutorial screen for Dungeon Upkeeper
date: 2022-01-30 01:47 -0600
---
A Discord server I frequent has a lovely little channel called #workshop, in which people can post progress on, or ask for help with, any projects they're working on. A few months ago, a fellow amateur dev there mentioned wanting to try a game jam at some point, and I (having at this point been several months removed from any work on _Hunchback_) asked him to let me know if he did. Cut to the weekend of 21 January, and [the 98th rendition](https://itch.io/jam/mini-jam-98-empty) of the biweekly 72-hour game jam [Mini Jam](https://minijamofficial.com/) begins. I and this person I've never met have three days to make a game. How do we do it?

## The Idea
Each Mini Jam features a **theme**, announced alongside the jam, and a **limitation**, which is kept hidden until the start of the 72 hours. The former (in this case, **Empty**) is optional and primarily there to spur imagination, but the latter is mandatory, and adherence to it is one of the axes each entry is graded on. Although the intention is for everyone to have the same amount of time to work around the limitation, there's a page where you can vote on submissions, so I knew what the possibilities were ahead of time, and tried to have an idea ready for each one. For the limitations "Status effects must be a main mechanic" and "You may not kill", for instance, I thought we could make a game about a witch who has to inflict "empty" as a status effect to make her enemies too depressed to fight. When the start rolled around, I had ideas fitting all of the possible limitations except one: under "Make it a roguelike", my notes only said "IT BETTER NOT BE THIS ONE".

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
I initially envisioned the map generation as **node-and-branch-based**. What I mean by this is that the game would start by determining which features were reachable from each other first, then place them on the map in workable locations, and finally draw the pathways between them. This idea was based on a tutorial for [a much more complex map generator](https://learn.unity.com/tutorial/generating-content?projectId=5c514ac8edbc2a0020694815#), and involved all kinds of craziness like a dedicated Hallway class that kept track of which coordinates were meant to be connected to each other. In the end, this idea didn't work out, due to a bug I was never able to pin down (until, uh, just now when I'm reviewing the code for this post, I think) that caused a particular room to be connected to everything. Drawing the hallways would have been a headache, too, though, so I'm ultimately glad I switched.

```c#
do
   {
       minDistance = mapSize.magnitude;
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

Funnily enough, I had a simpler version of this idea in my head, then got distracted and completely forgot about it until I looked up and noticed the method I was writing in was called `PlaceFeaturesInGrid()`. I immediately moved on to creating this simpler version, which imposes a 3x3 grid on the map and only places features at its intersections. In addition to the constraint that features could not be in the same spot, I eventually added another stipulation that the start and end tiles had to be a minimum distance from each other.

### 4. Connect Features
The map starts by finding the "neighbors" of each feature: that is, which other features, if any, are exactly one spot away on the grid (i.e. three tiles away) in any cardinal direction. This still uses a simpler version of the **Connections** attribute from the earlier tutorial, as well as its associated methods.

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
{: file="Feature.cs"}
_The "stagnant" bool keeps the do-loop running until the foreach-loop ceases to cause any updates._

If any feature fails this reachability check, an "empty" feature is added to a random unused coordinate and the check is run again, and so on until every feature on the map is reachable[^1]. Because these empty features are added randomly, when the check finally succeeds most valid spaces are filled and the map just looks like a grid[^2]. To hew it into an actual dungeon-resembling shape, a second loop is run to prune redundant features.

```c#
// Prune unnecessary features
bool pruneStagnant;
do
{
    pruneStagnant = true;
    foreach (Feature trialFeature in featureList.FindAll(x => x.tile.name.ToLower() == "floor"))
    {
        // Back up feature data and set to zero
        Vector2Int tempPos = trialFeature.position;
        trialFeature.position = Vector2Int.zero;

        // Reset reachability on all features
        foreach (Feature feature in featureList)
        {
            feature.ClearConnections();
            feature.ResetReachable();
        }

        // Redetermine map exhaustibility
        bool newExhaustible = true;
        FindConnections(featureList);
        Feature.DetermineReachability(featureList);
        foreach(Feature feature in featureList.FindAll(x => x.position != Vector2Int.zero))
        {
            if (!feature.reachable || feature.numConnections < 2)
            {
                newExhaustible = false;
            }
        }

        // Restore feature if it affected exhaustibility
        if (!newExhaustible)
        {
            trialFeature.position = tempPos;
        }
        else // Overwrite feature if it did not
        {
            mapArray[tempPos.x, tempPos.y] = wallIndex;
        }
    }

    // Prune any feature that did not survive the trial
    if (featureList.Exists(x => x.position == Vector2Int.zero))
    {
        featureList.RemoveAll(x => x.position == Vector2Int.zero);
        pruneStagnant = false;
    }
} while (!pruneStagnant); // loop executes until no more features are culled
```
{: file="MapController.cs"}
_Essentially, this temporarily removes each feature, runs the reachability check again, and puts the feature back if it fails._

### 5. Draw Corridors
The map runs the connection finder one more time to get a guaranteed-accurate picture of each feature's neighbors, then runs a simple algorithm to place floor tiles.
```c#
void DrawCorridors(List<Feature> featureList)
    {
        int floorIndex = tileList.FindIndex(x => x.name.ToLower() == "floor");
        foreach (Feature feature in featureList)
        {
            foreach (Feature neighbor in feature.connections)
            {
                Vector2Int tile1 = Vector2Int.zero;
                Vector2Int tile2 = Vector2Int.zero;

                Vector2Int difference = neighbor.position - feature.position;
                if (difference.x == 3)
                {
                    tile1 = feature.position + new Vector2Int(1,0);
                    tile2 = tile1 + new Vector2Int(1, 0);
                }
                else if (difference.x == -3)
                {
                    tile1 = feature.position - new Vector2Int(1, 0);
                    tile2 = tile1 - new Vector2Int(1, 0);
                }
                else if (difference.y == 3)
                {
                    tile1 = feature.position + new Vector2Int(0, 1);
                    tile2 = tile1 + new Vector2Int(0, 1);
                }
                else if (difference.y == -3)
                {
                    tile1 = feature.position - new Vector2Int(0, 1);
                    tile2 = tile1 - new Vector2Int(0, 1);
                }

                mapArray[tile1.x, tile1.y] = floorIndex;
                mapArray[tile2.x, tile2.y] = floorIndex;
            }
        }
    }
```
{: file="MapController.cs"}
_Since there are only two spaces between features, we can simply take each feature and lay tiles next to it for every direction in which it has a neighbor._

### 6. Check for solvability
A stipulation in the map generation is that each level is in theory _perfectly solvable_: that is, it's possible to reach every chest and reset every trap without going over the same trap twice (as doing this springs the trap and hurts you). So before the map goes live, it is run through a solvability algorithm written by serp:
```c#
public bool IsSolvable(char[][] grid, Point playerLoc)
    {
        if (grid.TileAt(playerLoc) == X) return true;
        if (IsSolvableAfterMoving(grid, playerLoc, playerLoc.North()) ||
            IsSolvableAfterMoving(grid, playerLoc, playerLoc.South()) ||
            IsSolvableAfterMoving(grid, playerLoc, playerLoc.East()) ||
            IsSolvableAfterMoving(grid, playerLoc, playerLoc.West()))
        {
            return true;
        }

        return false;
    }

    bool IsSolvableAfterMoving(char[][] grid, Point curLoc, Point newLoc)
{
    if (!newLoc.InBoundsOf(grid) || grid.TileAt(newLoc) == W || grid.TileAt(newLoc) == V ||
        grid.TileAt(newLoc) == A) return false;
    if (grid.TileAt(newLoc) == T) //if we just moved onto a trap
    {
        var newGrid = grid.Copy();
        ClearVisisted(newGrid);
        newGrid.SetTile(newLoc, A); //this trap is now armed, we don't want to walk on it again
        return IsSolvable(newGrid, newLoc);
    }
    else if (grid.TileAt(newLoc) == G) //if we just moved onto gold
    {
        var newGrid = grid.Copy();
        ClearVisisted(newGrid);
        newGrid.SetTile(newLoc, V); //this treasure chest is now filled, so treat it as visited
        return IsSolvable(newGrid, newLoc);
    }
    else if (grid.TileAt(newLoc) == X)
    {
        if (objectivesCompleted(grid))
        {
            return true;
        }

        return false; //we escaped without completing objectives, so this is a fail path
    }
    else
    {
        grid.SetTile(newLoc, V);
        return IsSolvable(grid, newLoc);
    }
}
```
{: file="MapAnalysis.cs"}
_As best I can tell, this is a recursive algorithm that traces every possible path from the start outwards until it hits either a loss or win state[^3]._

I had a couple contentions about how much "perfect solvability" mattered (which I'll get to later), so I ended up not integrating this until the very end. In the final code, then, there's a loop that just does steps #1-6 over and over again, from scratch, until it gets a perfectly solvable level. I put a "max tries" variable into most of my loops so they'd force quit if they ran too long, but this was a late addition and so in the final product the last few levels can simply fail to generate, forever (causing an Out of Memory error on the web version). For the most part, though, the map generator runs cleanly, and outputs something like this:

![An example map from Dungeon Upkeeper](samplemap.png){: w="982" h="552"}

Though there are some oddities[^4], overall this looks proper dungeony and I'm quite proud of it. But a map is only part of the goal - the next question is, how do we make it a game?

## Game Design
Although we nailed down the basic conceit of "resetting a dungeon" pretty early, there's a wide possibility space within that idea and we had some difficulties nailing down specifics. The idea in serp's head was something closer to a pure puzzle game - you had to complete every objective before you got to the exit or it was a failure, and if you walked over a trap you had already reset, you would die immediately. I liked the idea, and I think it would be a great starting point for a game, but I had two major worries about it as a jam entry.

First, the difficulty. One of the core insights I've had from playing too many video games is that game _difficulty_ is necessarily dependent on game _quality_. If your game is difficult, it had better be fun in equal measure or nobody will play it. That's why most people will play _Super Meat Boy_ but only watch _other_ people play _I Wanna Be the Guy_. So for a game made over the course of 72 hours, I wanted to go for leniency, and make sure people would stick with the game long enough to see some fun in it rather than get frustrated[^5].

Second was the jam's theme: "Make it a Roguelike". I got _really_ hung up on adhering to this theme as much as possible[^6], and I was worried leaning too far in the puzzle direction would make it feel like a different genre. The entire HUD from that earlier screenshot (health, moves remaining, score, and current floor) is, strictly speaking, not necessary for the core game, but I threw them in anyway because I felt that having values that visibly persisted between floors would give the player a "per-run" mindset instead of a "per-level" mindset. This is also where the increased map size/complexity came into play - I wanted there to be a sense of progress throughout a run instead of identical levels. I still feel rather guilty about my pitch strategy being "I implemented this feature without asking, do you like it", but in my defense it worked - our entry [scored](https://itch.io/jam/mini-jam-98-empty/rate/1366187) 24th in the "use of the limitation" category compared to 54th overall.

I was also initially rather scared of implementing serp's solvability check algorithm, as its brute-force pathfinding looked to me like it would get exponentially complicated rather quick. This turned out to be an unfounded fear, though, and if I had tried to work with it from the beginning the map generation probably could have ended up a lot smoother overall.

### Playing the Game
When the player starts the game, they're treated to a tutorial written by serp that gets across the main mechanics:
> Dungeons beckon, dragons sleep, and goblins mind the grounds upkeep!
>
> Refill the chests with jewels and gold, from all the swords and armor sold!
>
> Rearm the traps, wipe them dry, then leave them be, or else you'll die!
>
> Mind your steps, don't take too long, a hero's sure to be along!
>
> But don't you rush right to the door, the gods reward a perfect floor!

In short: hit the chests, hit the traps (but not more than once), don't let yourself run out of moves but go for a perfect clear if you can. We thought the tutorial would be necessary as there are a couple of elements that aren't immediately clear without it. For one, the player is supposed to walk into the spikes to rearm the traps - this was something it even took me a while to condition myself to, even though I designed the thing - spikes just look dangerous!

Having the tutorial be represented in goblin song is more than just a cutesy touch - we felt that attaching a narrative justification to the mechanics would help them feel more intuitive. For example, how can the player remember that moves are added at the end of the floor based on their score from that floor? Well, the "moves remaining" represent the time until a hero crawling the dungeon catches up with you, and the traps and chests are there to keep him busy, so it just makes sense that the more of those you interact with, the more time you'll have[^7].

From there, the game proper begins. The player finds themselves at one tile, and needs to get to the exit to proceed to the next level. To win the game as a whole, they must reach level B10, which will require them to reset enough traps and chests to keep their move counter from hitting 0. In this sense, each level comes with two layers of puzzle: "How can I clear the level perfectly", yes, but also "Is it _worth it_ to clear the level perfectly?" Part of the design here is a hedge against tedium - it's totally possible for the map generation to stick a single floor trap at the end of a long hallway, and I didn't want to make players feel obligated to go trip it if that happened. Instead, it becomes a decision; an easy one, in that example, but more generally the player is asked to chart not a _perfect_ path through the level but a _move-optimal_ one, which is a more challenging target to hit[^8].

If you fail to reach your goal, you get treated to a game over screen, with a different goblin verse corresponding to how you died; if you get to B10, you see the win screen, which looks basically identical but comes with more verses and a triumphant sound bite. Like the tutorial, these screens offer narrative to connect your performance with an in-universe event to make it feel more natural. Your score is also reported on these screens. Score is another mechanic that I just kind of threw in at some point, but it does serve as a basic incentive towards riskier behavior. Ending the game with spare moves does nothing for you, and you get a nice bonus when you perfect levels, so a score-minded player will want to hit a very narrow target of "barely enough moves to finish the game", using the rest of those moves to scrape together extra points where possible.

## What Did I Learn?
### Teamwork
This was my first official game jam, but I had previously helped out with a few jams put on by the group house I was living in (which boasted a decent number of actual programmers). One of the tools we had was the Scene Pez Dispenser: like a totem in a meeting that you have to be holding in order to talk, this granted special permissions to its holder. Namely, only they could edit the main scene of the game. Apparently, this was done because Unity scenes are difficult to resolve merge conflicts in, so to keep things simple I suggested doing the same for this jam.

I don't think it worked out super well. Although we were both able to work on some features simultaneously (we did this by editing copies of the main scene), a lot of serp's time was spent drawing assets or rounding up sound files because I had monopolized the scene to go down some implementation rabbit hole or another (and I'm damn sure I'm the worse coder of the two of us). Though I no doubt learned a lot from doing this, I probably did so to the detriment of the project as a whole, and I really ought to learn to be more comfortable without having to steal the reins of design.

### Memory
I spent a long time agonizing over the "proper" ways to implement certain things when the small scope meant it probably didn't matter. For instance, `Feature.cs` is a collection of variables, such as a tile, a position vector, and a list of connections. Should this be a **struct** or a **class**? `MapParameters.cs` is, as it sounds, just a list of map generation parameters including size, number of features, and how these things change per level. There's only going to be one copy of this throughout runtime - should it be **static**? Should I put it in a **singleton**? Should I do both? In the end, I have no idea if I made the right choices with any of this, but the attempt did leave me with a much better understanding of why these differences matter than I previously had.

The roguelike nature of the game also presented a number of unique problems in this regard. For example, when the player reaches a new level, the scene simply reloads, as this is the quickest way to redo all the map generation code. However, the player's values, like health, score, etc. need to be retained, so I put these in an empty game object with `DontDestroyOnLoad()`. But this posed another problem, because it meant the HUD stuck around when the game wanted to load a different scene (like one of the game over screens), so I had to write additional code disabling the canvas. There were other things, like the random number generator, that had to get shuffled around and relabeled as the project continued, but I don't remember very well exactly where they started or what problems prompted the changes. If I ever do this again I'll keep a better activity log, because I think this would've been pretty important to keep for posterity.

### Scope
For the most part, I think we handled the time limit really well. We started with the simplest possible version of our idea, and wrote down additional ideas in case we had time to implement them. When I was first putting in things like the list of tiles or the map parameters, I put in extra effort to keep these modular and easily scalable so that these ideas would be easier to add. I certainly went overboard in places: the `PopulateMap()` method has a switch/case statement, with a corresponding drop-down menu in the Inspector, asking which map generation method to use as though we'd have time to try out more than one. Even the simpler features, like the tweakable reward values, didn't end up seeing much use, and it leaves me wondering exactly how much time I could've saved if I just bit the bullet and hardcoded more things. On the other hand, even if that kind of advance planning isn't the best idea for a jam, it's undeniably a good habit to get into for long-term projects, so I'm glad I got more practice with it.

### Juice
Neither serp nor I had art experience, so for a while I was kind of resigned to the game just being a collage of rectangles. But even a little bit of "juice" - a colloquial term for the kind of audiovisual cues that make a game feel better to play - goes a long way. The sound effects and character design, which I had considered mostly an afterthought, got as many comments as more core aspects of the game and level design. Given our level of skill, I don't think we could have done any better than we did in this aspect without sacrificing a lot, but it's still worthwhile to hammer in its importance.

## Conclusion
I imagine it's going to be a while before I'm willing to sacrifice a whole weekend (or more) to another one of these, but I really enjoyed the experience, and it's a huge ego boost to have my itch.io page no longer be completely blank. I'll try and do at least a couple more this year, and be more amenable to the idea of side projects in general - but I'll have to make sure I balance that with getting the right amount of work in on _Hunchback_, or I'll never get anywhere with it.

If you want to play our game, _Dungeon Upkeeper_, here it is:
<div style="text-align:center">
<iframe frameborder="0" src="https://itch.io/embed/1366187?linkback=true&amp;border_width=2&amp;bg_color=1a1817&amp;fg_color=e6eef2&amp;link_color=e6dac3&amp;border_color=f2320c" width="554" height="169"><a href="https://s3rp.itch.io/dungeon-upkeeper">Dungeon Upkeeper by s3rp, Intermittent</a></iframe>
</div>

---
[^1]: There are a couple extra stipulations on this check, too. You can see in the code that it specifies the "reachable" neighbor cannot be the exit. That's because the level ends when you hit the exit, so you want the features to be reachable without having to go through that tile. Elsewhere, there's a second check that specifies that each feature must have at least two connections, simply because I thought dead ends would be less fun.
[^2]: Now that I think about it, as soon as I realized this I should have just had the map initialize filled with empty features instead of adding them all in a recursive loop. OH WELL
[^3]: I'm very bad at reading others' code, which came back to bite me a few times. In this code, which was written before I did anything, the map is represented as a character array, where 'X' represents the exit, 'T' represents a sprung trap that needs to be reset, etc. When I wrote the map generator, I used an array of integers, not knowing how or even if we'd be using this code, so when serp did end up implementing it he had to write a whole other function just to convert from one array type to the other. Sorry about that!
[^4]: Mostly, the maps tend to have those weird little "loops" you see in the image. This is an artifact of the "at least two neighbors" criterion - essentially, the game doesn't prune _any_ tiles that would result in a dead end, even if the whole dead end could itself be pruned safely. I caught this early, but I left it in because I thought levels would look too straightforward or artificial if I corrected it.
[^5]: From a more practical standpoint, I also wanted people to be able to finish the game so they'd be more likely to rate it during the jam's voting phase.
[^6]: I even spent like an hour implementing a custom seed field in the start menu, _just_ so I could point to it and go "look, procedural generation! Try it yourself!"
[^7]: Now that I look at it again, I guess we didn't actually say that. WHOOPS. In retrospect, I think the narrative idea came first, and the mechanics grew out of it, so that kind of thinking is useful in more ways than one.
[^8]: I mean, that's mostly because the actual math behind granting bonus moves is hidden, but still.
