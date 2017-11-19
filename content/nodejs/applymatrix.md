+++
date = "2017-11-19T09:23:22+02:00"
draft = false
title = "PaperJS: What does applyMatrix do?"

+++

PaperJs is great library for building scene hierarchies and virtual worlds (e.g. game worlds). It is somewhat beginner-friendly; the documentation could be better, but for the most part, PaperJS library simply does what is expected.

However, there is one big gotcha that tripped me over when I started using PaperJs; behaviour of *applyMatrix* -attribute.

Lets start with an example. I want to build a christmas-themed scene. 

This scene is pretty simple; it has one single room, with nicely decorated Christmas tree standing in the middle of the room. 

Something like this should achieve our setup of the scene:

```
  // Our room is equivalent to PaperJs global project coordinate system.
  // In other words, top-left corner of the room is point [0,0] in global space.

  // Lets create scene.
  // Start by creating a Group that holds all objects for our Christmas tree.
  var xmasTree = new paper.Group({});

  // Place the xmasTree Group to the middle of the room.
  xmasTree.position({x: 0.5 * roomMaxX, y: 0.5 * roomMaxY});

  // Create a tree 
  var tree = new paper.Path.Rectangle(
    // Tree's relative position within xmasTree group
    new paper.Point(0, 0),
    // Tree trunks size
    new paper.Size(20, 80)
  );
  tree.fillColor = 'green';

  // Add tree as a child of our xmasTree group
  xmasTree.addChild(tree);

  // Create tree decorations, and add to xmasTree group.
  // ...

```

Code above looks like it gets the job done. What we are doing above is:

1. Create xmas tree group that'll logically group together all individual objects (actual tree, christmas balls, candles, etc.) of the xmas tree.
2. Place the group into the middle of the room.
3. Add a tree to the group, and place it to relative (to the group!) position of {0,0}.
4. Add decorations (not shown in the code)

Logically that should do it, but what you'll see in the screen is something quite else.

![Xmas tree NOT in the middle of the room](/blog/public/img/tree-in-corner.png)

The actual tree (green rectangle) is of correct size, but it is not in the middle on the room!

What happened? We clearly specified that our Group object (xmasTree) is placed to middle of the room. Then we created child object for that group, and placed it to position {0,0} relative to the Group.

Or is it relative to the Group? If you look at the code closely, we specify tree's position BEFORE specifying the tree is a child of the xmasTree group. Maybe you could solve the issue by setting tree's position AFTER its group membership:


```

  // Start by creating a Group that holds all objects for our Christmas tree.
  var xmasTree = new paper.Group({});

  // Place the xmasTree Group to the middle of the room.
  xmasTree.position({x: 0.5 * roomMaxX, y: 0.5 * roomMaxY});

  // Create a tree 
  var tree = new paper.Path.Rectangle(
    // Tree's relative position within xmasTree group
    new paper.Point(0, 0),
    // Tree trunks size
    new paper.Size(20, 80)
  );
  tree.fillColor = 'green';

  // Add tree as a child of our xmasTree group
  xmasTree.addChild(tree);

  // NEW! Now that tree is a child or xmasTree, lets re-set tree's position!
  tree.position = {x: 0, y: 0};

  // Create tree decorations, and add to xmasTree group.
  // ...

```

Does this help? No. Nothing changes. Our green tree rectangle is still not in the middle of the room.

![Xmas tree still NOT in the middle of the room](/blog/public/img/tree-in-corner.png)

Next we might think: "hmm, what if we also re-set group's position AFTER adding tree as its child":

```
  // Start by creating a Group that holds all objects for our Christmas tree.
  var xmasTree = new paper.Group({});

  // Place the xmasTree Group to the middle of the room.
  xmasTree.position({x: 0.5 * roomMaxX, y: 0.5 * roomMaxY});

  // Create a tree 
  var tree = new paper.Path.Rectangle(
    // Tree's relative position within xmasTree group
    new paper.Point(0, 0),
    // Tree trunks size
    new paper.Size(20, 80)
  );
  tree.fillColor = 'green';

  // Add tree as a child of our xmasTree group
  xmasTree.addChild(tree);

  // NEW! Now that tree is a child of xmasTree, lets re-set tree's position!
  tree.position = {x: 0, y: 0};

  // MORE NEW! Now that tree is child of xmasTree, lets re-set group's position!
  xmasTree.position({x: 0.5 * roomMaxX, y: 0.5 * roomMaxY});

  // Create tree decorations, and add to xmasTree group.
  // ...

```

Does this help? Yes! Now the tree is in the middle of the room.

![Xmas tree NOT in the middle of the room](/blog/public/img/tree-in-middle2.png)

So the problem was that our group's global position got set too early; when we later added a tree (the green rectangle) as xmasTree's child, group's position did not *propagate* to its new child object. Thus, the tree-object got position relative to the **global project space**. Thats why it was right next to the screen edge in the first screeshot.

We - of course - want it to be positioned in terms of the xmasTree group; that is, we want xmasTree to create its own **local coordinate space**, and we want all child objects to be positioned relative to that space!

> Understanding the difference between *global coordinate space* versus *local coordinate space(s)* is absolutely crucial; you can not work with PaperJs without ability to transform one space to another. Of course, all the calculations are being performed by PaperJS, but you should at least understand *why* local coordinate spaces are needed.
>
> 
>
> Think about our real world, and how it forms a hierarchy of local coordinate spaces. You have latitudes and longitudes, and those help you find - for example - a route to Tokyo. But when you are in the Tokyo, it is much more convenient to use some *local coordinate space* that is relevant only inside Tokyo. That coordinate space is probably arranged using street names etc. 
>
>
>
> Then, you go into a restaurant in Tokyo. Inside the restaurant you won't use street names anymore. When a waiter gives you directions to restaurant's toilet, she will talk in terms of *restaurant's local coordinate space*: "take the stairs down and turn left, you'll find our restroom there".

So lets get to it. How do we create a local coordinate space that actually *stays alive* for more than a single function call?

## applyMatrix = false

The name of game is this: paperJs Group-objects have an attribute named *applyMatrix*, which controls the *lifetime* of group's local coordinate space!

In our code example, we did not care about applyMatrix-attribute, allowing paperJs to set it to whatever value it wants. And, perhaps bit questionably, paperJS uses *applyMatrix = true* as a default value (for Groups).

Setting applyMatrix to true means this: whenever we do some transform operation on the Group-level, that operation is *instantly* applied to Group's children.

> We have been using *positioning* as an example of more general concept called *transform/translate operation*. Positioning is not the only one; there are other transform/translate operations like scaling, rotating, skewing etc. Importantly, *exactly* same rules apply to all transform operations! All these individual operations combine into a concept called *transformation matrix*, and each PaperJS object has its own transformation matrix. This matrix is - very informally - a set of *mirrors, lenses and magnifying glasses* that define how the actual object looks from a particular point of view.

This means that if we set Group's position to - lets say - {x: 20, y: 30}, what we are actually doing is setting the origin of the Group's local coordinate space to global coordinate space point {x: 20, y: 30}. 

Notice that this is exactly what we want! We want to define our group's position in relative to the global space. However, with applyMatrix === true, *this new position is **not** stored anywhere in the Group object*; instead, for each child a new global position is calculated and object is rerendered when the position of the group is being set.

Now think about this - what happens if you set a new position for a Group with *no children*?

It is a no-op! Literally. Nothing happens. Because the group tries to calculate new position of each of its children, but there are none - thus there is nothing to calculate.

When you later add a child to the group, you might expect its position to be relative to the position of the group you previously set. But it can not be so. Because... applyMatrix is true means that *the group does not store its own position in its own transformation matrix*. 

Its exactly like telling an Alzheimer's patient to remember numbers 3 and 5. Later, we ask that same patient to sum up the two numbers he was told earlier with a number 2. What will he answer? 10? Nope. He will answer 2.

Taking all this into account, we come to a solution:

```
  var xmasTree = new paper.Group({});

  // Important!!! 
  // ApplyMatrix must be set false before setting position of the Group!
  xmasTree.applyMatrix = false;

  // Place the xmasTree Group to the middle of the room.
  xmasTree.position({x: 0.5 * roomMaxX, y: 0.5 * roomMaxY});

  // Create a tree 
  var tree = new paper.Path.Rectangle(
    // Tree's relative position within xmasTree group
    new paper.Point(0, 0),
    // Tree trunks size
    new paper.Size(20, 80)
  );
  tree.fillColor = 'green';

  // Add tree as a child of our xmasTree group
  xmasTree.addChild(tree);

  // Create tree decorations, and add to xmasTree group.
  // ...

```

Now everything works correctly and, importantly, *does not depend on the order of setting group position versus child position*. Whenever you add new child objects (Christmas balls, tree candles, presents under the tree, etc.) to our xmasTree group, they will get automatically positioned correctly.

![Xmas tree NOT in the middle of the room](/blog/public/img/tree-in-middle2.png)

And more importantly, if you ever reposition our xmasTree object, all its child will "get carried" with the group. This is then just what we want.

```
  // Woman of the household decides xmasTree should be moved to the corner of the room   
  xmasTree.position({x: 0, y: 0});

  // Whole xmasTree is now correctly moved to the corner.

```

## Quiz

Lets take a test.

Take a look of the following code snippets, and determine what is the position (in terms of the global space!) of the tree object.

### 1

```
  var xmasTree = new paper.Group({});
  // Group global position set to {x: 100, y: 0}, right?
  xmasTree.position = {x: 100, y: 0}

  var tree = new paper.Path.Rectangle(
    // Local (or is it?) position is {x: 0, y: 0}
    new paper.Point(0, 0), 
    // Size is irrelevant, lets say 1x1.
    new paper.Size(1, 1)
  );

  xmasTree.addChild(tree);

  // Whats the global x-coordinate offset of tree: 0 or 100?

```

### 2

```
  var xmasTree = new paper.Group({});
  
  var tree = new paper.Path.Rectangle(
    // Local (or is it?) position is {x: 0, y: 0}
    new paper.Point(0, 0), 
    // Size is irrelevant, lets say 1x1.
    new paper.Size(1, 1)
  );

  xmasTree.addChild(tree);

  // Group global position set to {x: 100, y: 0}, right?
  xmasTree.position = {x: 100, y: 0}

  // Whats the global x-coordinate offset of tree: 0 or 100?

```

### 3

```
  var xmasTree = new paper.Group({});

  // Group global position set to {x: 100, y: 0}, right?
  xmasTree.position = {x: 100, y: 0}

  xmasTree.applyMatrix = false;

  var tree = new paper.Path.Rectangle(
    // Local (or is it?) position is {x: 0, y: 0}
    new paper.Point(0, 0), 
    // Size is irrelevant, lets say 1x1.
    new paper.Size(1, 1)
  );

  xmasTree.addChild(tree);

  // Whats the global x-coordinate offset of tree: 0 or 100?

```

### 4

```
  var xmasTree = new paper.Group({});

  xmasTree.applyMatrix = false;

  // Group global position set to {x: 100, y: 0}, right?
  xmasTree.position = {x: 100, y: 0}

  var tree = new paper.Path.Rectangle(
    // Local (or is it?) position is {x: 0, y: 0}
    new paper.Point(0, 0), 
    // Size is irrelevant, lets say 1x1.
    new paper.Size(1, 1)
  );

  xmasTree.addChild(tree);

  // Whats the global x-coordinate offset of tree: 0 or 100?

```

> Answers below...
>
> 
> 
> ...
>
> 
> 
> ...
>
> 
> 
> ... bit more...
>
> 
> 
> ...
>
> 
>
> Answers:
>
>
>
> 1: 0
> Reason: applyMatrix = true, setting Group position too early is no-op!
>
>
> 2: 100
> Reason: applyMatrix = true, setting Group position after adding child.
>
>
> 3: 0
> Reason: applyMatrix = false, but it is set false AFTER group position setup.
>
>
> 4: 100
> Reason: applyMatrix = false, and set false before anything else.