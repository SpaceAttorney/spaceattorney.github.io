---
layout: post
title: 'Hunchback Devlog #1: The Old to the New'
categories:
- Gamedev
- Hunchback
tags: [input system, singleton, unity events, textmeshpro]
img_path: "/assets/img/post/2022/feb/hunch1/"
image:
  src: title.png
  width: 1280
  height: 523
  alt: A comparison of screenshots from the old and new projects
date: 2022-02-09 03:16 -0600
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

## Permanence
Some elements of the game, such as the player character and the canvas, are located under a game object called the **Permanent Container**. This contains a simple script designed to make the object (and its children) part of a **singleton**, like so:
```c#
public class PermanenceManager : MonoBehaviour
{
    public static PermanenceManager Instance;

    void Awake()
    {
        // Destroys object and replaces it with the initial instance (unless it *is* the initial instance)
        if (Instance == null)
        {
            DontDestroyOnLoad(gameObject);
            Instance = this;
        }
        else if (Instance != this)
        {
            Destroy(gameObject);
        }
    }
}
```
{: file="PermanenceManager.cs"}

When the first scene that contains a Permanence Container is loaded, that Permanence Container is given `DontDestroyOnLoad` status, which lets it persist across multiple scenes. This is important for maintaining data, such as the clues you've gathered, when you transition between areas. If another Permanence Container is queued to load, it will notice that one already exists, and promptly destroy itself. This is useful because the elements within the singleton will need to be in every scene for testing purposes, but would cause bugs if they were allowed to duplicate[^5].

## Interactions
Interacting with an object is initiated by sending out a **raycast** and checking whether it hits an interactable object. That's also covered by [Ruby's Adventure](https://learn.unity.com/project/ruby-s-2d-rpg?courseId=5c5c1e08edbc2a5465c7ec01), so I won't belabor it. The actual dialogue that happens afterward required original code, and it might not surprise you to learn that in the midst of drafting this post I changed the system around considerably.

### Canvas Setup
The canvas is located under the Permanent Container, and contains a single object, called the **Dialog Contraption**. This contains the **Dialog Controller** script, and it has two portrait sprites and a dialogue box[^6] as children. In the old code, I had a massive section in `DialogController.Start()` where it would grab references to each of its children using various indices of `gameObject.transform.GetChild(x).gameObject` because I didn't want to have to manually assign each child of the Dialog Contraption in every scene. In the new project, the Permanent Controller is a prefab, so the relationships I assign within it should stay in new instances.

Another minor difference is that, in the old code, there are two distinct "speaker boxes", one on the right and one on the left, so the speaker's name can pop up on the same side of the screen as their portrait. The new project integrates the text box with the speaker's name and the text box containing the dialog into the same sprite, so if I wanted to re-implement this functionality I'd have to change it a bit. For now, I've put doing so on the backburner.

![A comparison of the old and new dialog systems](dialogcomp.png){: w="1036" h="1055"}
_Top: Old project. The speaker's name is on a separate game object, rendered on top of the dialog box. Bottom: New project. The speaker and dialogue text boxes are both attached to the same sprite._

### Interaction Data
The dialogue that occurs when you interact with a valid object is stored in the object, in a component called **Interaction Data**. This component contains only a list of **Dialog Events**, which are scriptable objects containing the elements of the interaction including dialogue lines and portraits to display.

In the old project, `DialogEvent` was an empty class at the top of a hierarchy of subclasses[^7]. Below it was `LineEvent`, which contained the dialog to display; below that was `PortraitUpdate`, which contained a new sprite, and below that was `SpeakerUpdate`, which contained the speaker's name and a boolean indicating which side of the screen they were on. This system is attractively elegant, in a way: logically, every time the speaker changes, Ill want to update their portrait to show this, so every `SpeakerUpdate` will necessarily be a `PortraitUpdate` but not vice versa. Likewise, every portrait update will come with a new line of dialogue, but not every line will need to be accompanied by a new portrait. Overall, this system made basic sense, and each asset only contains the fields that will be used, which is convenient.

However, it's not very robust. When I was trying to explain it in a previous draft, I realized that new interactions I wanted to add would not fit into this clean hierchical structure. For instance, I intend to add a "Clue Event" that adds a clue into your inventory when inspecting evidence. But there's no guarantee on whether this will accompany a portrait change or not, so it's unclear what exactly this event type would inherit from[^8]. Hence, in the new project, `DialogEvent` has no children and simply contains all the data itself.

### Dialog Controller
The first thing the Dialog Controller does is make sure that the Canvas is invisible:
```c#
void Start()
{
    EventManager.beginDialog.AddListener(BeginDialog);

    // Make sure the dialog contraption is invisible.
    dialogBox.SetActive(false);
    leftPortrait.SetActive(false);
    rightPortrait.SetActive(false);
}
```
{: file="DialogController.cs"}

I use `GameObject.SetActive()` to hide the canvas elements because it's simpler than trying to suss out and target the actual render component, but this method can be tricky since you can only access data from active objects. This means I have to make sure to reactivate these objects before I try to do any updates.

You can also see a reference to the **Event Manager** in this snippet. **Unity Events** are a technique I picked up from a roommate (and professional game dev) at one of our in-house game jams a few years back. As far as I can tell, it's meant to be a cleaner way of passing data between scripts than calling a public function. The Event Manager looks like this:
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

`beginDialog` is classified as an **Interact Event**, a Unity Event that takes an Interaction Data as an argument[^9]. The Dialog Controller has a **listener** for this event, attached to the local method `BeginDialog()`. This means that `DialogController.BeginDialog()` will run whenever the `beginDialog` Event is invoked. In this case, the event is invoked when the player successfully interacts with an object that contains an Interaction Data to pass.

So, what does `BeginDialog()` do when called?
```c#
public void BeginDialog(InteractionData thisInteraction)
{
    // Switch player input to Dialog Mode
    EventManager.switchInputMap.Invoke("Dialog");

    // Activate and clear the dialog box.
    dialogBox.SetActive(true);
    dialogText = dialogTMP.GetComponent<TextMeshProUGUI>();
    dialogText.text = "";

    speakerText = speakerTMP.GetComponent<TextMeshProUGUI>();
    speakerText.text = "";

    // Activate and clear portraits.
    leftPortrait.SetActive(true);
    leftImage = leftPortrait.GetComponent<Image>();
    leftImage.sprite = defaultSprite;

    rightPortrait.SetActive(true);
    rightImage = rightPortrait.GetComponent<Image>();
    rightImage.sprite = defaultSprite;

    // Create dialog list.
    dialogList = new List<DialogEvent>(thisInteraction.interactionLog);

    //Resolve the first dialog event.
    DialogEvent activeEvent = dialogList[0];
    dialogList.RemoveAt(0);
    ResolveEvent(activeEvent);
}
```
{: file="DialogController.cs"}

First, the input map is switched to Dialog mode, so the player cannot move until the interaction is over[^10]. Each element of the canvas is enabled and set to a default empty value (`defaultSprite` is just an empty .png). The dialog list is then taken from the invoking object and the first event is resolved.

```c#
private void ResolveEvent(DialogEvent activeEvent)
 {
     // Iterate through and resolve each potential feature of the event.

     if (activeEvent.leftPortrait != null)
     {
         leftImage.sprite = activeEvent.leftPortrait;
     }

     if (activeEvent.rightPortrait != null)
     {
         rightImage.sprite = activeEvent.rightPortrait;
     }

     if (!string.IsNullOrEmpty(activeEvent.newSpeaker))
     {
         speakerText.text = activeEvent.newSpeaker;
     }

     // Display line (every dialog event should contain a line)
     advanceButton.SetActive(false);
     dialogText.text = activeEvent.dialogLine;
     activeText = dialogTMP.GetComponent<TMP_Text>();
     StartCoroutine(RevealByCharacter(activeText));
 }
 ```
 {: file="DialogController.cs"}

 In the old code, this function would check what type of Event was next in the list and run a separate function depending on the answer. This does basically all the same work in a much shorter block of code. When everything is updated, the event starts a **coroutine**, a function that runs in parallel with the rest of the game, which is necessary for functions that run on a timer, as we'll see.

 ```c#
 IEnumerator RevealByCharacter(TMP_Text textComponent)
 {
     textComponent.ForceMeshUpdate(); // i have no idea what this does lmao

     TMP_TextInfo textInfo = textComponent.textInfo;

     int totalCharacters = textInfo.characterCount; // Get number of characters in line
     int visibleCount = 0;

     while (visibleCount <= totalCharacters)
     {
         // Increment number of visible characters in line
         textComponent.maxVisibleCharacters = visibleCount;
         visibleCount++;

         yield return new WaitForSecondsRealtime(textDelay);
     }

     advanceButton.SetActive(true);
 }
 ```
{: file="DialogController.cs"}

As the flippant comment may tell you, this is not original code - I cribbed it from an example scene provided in **TMP Examples and Extras**, which you can import from **Window > TextMeshPro** if you have the **TextMeshPro** package installed[^11]. This coroutine takes advantage of TMP-specific features to only show a specific portion of the current line, which is incremented by one character after `textDelay` seconds.

The final piece of the Dialog Controller is the **Advance Button**:
```c#
public void Advance(InputAction.CallbackContext context)
{
    if (context.performed)
    {
        StopAllCoroutines();

        // Fully display the current line if not already visible
        if (activeText.maxVisibleCharacters < activeText.textInfo.characterCount)
        {
            activeText.maxVisibleCharacters = activeText.textInfo.characterCount;
            advanceButton.SetActive(true);
            return;
        }

        // Close dialog apparatus if interaction is over
        if (dialogList.Count == 0)
        {
            dialogBox.SetActive(false);
            leftPortrait.SetActive(false);
            rightPortrait.SetActive(false);

            EventManager.switchInputMap.Invoke("Player"); // switches control back to Player Mode
            return;
        }

        // Resolve next line in list.
        DialogEvent activeEvent = dialogList[0];
        dialogList.RemoveAt(0);
        ResolveEvent(activeEvent);
    }
}
```
{: file="DialogController.cs"}

`Advance()` is called when the player presses the Advance key, and moves to the next dialog event if and only if the current line is fully displayed. If that isn't the case, it forces the full line to be visible, and a second press is required to move to the next event. This added control is why moving to the next event (and ending the interaction) is handled here on button press, rather than having the event list be handled by a `foreach` loop.

## Art
The most obvious difference between the two projects is that I redid the sprites in a higher resolution. I know I'm no artist, and I know that I'm working on a prototype where the asset quality shouldn't matter, but it's still very difficult for me to avoid wasting large amounts of time polishing assets to the best of my ability. Each project represents a different attempt at a strategy to mitigate this: in the former, using a low resolution to lower the bar for "passable"; in the latter, combining a higher resolution with a more limited palette to give the whole thing a "notebook doodle" aesthetic. Neither has really worked on the time-saving front so far, but the current project looks better so I guess I'll stick with it going forward.

## Conclusion
After a year on the old project and a couple of weeks on the new one, we're at a point where we can walk around and look at an object to display some dialog. Not super impressive, but that's all the more reason to start working even harder for the next blog entry.

My next goal for the project is to begin work on the pause menu, which is tougher than it sounds since it's where the most important gameplay is going to take place. So, see you whenever that's done!

---
[^1]: I'm using `playerInput` here as a generic name for an instance of the class `PlayerInput`. If you don't know already, common Unity parlance (or C in general?) is to differentiate between classes and class instances with this sort of case difference. Also, `"newMap"` is a string specifying the name of the action map you want to switch to (e.g. "Player" or "UI").
[^2]: The third one you see, **UI**, is automatically added if you create the asset by clicking "Create Actions" in the Player Input component window. Doing this also adds a default "Player" map, but I've edited mine slightly. The default UI map is pretty complex and would be a pain to re-create, so I just left it in there in case I need it later.
[^3]: For example, in the **Interactions** section you can say that an action only triggers if the button is held for a certain amount of time, or tapped several times in succession.
[^4]: The only other behavior I know much about, `Send Messages`, will attempt to call the method `OnX(context)` on every component attached to the same game object when action X is performed. I didn't use it because I wanted to be able to refer to methods in other game objects.
[^5]: Right now I worry I may be overdoing it with how much stuff I'm subordinating to the singleton, so maybe don't copy me in putting *everything* that stays constant between scenes there. But hey, if it causes any problems down the line I'll be sure to blog about it.
[^6]: I'm used to writing it "dialogue", but "dialog" is common parlance in code (presumably because it's shorter) so I kind of flip-flop on usage. I try to keep it consistent in my own way, but it's probably confusing anyway and I'm sorry.
[^7]: The reason for having an empty class at the top is because you can only make a **List** of a single class type, and having the Dialog Events arranged into a List makes it way easier to construct interactions in the Inspector.
[^8]: I could technically have wrangled it into functionality by making it a child of `SpeakerUpdate` and just had any redundant fields "update" with the same values as the previous event, but avoiding needless redundancy was the whole point of the old system, so having to do something like that is a red flag.
[^9]: I made the events **static** mostly because that's how they were in the game jam code I was copying. I think it's this way so the Event Manager script doesn't actually have to be in the scene anywhere to function.
[^10]: The listener for this event is on the Input Manager, and all the attached function does is switch input maps as described above. It's _slightly_ cleaner than just calling a public function.
[^11]: TMP provides other benefits, too. One I'm using in the prototype is that you can put HTML tags in a string and TMP will parse those as it displays text. This way I can give certain words or lines different colors without any scripting at all.
