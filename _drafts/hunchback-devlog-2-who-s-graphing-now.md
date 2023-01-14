---
layout: post
title: 'Hunchback Devlog #2: Who''s Graphing Now?'
categories:
- Gamedev
- Hunchback
tags:
- custom editors
- graphview
img_path: "/assets/img/post/2022/apr/hunch2"
---
Hello again! [Last time]({% post_url 2022-02-09-hunchback-devlog-1-the-old-to-the-new %}) on the devblog, I mentioned I'd be moving on to menu design. And, while I did do that, I quickly got sucked into a side distraction for about a month, so I'll be devoting this entry to that instead. And so it goes.

## What Happened?
I started by working on the main pause menu, the one that pops up when you press the Menu key in the overworld. I still plan on giving the menu design and code its own post later, but here it is for reference:

![The main "office" menu](office.png){: w="842" h="470"}
_The main pause menu. I've been referring to this internally as the "office menu" due to its stylization, as "main menu" makes me think of the menu that would appear when you first boot the game._

After setting the office menu up, I moved on to the **Cases** menu. This is where the action of _Hunchback_ is meant to happen, so I wanted to work on it as early as possible. In fact, I was so eager to get to working on the core mechanics that I even overcame my fear of default Unity assets:

![The Cases menu](cases.png){: w="835" h="471"}
_A case consists of up to four questions, plus identifying the culprit. Currently nothing here is functional except the return button._

With the mockup done, my next task was to create a sample Case for the menu to read and display. At this point, Cases were a purely theoretical construct, so I had to figure out how to translate them into readable data. This turned out to be a bit of a nightmare.

### Distraction 1: Case Data
One difference I didn't fully appreciate between _Hunchback_ and its chief inspiration, _Return of the Obra Dinn_, is that my game is supposed to be filled with living NPCs that you have a reason to talk to. This required complicating the question/answer dynamic a bit. In _Hunchback_, you can interview an NPC about a **question** tied to a case, a piece of **evidence**, or another **suspect**[^1]. If they have information about the topic, a **detail** will be added to it. This was the first source of complications for me.

Initially, I'd planned to have Cases be represented as **scriptable objects**[^2] which stored every question and answer, so the Cases menu could just read and compare them. But what happens when the player uncovers a Detail? We need some way to represent the _current state_ of a Case to keep track of the Details the player has uncovered, but scriptable objects aren't designed to be modified at runtime. Instances of regular classes don't have this problem, but they don't have the advantages of scriptable objects either. Eventually, I settled on having both: The `CaseData` scriptable object is where I fill in the data for each Case, and the `DynamicCase` class is where this gets integrated with the player's progress and interpreted at runtime[^3]. I spent a long time avoiding this solution because it felt like I was missing some more elegant path, but in hindsight it doesn't really seem kludgy at all. Hopefully this reflects greater understanding on my part.

The other biggest obstacle to writing the case-related scripts was that it forced me to answer a lot of questions about the game mechanics that I'd been internally vague about up to that point. While we're on the topic of Details, let's discuss what they actually _do_. When I first came up with the idea it was just "extra information", but how is that information supposed to be _used_? Let's give an example. While investigating a Case, you find a romantic note among the victim's possessions and add it to your Evidence drawer; however, when you present this note to a certain NPC they say they recognize the handwriting - but it's not from the victim's partner, and this gets appended to the evidence as a Detail. Sounds like this is the perfect evidence to answer the question "What was the motive for the crime?" But should that Detail be _necessary_ before the game will accept that evidence as a valid answer?

What about Details on Questions? The reason these exist is so that simple, vague questions on a case like "How did the culprit enter the area?" can be refined through interviews - maybe somebody was on lookout, and the culprit somehow slipped by them, or perhaps someone else remembers seeing a _monster_ nearby, which turns into a crucial lead. Here we're hit with a different dilemma: Does evidence get pinned to the Question itself (necessitating only that the player provide the correct means of entry) or do some Details require their own evidence (in this case, you might need to explain what caused the "monster" illusion)?

I'm not going to tell you how I eventually answered these questions (not least because playtesting will invariably prove those answers to be dumb). What matters is that the answers informed how I needed to set up my classes, and having any answers at all was a must before I could confidently move forward on the game. I spent a week on this, including drafting an entire case to see how my theoretical framework handled it, and though it didn't result in a single line of code I think it was well worth it to have a more solid understanding of what I want this game to be.

### Distraction 2: Dialogue Sequences
In the last devlog, we went over the **Dialogue Controller**, which iterates through scriptable objects called **Dialogue Events** in a linear list. Creating a conversation required creating an Event for each line of dialogue, keying it with any portrait or speaker updates, and adding it to the `InteractionData` component on the relevant game object. This was fine for simple conversations, but the more I thought about Details, interviews, and the like, the more I dreaded having to write any complex interaction this way. This game is meant to be really big, eventually, so an NPC could have unique responses to dozens of interview topics. Additionally, I thought it would make sense for some conversations to change based on certain parameters (like whether the player has solved that NPC's case or acquired certain Details), and such a nonlinearity would be a nightmare to set up in the current system if it was possible at all.

This is where I started looking into custom editors. And this is where the last three weeks of my life went.

## The Dialogue Editor
What I wanted out of a hypothetical Dialogue Editor would look a lot like a state machine or flowchart: I wanted to be able to place Dialogue Events on the window and connect them into a sequence, with the ability to branch off into different paths depending on the values of external variables. I figured all this would be possible, since it wouldn't be too dissimilar to Unity's built-in **Animator** function.

![The Unity Animator window](animator.png){: w="497" h="353"}
_The animation flowchart for Constance's movement. The arrows transition between "moving" and "idle" depending on the value of a boolean set by `PlayerController`._

And this was _kind of_ correct. Although the guts of the Animator window aren't accessible for me to copy directly, the experimental **GraphView** library has much of the same functionality buried within it[^4]. I hadn't worked with either custom editors or experimental libraries before, so I was way in over my head. I tried out every online resource I could get my hands on, but most of the non-original work I ended up with came from [this tutorial series](https://www.youtube.com/playlist?list=PLF3U0rzFKlTGzz-AUFacf9_OKiW_hGYIR), which IIRC was itself a combination of an entire Unity Forum thread's worth of insights. I recommend checking that out if you're interested, as this post is long enough without trying to rehash it.

We start with `DialogTree`[^5], a scriptable object containing a list of other scriptable objects of type `NodeData`. What happens when we open it?

### Loading Trees
First, the tree is opened:

```c#
[OnOpenAsset(1)]
public static bool OpenDialogTree(int instanceID, int line)
{
    // Find the game object with this ID and ensure it is a Dialogue Tree
    DialogTree dialogTree = EditorUtility.InstanceIDToObject(instanceID) as DialogTree;
    if (dialogTree)
    {
        LoadGraph(dialogTree);
        return true;
    }

    return false;
}
```
{: file="GraphSaveUtility.cs"}

This method catches any asset we try to open, attempts to cast it as a `DialogTree`, and if successful passes it to the rest of the script. Returning `true` indicates that the file is properly open and Unity can stop trying to do it some other way.

```c#
public static void LoadGraph(DialogTree dialogTree)
{
    // Add a start node to newly created (i.e. empty) dialogue trees
    if (dialogTree.nodes.Count == 0)
    {
        var startNode = ScriptableObject.CreateInstance<NodeData>();
        startNode.nodeID = Guid.NewGuid().ToString();
        startNode.portIDs = new List<string>
        {
            Guid.NewGuid().ToString()
        };
        startNode.inputPorts = new List<string>();
        dialogTree.nodes.Add(startNode);
    }

    // Open editor window and obtain associated GraphView
    DialogEditorWindow window = DialogEditorWindow.ShowWindow();
    DialogGraphView graphView = window.GetGraphView();

    // Clear graph of existing contents
    graphView.DeleteElements(graphView.graphElements.ToList());

    // Add nodes to graph
    foreach (NodeData n in dialogTree.nodes)
    {
        graphView.LoadNode(n);
    }

    // Restore connections between nodes
    foreach (NodeData n in dialogTree.nodes)
    {
        // Ignore if the node has no connections
        if (n.inputPorts.Count == 0)
        {
            continue;
        }

        // Find the node associated with this data file
        TK_Node activeNode = graphView.nodes.ToList().Find(x => (x as TK_Node).nodeID == n.nodeID) as TK_Node;

        // Find this node's input port
        Port endPort = (Port)activeNode.inputContainer[0];

        // Review each of this port's connections
        foreach (string inputID in n.inputPorts)
        {
            // Find the port that corresponds with this ID
            TK_Node inputNode = (TK_Node)graphView.nodes.ToList().Find(x => (x as TK_Node).portIDs.Contains(inputID));
            if (inputNode == null)
            {
                Debug.Log("Could not find input node.");
                continue;
            }
            else
            {
                int inputPortIndex = inputNode.portIDs.FindIndex(x => x == inputID);
                Port startPort = (Port)inputNode.outputContainer[inputPortIndex];

                // Connect active port to its input
                Edge newEdge = new Edge
                {
                    output = startPort,
                    input = endPort
                };
                newEdge.input.Connect(newEdge);
                newEdge.output.Connect(newEdge);
                graphView.Add(newEdge);
            }
        }
    }

    // Remove any nodes in the folder that are not represented in the tree
    string filePath = AssetDatabase.GetAssetPath(dialogTree);
    string assetName = Path.GetFileNameWithoutExtension(filePath);
    string nodeFolder = (Path.GetDirectoryName(filePath) + "/" + assetName + " Nodes");
    string[] nodes = AssetDatabase.FindAssets("t: NodeData", new[] { nodeFolder });
    foreach (string assetID in nodes)
    {
        string assetPath = AssetDatabase.GUIDToAssetPath(assetID);
        UnityEngine.Object asset = AssetDatabase.LoadAssetAtPath(assetPath, typeof(NodeData));
        if (!dialogTree.nodes.Contains(asset))
        {
            AssetDatabase.DeleteAsset(assetPath);
        }
    }

    // Set filename in editor window
    window.SetFilename(AssetDatabase.GetAssetPath(dialogTree));
}
```
{: file="GraphSaveUtility.cs"}

Let's go through this bit by bit:

```c#
// Add a start node to newly created (i.e. empty) dialogue trees
if (dialogTree.nodes.Count == 0)
{
    var startNode = ScriptableObject.CreateInstance<NodeData>();
    startNode.nodeID = Guid.NewGuid().ToString();
    startNode.portIDs = new List<string>
    {
        Guid.NewGuid().ToString()
    };
    startNode.inputPorts = new List<string>();
    dialogTree.nodes.Add(startNode);
}
```
{: file="GraphSaveUtility.cs"}

There are two ways to create a new Tree: from the main Editor, using Unity's `CreateAssetMenu` attribute; or by constructing and saving it from the Dialogue Editor window. Doing it the second way gives it a Start node, while the first way leaves the Tree's node list empty, so this code brings the two into parity.

```c#
// Open editor window and obtain associated GraphView
DialogEditorWindow window = DialogEditorWindow.ShowWindow();
DialogGraphView graphView = window.GetGraphView();

// Clear graph of existing contents
graphView.DeleteElements(graphView.graphElements.ToList());

// Add nodes to graph
foreach (NodeData n in dialogTree.nodes)
{
    graphView.LoadNode(n);
}
```
{: file="GraphSaveUtility.cs"}

To explain these, we'll have to jump over to some other scripts. `DialogEditorWindow` is broadly the same as in the tutorial, so I'll skip it for now; just know that `ShowWindow()` opens and/or focuses the window as necessary, and `GetGraphView()` returns an object of type `GraphView`, which contains all of the data that's represented in the window. Then the contents of the graph are deleted so that we only see the Tree we're loading in[^6]. Now, here's what `LoadNode()` looks like:

```c#
public void LoadNode(NodeData nodeData)
{
    if (nodeData is DialogNodeData)
    {
        // Handle dialogue nodes
        DialogNodeData n = nodeData as DialogNodeData;

        DialogNode newNode = new DialogNode
        {
            nodeID = n.nodeID,
            portIDs = n.portIDs,
            position = n.position,

            dialogText = n.dialogText,
            speakerText = n.speakerText,
            leftPortrait = n.leftPortrait,
            rightPortrait = n.rightPortrait
        };
        // Debug.Log("Dialogue box should say " + newNode.dialogText);

        AddDialogNode(newNode);
    }
    else
    {
        // By elimination, node must be a Start Node
        AddElement(GenerateStartNode(nodeData.nodeID, nodeData.portIDs[0]));
    }
}

public void AddDialogNode(DialogNode newNode)
{
    // Generate input port
    Port inputPort = GeneratePort(newNode, Direction.Input);
    inputPort.portName = "Prev";
    newNode.inputContainer.Add(inputPort);

    // Generate output port
    Port outputPort = GeneratePort(newNode, Direction.Output);
    outputPort.portName = "Next";
    newNode.outputContainer.Add(outputPort);

    // Generate text field for dialogue line and have it update on editor input
    TextField dialogField = new TextField();
    dialogField.name = "DialogLine";
    dialogField.label = "Dialogue Line";
    dialogField.multiline = true;
    dialogField.value = newNode.dialogText;
    dialogField.RegisterValueChangedCallback(evt =>
    {
        newNode.dialogText = evt.newValue;
    });
    newNode.mainContainer.Add(dialogField);

    // Generate button for adding new Dialogue Event fields
    var newFieldButton = new Button(clickEvent: () => { DisplayEventFieldMenu(newNode); });
    newFieldButton.text = "+";
    newNode.titleButtonContainer.Add(newFieldButton);

    // Add other fields, conditional on their existence
    if (!string.IsNullOrEmpty(newNode.speakerText))
    {
        AddSpeakerChange(newNode);
    }
    if (newNode.leftPortrait != null)
    {
        AddLPortraitChange(newNode);
    }
    if (newNode.rightPortrait != null)
    {
        AddRPortraitChange(newNode);
    }

    // Add stylesheet to node
    newNode.styleSheets.Add(Resources.Load<StyleSheet>("Editor/Node"));

    // Display dialogue node on graph
    newNode.title = "Dialogue Event";
    newNode.RefreshExpandedState();
    newNode.RefreshPorts();
    newNode.SetPosition(new Rect(newNode.position, defaultNodeSize));
    AddElement(newNode);
}

public TK_Node GenerateStartNode(string nodeID, string portID)
{
    // Initialize node
    var node = new TK_Node
    {
        nodeID = nodeID,
        title = "Start",
        portIDs = new List<string>
        {
            portID
        }
    };

    // Add output port
    var outputPort = GeneratePort(node, Direction.Output);
    outputPort.portName = "";
    node.outputContainer.Add(outputPort);

    // Prevent this port from being moved or deleted
    node.capabilities &= ~Capabilities.Movable;
    node.capabilities &= ~Capabilities.Deletable;

    // Reset layout of node
    node.RefreshExpandedState();
    node.RefreshPorts();

    // Set default node position
    node.SetPosition(new Rect(100, 200, 100, 150));

    return node;
}
```
{: file="DialogGraphView.cs"}

For each node in the Tree, `LoadNode()` checks what kind of node data it's reading and adds a corresponding node to the graph[^7]. Right now, there's only one subclass of `NodeData`, but in the future I'll need extra node types, which is why the method is structured the way it is.

`GenerateStartNode()` works the same as in the tutorial. `AddDialogNode()` works similarly, except instead of adding choices and output nodes, the button in the corner adds extra fields corresponding to the same extra event features as the old system (speaker and portrait updates). Since the fields are invisible to begin with, it's easier to tell at a glance which dialogue events do or do not have these features.

![A tree seen in the Dialogue Editor](graph.png){: w="1100" h="212"}
_A short test interaction, as seen in the Dialogue Editor._

Once the nodes are in place, it's time to connect them:

```c#
// Restore connections between nodes
foreach (NodeData n in dialogTree.nodes)
{
    // Ignore if the node has no connections
    if (n.inputPorts.Count == 0)
    {
        continue;
    }

    // Find the node associated with this data file
    TK_Node activeNode = graphView.nodes.ToList().Find(x => (x as TK_Node).nodeID == n.nodeID) as TK_Node;

    // Find this node's input port
    Port endPort = (Port)activeNode.inputContainer[0];

    // Review each of this port's connections
    foreach (string inputID in n.inputPorts)
    {
        // Find the port that corresponds with this ID
        TK_Node inputNode = (TK_Node)graphView.nodes.ToList().Find(x => (x as TK_Node).portIDs.Contains(inputID));
        if (inputNode == null)
        {
            Debug.Log("Could not find input node.");
            continue;
        }
        else
        {
            int inputPortIndex = inputNode.portIDs.FindIndex(x => x == inputID);
            Port startPort = (Port)inputNode.outputContainer[inputPortIndex];

            // Connect active port to its input
            Edge newEdge = new Edge
            {
                output = startPort,
                input = endPort
            };
            newEdge.input.Connect(newEdge);
            newEdge.output.Connect(newEdge);
            graphView.Add(newEdge);
        }
    }
}
```

In the tutorial, Edges (the connections between nodes) are serialized separately. I ran into problems with serialization that I could only solve if every Node in a Tree was also saved as its own scriptable object, and I didn't want to have to do the same for Edges, so I had to make my own method. In my version, each Node has two string Lists for GUIDs - `portIDs` has IDs corresponding to each of the node's own output ports, while `inputPorts` lists the IDs of the ports that connect to the node's input port. This loop takes every ID in each node's `inputPorts` list, finds the matching ID from the other nodes' `portIDs` lists, and then creates the associated Edge[^8].

```c#
// Remove any nodes in the folder that are not represented in the tree
string filePath = AssetDatabase.GetAssetPath(dialogTree);
string assetName = Path.GetFileNameWithoutExtension(filePath);
string nodeFolder = (Path.GetDirectoryName(filePath) + "/" + assetName + " Nodes");
string[] nodes = AssetDatabase.FindAssets("t: NodeData", new[] { nodeFolder });
foreach (string assetID in nodes)
{
    string assetPath = AssetDatabase.GUIDToAssetPath(assetID);
    UnityEngine.Object asset = AssetDatabase.LoadAssetAtPath(assetPath, typeof(NodeData));
    if (!dialogTree.nodes.Contains(asset))
    {
        AssetDatabase.DeleteAsset(assetPath);
    }
}

// Set filename in editor window
window.SetFilename(AssetDatabase.GetAssetPath(dialogTree));
```

In the saving function, which we'll see later, each Node in a Tree is saved as its own scriptable object in a folder which is auto-created based on the Tree's filename. Since Nodes in the editor window are just graphic representations, deleting one doesn't do anything to its associated data file. To prevent the folder from getting bloated with unused assets, we delete them here[^9].

Lastly, loading sets the filename in the window's toolbar. In the tutorial, this toolbar contained a text field where you could enter a filename for a Tree to have when it was saved. This method prescribes that every Tree be saved in a specific folder, though, which isn't what I want: I'm going to have a _ton_ of these things, and I'd prefer to be able to organize them at my discretion. In my version, the text field is read-only, and only set when an asset is loaded or saved.

### Saving Trees
When the "Save" button is pressed, this happens:

```c#
private void SaveGraph()
{
    // If it is a new graph, bring up the save window to determine a file path.
    if (string.IsNullOrEmpty(_filePath))
    {
        string absPath = EditorUtility.SaveFilePanel("Save graph to new asset", "Assets/Interactables", "New Tree.asset", "asset");

        // The Save Window returns an absolute path, so we convert it to a relative path before continuing
        if (absPath.StartsWith(Application.dataPath))
        {
            _filePath = "Assets" + absPath.Substring(Application.dataPath.Length);
        }
    }

    // Save the asset
    GraphSaveUtility.SaveGraph(_graphView, _filePath);
}
```
{: file="DialogEditorWindow.cs"}

It's possible to bring up the Dialogue Editor without an asset loaded. In this case, the toolbar simply displays "New Asset" in place of the file name. The first thing the save function does is check for this case. While the tutorial just throws an exception in the case of a lacking file name, I use the `EditorUtility` library to bring up a save window, allowing the user to save their new asset wherever they want. This library returns a full filepath, whereas most Unity functions use partial filepaths starting with the project's "Assets" folder, so we have to do some mangling to convert the two. When that's done, we move on to the bigger function.

```c#
public static void SaveGraph(DialogGraphView graphView, string filePath)
{
    // Initialize Dialogue Tree
    var dialogTree = ScriptableObject.CreateInstance<DialogTree>();
    dialogTree.nodes = new List<NodeData>();

    // Grab list of edges
    var edges = graphView.edges.ToList();

    // Grab nodes from graph
    foreach(TK_Node n in graphView.nodes.ToList())
    {
        // Write master parameters
        var tkData = ScriptableObject.CreateInstance<NodeData>();
        tkData.nodeID = n.nodeID;
        tkData.portIDs = n.portIDs;
        tkData.position = n.GetPosition().position;
        tkData.inputPorts = new List<string>();

        if (n.inputContainer.childCount != 0)
        {
            // Grab all edges that connect to the input port
            Port endPort = (Port)n.inputContainer[0];
            List<Edge> inputEdges = edges.Where(x => x.input == endPort).ToList();

            // Write IDs of input ports
            foreach (Edge edge in inputEdges)
            {
                Port startPort = edge.output;
                Node startNode = startPort.node;
                int startPortIndex = startNode.outputContainer.IndexOf(startPort);
                string startPortID = (startNode as TK_Node).portIDs[startPortIndex];
                tkData.inputPorts.Add(startPortID);
            }
        }

        if (n is DialogNode)
        {
            // Write dialogue event-specific parameters
            var dialogData = ScriptableObject.CreateInstance<DialogNodeData>();
            dialogData.nodeID = tkData.nodeID;
            dialogData.portIDs = tkData.portIDs;
            dialogData.position = tkData.position;
            dialogData.inputPorts = tkData.inputPorts;

            var dialogNode = n as DialogNode;
            dialogData.dialogText = dialogNode.dialogText;
            dialogData.speakerText = dialogNode.speakerText;
            dialogData.leftPortrait = dialogNode.leftPortrait;
            dialogData.rightPortrait = dialogNode.rightPortrait;

            // Save the data
            dialogTree.nodes.Add(dialogData);
        }
        else
        {
            // The node is not a subclass
            dialogTree.nodes.Add(tkData);
        }
    }

    // Create folder for nodes
    string assetName = Path.GetFileNameWithoutExtension(filePath);
    string fileDir = Path.GetDirectoryName(filePath);
    if (!AssetDatabase.IsValidFolder(fileDir + "/" + assetName + " Nodes"))
    {
        AssetDatabase.CreateFolder(fileDir, assetName + " Nodes");
    }

    // Save all nodes to this folder
    foreach (NodeData n in dialogTree.nodes)
    {
        string nodePath = fileDir + "/" + assetName + " Nodes/" + n.nodeID + ".asset";
        AssetDatabase.DeleteAsset(nodePath);
        AssetDatabase.CreateAsset(n, nodePath);
    }

    // Save the complete tree
    AssetDatabase.DeleteAsset(filePath);
    AssetDatabase.CreateAsset(dialogTree, filePath);
    AssetDatabase.SaveAssets();
}
```
{: file="GraphSaveUtility.cs"}

This one is pretty straightforward: we start by creating a new instance of the `DialogTree` scriptable object. Then we make a new instance of `NodeData` for every node in the graph, filling in the appropriate fields. Unfortunately, you can't just recast a scriptable object as a subclass, so dialogue nodes have to be declared as new `DialogNodeData` and filled in again, which is quite redundant.

After some more filename mangling, the script turns the Node Data into an asset and creates a folder for the Tree's Nodes alongside it (if necessary), saving those as well.

## The New New Dialogue System
I changed the way the game handles dialogue in the last devblog, and although I wouldn't have been surprised to have to change it again, I did think it would last more than one update. Oh well.

As a refresher, each object that's meant to be interactable has a Component called `InteractionData` attached to it. This used to contain a List of scriptable objects called `DialogEvent`s, but now all of the requisite data is stored in the Nodes of `DialogTree`s. So now the Tree is what the component contains; the old dialogue classes are outdated[^10].

Since Dialogue Trees are not necessarily straight lines, the Dialogue Controller needs a few extra functions to read them in place of simple Lists. It starts by locating the Start Node, which it does by finding the only Node in the tree with an empty `inputPorts` list. Then it runs two methods: `FindNextNode()` and `InterpretNode()`.

```c#
private NodeData FindNextNode(string outputPortID)
{
    foreach (NodeData n in _dialogTree.nodes)
    {
        if (n.inputPorts.Count > 0)
        {
            if (n.inputPorts.Contains(outputPortID))
            {
                return n;
            }
        }
    }

    return null;
}

private void InterpretNode(NodeData node)
{
    _activeNode = node;

    if (node is DialogNodeData)
    {
        DialogNodeData dialogNode = node as DialogNodeData;
        ResolveDialogEvent(dialogNode);
    }
}

private void ResolveDialogEvent(DialogNodeData node)
{
    // Refresh portraits and speaker display, if applicable
    if (node.leftPortrait != null)
    {
        _leftImage.sprite = node.leftPortrait;
    }
    if (node.rightPortrait != null)
    {
        _rightImage.sprite = node.rightPortrait;
    }
    if (!string.IsNullOrEmpty(node.speakerText))
    {
        _speakerText.text = node.speakerText;
    }

    // Display associated dialogue line
    advanceButton.SetActive(false);
    _dialogText.text = node.dialogText;
    _activeText = dialogTMP.GetComponent<TMP_Text>();
    StartCoroutine(RevealByCharacter(_activeText));
}
```
{: file="DialogController.cs"}

Like the edge connector in the loading function, `FindNextNode()` works by attempting to match a given port ID (the output) to the node that has that ID in its list of input ports. `InterpretNode()` is a sort of junction method which determines the specific type of Node being read and passes it to the appropriate handling function[^11].

After that point, the Controller functions as normal, excepting that the `Advance()` method also now needs to run `FindNextNode()` and `InterpretNode()`. The sequence ends when `FindNextNode()` returns a null value.

## Conclusion
Given how long it took to whip this editor up, and how it (currently) only offers the same functionality as the previous dialogue system, it would be easy to claim that it was a waste of time. And maybe it was! Luckily, I'm an amateur, beholden to no deadlines or expectations but my own, and as long as I'm putting the effort in and learning new things, I can waste as much time as I please.

That's not to say _I_ think it was a waste, though. Complex, non-linear interactions were always a desired feature of this project, and before I started this task I hadn't even considered how impossible they would be under the old system. Now, I think I've created a framework for these interactions that I genuinely look forward to using.

Next time: Menu talk! Once I can properly add and display a Case in the Cases Menu, I'll be happy to go over all of that.

---
[^1]: I mean, you can't. NPCs won't be implemented for a while. Man, it feels weird to talk about planned features in the present tense. But it also feels weird not to! Argh!
[^2]: For those unfamiliar with Unity, **Scriptable Object** is a class designed for data files. Instances of any class that derives from `ScriptableObject` can be created in the Editor with their own values without needing to be attached to game objects the way `MonoBehaviour`s do.
[^3]: Because evidence and suspect profiles can also accrue Details, there's a similar system in place for them.
[^4]: Since it's experimental, there's no telling what'll happen to it in future versions (for the record, I'm using 2020.3). I read they're working on something else to replace it, but hopefully it's similar enough for this post to still be helpful.
[^5]: At some point I settled on spelling it "dialog" in code and "dialogue" elsewhere. I still don't know why, but at least it's consistent now.
[^6]: As I write this, I'm realizing that there's no safety system in place for prompting the user to deal with unsaved changes on an existing Tree before loading in a new one. See, blogging _is_ a great way to review your own work!
[^7]: Like Details, nodes have to be represented as both a `ScriptableObject`-derived data class and another class (this time deriving from the GraphView library's `Node` class) for actual use.
[^8]: A funny story: for the longest time I was getting exceptions at this part, and had no idea why since the error was in pre-provided methods I couldn't access. The mistake? I'd mixed up the definitions of `input` and `output` in the default Edge class! I thought "input" referred to the port at the _start_ of the Edge (since that's where the "input" half of things usually is), but instead it meant "the input port the edge is connected to", which is at the _end_.
[^9]: Another thing I'm only catching on this review: it makes way more sense to do this check when the graph is saved as opposed to loaded. Probably not worth fixing, but worth pointing out.
[^10]: The Tree in `InteractionData` is labeled `openingTree`, since it's possible interactions will involve multiple Trees, and this component only needs to reference the one that happens on first interaction.
[^11]: Again, the Editor only contains dialogue nodes at the moment, so there's only one case implemented here.
