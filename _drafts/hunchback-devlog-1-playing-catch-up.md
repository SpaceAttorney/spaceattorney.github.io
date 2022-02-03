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

For now, we'll ignore most of the attributes and move down to **Behavior**. I initially used `Invoke Unity Events` because that's the only one covered in depth in the documentation, but it is, I think, the most robust. How it works is: each Action in your Input Actions asset appears here as an empty list. At the bottom of the image are two lists with one element each - one for movement, and one for interaction. To create a response to an action, simply add an element to its list and attach a game object (in my case, the player character **Constance**). This will create a drop-down menu in the element listing possible responses for each of that object's components. In addition to pre-populated **static parameters** (which I have no clue about), you'll be able to select any methods you've put on that object that take `InputAction.CallbackContext` as an argument. This is a struct[^1] that has all kinds of readable data about the input.

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

![An image of the Rigidbody2D component](rigidbody.png){: w="353" h="322"}

There are three **body types** available in the component of varying complexity:
- `Static`, which has no physics attached. As the name implies, this is for objects which never move but which you still want things to collide with. I use this for interactable objects and bounding boxes.
- `Kinematic`, which allows the object to move linearly but does not apply forces
- `Dynamic` (pictured), which includes a full suite of simulated physics parameters but is computationally costly to match

Since _Hunchback_ is a simple, top-down adventure game, Constance has no need for acceleration or gravity, and so you might think I'd prefer to use `Kinematic` for them. And you'd be right - I would prefer that! Unfortunately, however, you cannot apply external forces of any kind to kinematic objects, and that includes collisions. Instead, I have to pick `Dynamic` and change around a few things. Changing the **Gravity Scale** to `0` prevents the character from falling, and enabling **Freeze Rotation** keeps Constance from rotating[^5]. I've also set **Collision Detection** to `Continuous` here - it's a more accurate but more costly model that stops your character from being able to move through entire objects if they're going fast enough. I don't think it'll be necessary, in retrospect - it's just a habit drilled into me by some earlier tutorials - but this is a simple enough game that the inefficiency is unlikely to make a difference.

The Rigidbody component doesn't itself provide a hitbox, however. To get that, you need to add another component: a **Collider**. Unity offers several different shapes of collider, but for Constance (and interactable objects) I use the simple **Box Collider 2D**.

#### Tilemaps
To make the levels, I use Unity's **Tilemap** system. Tilemaps are located under **2D Objects** in the Asset menu, and creating one automatically creates a **Grid** object as well, which the tilemap is a child of. Tilemaps are painted on using a **Tile Palette** and **Tiles**, two bespoke asset types used for this, but the process is very simple. After you create a tile palette, creating a tile is as simple as dragging a sprite onto it.

![My tile palette, as it currently stands](tilepalette.png){: w="377" h="591"}

Right now I only have two tiles. At the bottom is a 3x3 grid of a wooden floor tile. The reason for this is that, using the selector at the top, you can select drag across multiple tiles in the palette to paint with all of them at once. This would be good for painting preset patterns of tiles as well, if I had them. The black square at the top is my "boundary tile". This is the same color as the game's background[^6], so in-game it's invisible. The reason it's useful is that it has a collider. **Collider Type** is a parameter of any Tile asset, so it's simple to turn on.

Well, not _that_ simple. To enable collision, you also have to add the **Tilemap Collider 2D** component to the tilemap, as well as a static **Rigidbody 2D**. By default, the tilemap collider will treat every tile with a collider as its own hitbox, which usually causes other objects to hitch on the edges between tiles. This is solved by adding a **Composite Collider 2D** and enabling the **Used by Composite** parameter on the tilemap collider, which merges the colliders into one.

### Animation
Considering my primary goal right now is to nail down and showcase the core mechanics of the game, animation is really not something I should be implementing this early. But I did it in the previous project for some reason, so it wasn't much work to re-implement it here.

Much like everything else so far, animation is handled by a component called the **Animator**. Unity's animation system relies on two asset types: The **Animation**, and the **Animator Controller**. The former is the actual clip - the series of sprites as tied to particular keyframes[^7]. The latter is a visual flowchart, which determines which animation is playing at any given time.

![The Animator window, showing Constance's animation flow](animator.png){: w="687" h="348"}

When Constance is loaded, the animator starts on **Entry** and immediately goes to **Idle**. When the **Moving** parameter (on the left) is set to `True`, the animator transitions into the **Moving** state (and vice versa). A simple two-frame loop is attached to this state, so that Constance bobs up and down as they move[^8].

[Insert moving Constance here - capturing on laptop would probably require Movie Studio, which ehhhhhh]

The movement-relevant code is as follows:
```c#
void Update()
   {
       // Update look direction if the player is moving
       if (!Mathf.Approximately(moveDirection.magnitude, 0.0f))
       {
           lookDirection.Set(moveDirection.x, moveDirection.y);
           lookDirection.Normalize();
           animator.SetBool("Moving", true);
       } else
       {
           animator.SetBool("Moving", false);
       }

       // Flip the sprite based on current look direction
       if (lookDirection.x > float.Epsilon)
       {
           mySprite.flipX = true;
       }
       else if (lookDirection.x < -float.Epsilon)
       {
           mySprite.flipX = false;
       }
   }
```
{: file="PlayerController.cs"}

Every update[^9], if the player is moving (recall that `moveDirection` is set by the method `Move()`, seen earlier), the game will transition from the **Idle** to the **Moving** state until they stop. The game also calculates whether the player is looking right or left and flips their sprite accordingly, using a parameter native to the **Sprite Renderer** component[^10]. This negates the need to have a separate sprite imported for this purpose, saving a bit of time and storage space.

### Code
I've already shared most of the code. The only thing remaining[^11] is this:
```c#
private void FixedUpdate()
 {
     myRigidbody.velocity = moveDirection * playerSpeed;
 }
```
{: file="PlayerController.cs"}

The actual line is simple: `playerSpeed` is a float set in the Inspector to translate the movement vector into a velocity that is then input into Unity's physics simulator. The only stickler, really, is the method. `Update()` runs every frame, i.e. as fast as you can run the game, but `FixedUpdate()` only runs at a set rate. Generally, you're only supposed to refer to the physics engine in this latter method, as you want the physics to run at the same speed on every computer that plays the game.

Okay, so far this has basically just been a rehash of some standard functions I picked up from tutorials. Here's where I actually started working.

## Interaction
The last method in the Player Controller script is this:
```c#
public void Interact(InputAction.CallbackContext context)
   {
       if (context.performed)
       {
           // Find center of Constance's hitbox for slightly better interactions maybe
           BoxCollider2D myHitbox = gameObject.GetComponent<BoxCollider2D>();
           float hitboxOffset = myHitbox.size.y / 2;
           Vector2 hitboxCenter = myRigidbody.position;
           hitboxCenter.Set(hitboxCenter.x, hitboxCenter.y + hitboxOffset);

           RaycastHit2D hit = Physics2D.Raycast(hitboxCenter, lookDirection, interactDistance, LayerMask.GetMask("Interactables"));
           if (hit.collider != null)
           {
               InteractionData thisInteraction = hit.collider.GetComponent<InteractionData>();
               if (thisInteraction != null)
               {
                   EventManager.beginDialog.Invoke(thisInteraction);
               }
           }
       }
   }
```
{: file="PlayerController.cs"}

Unlike movement, interaction is handled by a single button press, so in this case the only part of `CallbackContext` I care about is the boolean `performed`, which tells me whether the Interact key was pressed[^12]. If it was, the game checks if there's an object in front of Constance using the **Raycast** function.

First, the game calculates the center of Constance's hitbox. Constance's **pivot point** is the bottom center of their sprite[^13], but raycasting from there made it too easy to miss an object, so this was a quick hack to fix it. As you can tell from the tone of the comment, I'm not sure how well it works, but it's better than nothing.

Next, I invoke the built-in method `Raycast()`, which sends out an invisible beam. `lookDirection` is set earlier, in the movement code, and here it provides the direction for the beam. `interactDistance` is set in the editor, and dictates how far the beam goes. The **Layer Mask** argument allows you to specify which layers[^14] the beam can actually register a collision with.

The game first checks to see if the raycast hit something, then whether that thing has an **Interaction Data** component attached. If both checks succeed, the Interaction Data is passed through the Event Manager.

### Event Manager
The Event Manager is something I picked up from my roommate, a professional game dev, at one of our in-house game jams a few years back. As far as I can tell, it's meant to be a cleaner way of passing data between scripts than calling a public function. The Event Manager looks like this:

```c#
public class EventManager : MonoBehaviour
{
    public class StringEvent : UnityEvent<string> { }
    public class InteractEvent : UnityEvent<InteractionData> { }

    // For switching UI functionality.
    public static StringEvent switchInputMap = new StringEvent();

    // For beginning a dialog.
    public static InteractEvent beginDialog = new InteractEvent();
}
```
{: file="EventManager.cs"}

Let's just look at `beginDialog` for now. This is an **Interact Event**, which is defined above as a **Unity Event** that takes an Interaction Data as an argument. This is the event that the raycast invokes. When the event is invoked, any method that has an attached **listener** gets called. To see what that looks like, we introduce the **Dialog Controller**:

```c#
void Start()
{
    EventManager.beginDialog.AddListener(BeginDialog);

    // Make sure the dialog contraption is invisible.
    dialogBox.SetActive(false);
    leftPortrait.SetActive(false);
    rightPortrait.SetActive(false);
}

public void BeginDialog(InteractionData thisInteraction)
{
    // Activate and clear the dialog box.
    dialogBox.SetActive(true);
    dialogText = dialogTMP.GetComponent<TextMeshProUGUI>();
    dialogText.text = "";

    dialogList = new List<DialogEvent>(thisInteraction.interactionLog); // queues list of dialog events for the active interaction

    // Resolve the first dialog event.
    DialogEvent activeEvent = dialogList[0];
    dialogList.RemoveAt(0);
    ResolveEvent(activeEvent);

    EventManager.switchInputMap.Invoke("Dialog"); // switches player input to dialog mode
}
```
{: file="DialogController.cs"}

At the start of the program[^15], a listener is added for this event and attached to the method `BeginDialog()`. Note that this method takes Interaction Data as an argument. When the `beginDialog` event is invoked, the Interaction Data is passed to this method even without an explicit connection between the two different scripts.

This is where the other event, `switchInputMap`, is invoked as well. The Input Manager has a listener for this, and switches the input action map accordingly, so that when `BeginDialog()` runs the player can no longer move.

But what actually is Interaction Data? And what are all these other things?

### The Structure of Interactions
**Interaction Data** is a script that is attached to every interactable object. It consists of nothing but a list of **Dialog Events**. So, what's a Dialog Event?

Before I answer that, let's go over how interacting with an object works in the abstract. When you interact with an object, a dialog box pops up and a line of dialog is displayed. Sometimes, a portrait of the speaking character pops up, or updates to show a change in expression. Sometimes the speaker changes, which needs to be represented in the dialog window. But sometimes nothing changes except the line being displayed.

Each element of an interactable's Interaction Data is a Dialog Event[^16]. But actually, this means nothing in and of itself: `DialogEvent`, as a class, is totally empty. It only exists because **Lists** can only be composed of a single class. The real data is stored in its child classes.

`DialogEvent`'s first subclass is `LineUpdate`, which contains a string representing a single line of dialogue. Next up is `PortraitUpdate`, which 

---
[^1]: **Structs** are a data type that, like classes, can store data of multiple types as **attributes**. Specific memory allocation details make structs useful for simple, temporary datasets, like briefly initializing several parameters relating to a button press[^2].
[^2]: I'm never certain whether a coding term requires this kind of footnote definition or not; I just kind of go by how confusing the term was to me personally at the time. I'd welcome feedback on this.
[^3]: Theoretically, you might be able to have a different Player Input component on each object you want to accept inputs from, but I haven't tested this and it doesn't seem to have any advantages over doing it my way.
[^4]: Another Behavior, `Broadcast Messages`, does the same thing but will attempt to call the method on the game object's children as well.
[^5]: Unity kind of treats your hitbox as though it's pinned in place only at a single point (the **pivot point**), so without this enabled most collisions would cause Constance to rotate. And due to some of my simplifications in the physics, they'd usually cause them to rotate _insanely fast_.
[^6]: The background color is a property of the scene's camera. By default it's a shade of blue, so I had to change it.
[^7]: The animator is definitely something I don't have enough experience with to really talk about. It looks like you can change basically any property of a game object in the animation, which is really neat, but I haven't tried it.
[^8]: The idle state also has an animation attached, but it's just a static sprite. I don't recall if this is necessary, but I think it is.
[^9]: Every script you create in Unity defaults to inheriting from the class **MonoBehavior**, which gives it some default methods. `Update()` is a method that executes on every frame.
[^10]: Originally, I believe I had the sprite-flip set in the Animator, but the way it was implemented led to Constance looking the wrong direction a lot of the time. This way is much simpler.
[^11]: Aside from variable declarations, which I assume aren't necessary to talk about? I'll do that later if someone asks me to.
[^12]: It's a little more complicated than this, since the input system splits every action into multiple phases, but this works for my purposes.
[^13]: The point that Unity treats as its "center", so to speak. Constance's pivot point (as well as everything else's, for that matter) is the bottom center of their sprite. This is part of a perspective-forcing technique I picked up from a tutorial, wherein sprites are sorted according to the Y-position of their pivots such that lower sprites are always drawn on top.
[^14]: Every game object can be assigned a **Layer** in the inspector. Layers can be used to categorize objects and make processes like this one selective. There are also **Sorting Layers**, which affect draw order, and **Tags**, which are like Layers but can't be used by certain Unity features. Tags are useful in larger projects, since there is a strict limit to how many Layers a project can have.
[^15]: `Start()` is another MonoBehavior default. It is called once, before any `Update()` calls can begin.
[^16]: `DialogEvent` inherits from **Scriptable Object** rather than MonoBehavior, so it can be used as an asset instead of as a component. This is what allows it to be added to lists like this.
