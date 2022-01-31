---
layout: post
title: 'Hunchback Devlog #1: Playing Catch-Up'
categories: [Gamedev, Hunchback]
tags: []
img_path: "/assets/img/post/2022/jan/hunch1/"
image:
  src: "title.png"
  width: 968
  height: 546
  alt: A screenshot from Hunchback's demo room, showing two placeholder portraits and a dialog box with filler text
---
As I mentioned in [the last post]({% post_url 2022-01-29-hunchback-devlog-0-new-year-new-game %}), I spent a good chunk of last year working on an earlier form of this game. When I returned to it at the start of this year, I ported what I had done to a new project, partially because re-implementing every feature one by one served as a well-needed refresher on how the code actually worked (but mostly just to sate my mind goblins). Thanks to the process of setting this blog up, as well as the [Mini Jam]({% post_url 2022-01-30-side-projects-dungeon-upkeeper %}), it's now been a few weeks since I've touched the project, so as a second refresher I think it's time to go into how everything works so far.

## Movement
Last year's project was my first go at working with Unity's **Input System** package, which got out of release in 2020. I'd picked it mostly because Unity popped up a handy balloon telling me I should use it, and done so despite the advice of my professional-gamedev roommate, who had not himself worked with it and could offer no advice. What followed, I remember as a mind-numbing stumble through documentation, long and arduous enough that it inspired an entire blog post on its own (link omitted). I find this somewhat funny in retrospect, as the system is actually pretty simple; but to this day the documentation is pretty basic, so I still sympathize with my past self. Let's see if I can explain it better here.

### Input Actions
To use the Input System, you'll need two things. The first is a **Player Input** component, which I have attached to an empty game object called the **Input Manager**. The second is an **Input Action Asset**, which you attach to this component (and IIRC can create from the component's menu if you don't already have one). This asset is edited through its own window, as seen here:

![The layout of the Input Actions menu, as it currently looks in Hunchback](inputactions.png){: w="1157" h="331"}

Although the existence of these assets makes this system seem strictly more convoluted than just coding in key responses at first glance, I've come to appreciate it. The first reason for this is seen at the left of this image: the **Action Maps**. These action maps represent mutually exclusive modes of accepting input, so that WASD moves your character if and only if "Player" is the currently active map. Switching maps is as simple as calling `SwitchCurrentActionMap(string newMap)` on the component, which makes avoiding bugs about e.g. moving your player while the menu is up much easier. If you make the Input Actions asset from the component menu, it auto-populates with a standard "Player" and "UI" map, though as you can see I've added my own.

Within each map is a list of **Actions**, shown in the center (in this case, **Move** and **Interact**). Each of these actions further contains a list of inputs that will resolve to that action. You can capture inputs to add to these lists or choose them from an exhaustive menu. An advantage of this system is that this is the only time you need to deal with raw input - your code doesn't have to take specific buttons or keys into account at all.

To turn the actions into responses, I go back to the component:

![The Player Input component, with an expanded list of custom events](playerinput.png){: w="362" h="398"}

For now, we'll ignore most of the attributes and move down to **Behavior**. I initially used `Invoke Unity Events` because that's the only one covered in depth in the documentation, but it is, I think, the most robust. How it works is: each Action in your Input Actions asset appears here as an empty list. At the bottom of the image are two lists with one element each - one for movement, and one for interaction. To create a response to an action, simply add an element to its list and attach a game object (in my case, **Constance**). This will create a drop-down menu in the element listing possible responses for each of that object's components. In addition to pre-populated **static parameters** (which I have no clue about), you'll be able to select any methods you've put on that object that take `InputAction.CallbackContext` as an argument. This is a struct[^1] that has all kinds of readable data about the input.

The behavior `Invoke Unity Events` is useful for me because it allows me to have this single Input Manager make clean calls to other game objects. If you only have to interact with a single game object[^3], you can just put the Player Input component on that object and use the behavior `Send Messages`, which will attempt to call a method associated with your input on every component in its game object. To make use of this, you have to give your methods the specific name `OnX()`, where X is the name of the action in your asset[^4].

To clarify, let's go through everything that happens when I try to move my character. I tilt the stick up and to the right. This is mapped to "Move" in my Input Actions asset, which is attached to the Player Input component on my Input Manager. The component sees that the only response listed for any "Move" action is to call the method `Move()` on the PlayerController script, which is attached to Constance. This method only looks like this:

```c#
public void Move(InputAction.CallbackContext context)
    {
        moveDirection = context.ReadValue<Vector2>();
    }
```
{: file="PlayerController.cs"}

`context` is a data-rich variable, but in this case I only care about the base value, which corresponds to the vector the stick is tilted at. The actual movement is specified elsewhere.

### The Player, the Map, and Collision
At this point, I have a vector corresponding to the direction the player should move. To translate this into actual, on-the-screen movement, I use the built-in physics component **Rigidbody 2D**.

[image of the component here]

There are three **body types** available in the component of varying complexity: `Static` `Kinematic`, and `Dynamic`. You might think I would prefer the second, as I won't be needing any of the advanced physics of the third, and you'd be right - I would prefer the second! Unfortunately, it doesn't support collisions, so instead I picked `Dynamic` and just turned off gravity and rotation.

---
[^1]: **Structs** are a data type that, like classes, can store data of multiple types as **attributes**. Specific memory allocation details make structs useful for simple, temporary datasets, like briefly initializing several parameters relating to a button press[^2].
[^2]: I'm never certain whether a coding term requires this kind of footnote definition or not; I just kind of go by how confusing the term was to me personally at the time. I'd welcome feedback on this.
[^3]: Theoretically, you might be able to have a different Player Input component on each object you want to accept inputs from, but I haven't tested this and it doesn't seem to have any advantages over doing it my way.
[^4]: Another Behavior, `Broadcast Messages`, does the same thing but will attempt to call the method on the game object's children as well.
