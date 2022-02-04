---
layout: post
title: 'Hunchback Devlog #1: The Old to the New'
categories: [Gamedev, Hunchback]
tags: []
img_path: "/assets/img/post/2022/feb/hunch1/"
image:
  src: "title.png"
  width: 1280
  height: 523
  alt: A comparison of screenshots from the old and new projects
---
As I mentioned in [the last post]({% post_url 2022-01-29-hunchback-devlog-0-new-year-new-game %}), I spent a good chunk of last year working on an earlier form of this game. When I returned to it at the start of this year, I ported what I had done to a new project, partially because re-implementing every feature one by one served as a well-needed refresher on how the code actually worked (but mostly just to sate my mind goblins). Thanks to the process of setting this blog up, as well as the [Mini Jam]({% post_url 2022-01-30-side-projects-dungeon-upkeeper %}), it's now been a few weeks since I've touched the project, so as a second refresher I think it's time to go into how everything works so far.

## The Map
To make the map, I used Unity's **Tilemap** system, on which the player character (Constance) is overlaid as a sprite. If you want to know more about it, you should check out [Ruby's Adventure](https://learn.unity.com/project/ruby-s-2d-rpg?courseId=5c5c1e08edbc2a5465c7ec01), an official Unity tutorial that taught me most of how my own map is currently laid out.

## Movement with the Input System
Last year's project was my first go at working with Unity's **Input System** package, which got out of release in 2020. I'd picked it mostly because Unity popped up a handy balloon telling me I should use it, and done so despite the advice of my professional-gamedev roommate, who had not himself worked with it and could offer no advice. What followed, I remember as a mind-numbing stumble through documentation, long and arduous enough that it inspired an entire blog post on its own (link omitted). I find this somewhat funny in retrospect, as the system is actually pretty simple, but I still find the documentation to be pretty basic. Hopefully, I can explain it better here.

There are two principal parts to the Input System: an **Input Actions** asset, and a **Player Input** component. The asset has its own editor window, which looks like this:

![The layout of the Input Actions menu, as it currently looks in Hunchback](inputactions.png){: w="1157" h="331"}

On the left of the image are your **Action Maps**. These represent different modes of accepting input, which can be swapped in code by calling `playerInput.SwitchCurrentActionMap("newMap")`[^1]. Right now, I'm using two maps: **Player**, which governs movement and initiating interactions, and **Dialog**, which governs advancing dialog boxes[^2]. A nice feature of this system is that Action Maps are mutually exclusive: for instance, if the active map is "Dialog", I can't move Constance around until it switches back to "Player".

Within each Action Map, you can create a list of Actions, and within each Action you can create a list of inputs bound to that action. In my image, you can see that WASD and the left stick of a gamepad both resolve to "Move". On the right are settings related to the selected Action. I haven't messed with these, but in general they allow you to specify how each Action is triggered[^3].

A major advantage of the Input System is that the Input Actions asset is the only place you need to worry about referencing raw input. In your actual code, you need only refer to the Actions. To see what this looks like, let's check out the Player Input component:

![The Player Input component, with an expanded list of custom events](playerinput.png){: w="362" h="398"}

The important bit here is the **Behavior** attribute. I have this set to `Invoke Unity Events`, which is relatively complicated but allows me a bit more freedom[^4]. When this behavior is set, the component will populate with an empty list for every action in the attached Input Actions asset (as well as some default ones like losing an input device). Here I've shown the lists for the two actions in the "Move" map, which have one element each. When you attach a game object (Constance, in my case) to an element, it creates a drop-down menu on the right listing possible responses. In addition to pre-populated **static parameters** (which I have no clue about), the list contains any methods you've put on that game object that take `InputAction.CallbackContext` as an argument. This is a struct that contains all kinds of readable data about a player input.

Here's what this looks like in code:
```c#
private void FixedUpdate()
    {
        myRigidbody.velocity = moveDirection * playerSpeed;
    }

    public void Move(InputAction.CallbackContext context)
    {
        moveDirection = context.ReadValue<Vector2>();
    }
```
{: file="PlayerController.cs"}

When the player moves the stick on their controller, the game identifies this as a "Move" action and takes the stick's position as a `Vector2`. The Player Input component (which I have attached to an otherwise empty object called the **Input Manager**) sees this and, in response, calls `Move()` on the `PlayerController` script attached to Constance. The `Vector2` is passed along as an attribute of `context`, and the code translates that into a movement vector.

## Interactions

---
[^1]: I'm using `playerInput` here as a generic name for an instance of the class `PlayerInput`. For future reference, I'll be doing this sort of thing for all methods. `"newMap"` is a string specifying the name of the action map you want to switch to (e.g. "Player" or "UI").
[^2]: The third one you see, **UI**, is automatically added if you create the asset by clicking "Create Actions" in the Player Input component window. Doing this also adds a default "Player" map, but I've edited mine slightly. The default UI map is pretty complex and would be a pain to re-create, so I just left it in there in case I need it later.
[^3]: For example, in the **Interactions** section you can say that an action only triggers if the button is held for a certain amount of time, or tapped several times in succession.
[^4]: The only other behavior I know much about, `Send Messages`, will attempt to call the method `OnX(context)` on every component attached to the same game object when action X is performed. I didn't use it because I wanted to be able to refer to methods in other game objects.
