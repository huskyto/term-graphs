
# Overview

This project, temporary name term-graphs, aims to create a simple way to display different kinds of graphs and trees in the terminal. Ideally, it should would in a way similar as DOT, but display to the terminal, which has its own rendering and displaying challenges.

# Terminology

- Graph: the collection of nodes and relationships to be displayed.
- Node: the individual object that will be represented, and for which relationships can be displayed.
- Edge: the relationships themselves. They can be directional (arrows).
- Groups: collections of nodes that are connected directly or indirectly by edges. A graph can have many independent groups.
- Subgraphs: Parts of a graph that are kept together, and drawn as a unit. Will not be supported in the first version.

# Design Decisions

- We will focus on mostly hierarchical data for a first version. Trees will be the ideal form, but we will also be able to display graphs, but will try to display them as a hierarchy, like dot's base engine.
- Directionality. Edges can be arrows pointing to the target, to the source, both, or neither. However, for layout resolution, the source -> target relationship will be considered, regardless of arrows.
- Colors. Rim color (just color) and background color (bg_color) can be defined for nodes. Color can be defined for edes.
- Groups. Will float to a top level when completely disconnected from other groups.
- Subgraphs. Will not be supported in a first version, but will be in later versions.
- Screen fit. The first version will always try to fit the graph to the screen, at least horizontally. Future versions may consider alternate screen and paging.
- Selectable nodes and callbacks. Nodes should be transversable/selectable with the keyboard, and there should be a callback option so something is done when one is selected and/or acted.
- Implementation language. We will do the implementation in Rust, for performance, memory safety, and ease of embedding.

# Challenges

## Node positioning

Positioning nodes will be a challenge, and strategies to help and mitigate problems are discussed below in **Logic**.

## Edge generation

A particular problem for terminal rendering, since realstate is very constrained. In relation-heavy graphs it can get crowded very quickly, specially with back-references.

Strategies discussed below in **Logic**.

# Data types

## Inputs

For the first version we can probably reuse Node and Edge from intermediate representation. Maybe alias them, but it may not be necessary.

- Node list.
- Relations.
- Options. - Not there for first version.

## Input representation.

- Node {id(u32), label(String), color(Color), bg_color(Color), border(bool)}
- Edge {source(u32), target(u32), direction(Direction), decoration(Decoration), color(Color)}

## Intermediate representation

IEdge maps directly to Edge, but being only a struct, and not a Drawable enum.
INode maps from Node in a similar way, but it also has a space for the Edge's to be included.

Alternatively we could have created an 'edge repository' inside the containing graph concept, and that would have allowed us to use the same objects in both representations. However, doing it like this keeps things more separated, and allows for a cleaner API.

- Drawable: enum - Node, Edge
- INode {id(u32), label(String), color(Color), bg_color(Color), border(bool), edges(Vec[IEdge])} <- Drawable
- IEdge {id(u32), source(u32), target(u32), direction(Direction), decoration(Decoration), color(Color)} <- Drawable

- Level: enum - MainLevel, MiddleLevel
- MainLevel {Vec[Drawable]}
- MiddleLevel {Vec[Edge]}

- Stack: {Vec[Level]} - contains helpers to easily move levels around, and insert middle levels at any point.

- Color: wrapper for u32 for now.
- Direction: enum - Target, Source, Both, None
- Decoration: enum - None, Dots, Intermitent

## Pathing representation

- Cell: enum - NodeRef, EdgeRef, Empty
- NodeRef{id(u32)}
- EdgeRef{id(u32), node_id(u32), edge_type(EdgeType)}
- Empty()

- EdgeType: enum - Vertical, Horizontal, Corner(Orientation), Connection(Orientation, bool (false: 3, true: 4 ways)), Skip(u32, u32) [edge_id and node_id of the other one]

- Orientation: enum - North, East, South, West. For corners, the orientation is defined as North = Poiting North and East, and the rest are rotations. For Connection, on 3, North is: North + East + South. For 4, orientation is meaningless.

- Buffer{data(Vec[Cell])}

- Dictionary{nodes(Map[u32, INode]), buffer(Buffer)}

Edges can be resolved by retrieving the node, then checking the edges.
Buffer is sized to terminal screen size, and initialized to all Empty. Changes can be made directly on the buffer.

## Resulting representation

- ResNode {id(u32), callback(Option<?>), hover_callback(Option<?>), loc_data(LocData)}
- LocData {pos(Vec2), size(Vec2), neighbors([Option[u32]; 4])} - neighbors is for keyboard traversal. Ordered clockwise from North.
- ResGraph: {nodes(Map[u32, ResNode]), lines(Vec[String])}

# Logic

## Passes

### 1. Mapping

Go through the input Nodes and Edges.

1. Generate an INode for each Node, with empty edges list.
2. Go through each Edge. And create an IEdge, assigning an increasing id.
3. If either the source or target nodes of the edge doesn't exist; create it with default properties.
4. Add the IEdge to the source's edge list.

Allow for edge duplication.
Node duplication should result in overwriting defined properties.

### 2. Grouping

Group connected nodes, and process each group separately, to simplify generation.

### 3. Layering

#### 3.1 Relative Layering

- Pick the first node on the list and assign it a layer of `0` (i32).
- Follow its targets; each target is set to `source_node_layer - 1`. If not is already place, set to `min(target_node_layer, source_node_layer - 1)`
- Keep a work list, as to not to process nodes twice, and also to know what nodes have not been processed.
- If you run out of relations, pick an unprocessed node, set layer to `0` and process with temporary worklist.

#### Work lists

The will be a main work list that holds the completely processed nodes. We can call it `trunk`.

When working with nodes, detached from the the trunk (if you ran out of relations in the trunk) keep all processed nodes in a temporary work list.

This leads to two cases:

1. If a relation hits an existing node in the main branch, `trunk` (and it's not in the temporary work list) resolve all existing nodes by moving them by the layer of the target node, adding the nodes to `trunk` and clear the temporary worklist.

2. If you run out of relations, and the main branch is never touched... first cry, then pick a new unprocessed node and try again.

Nested here, there can now be three options:

1. Same as the previous one. You hit the trunk, and resolve, and continue.

2. You run out of relations. Same as before, pick a new unprocesses node.

3. You hit an orphaned list from before. This will be treated similar to hitting `trunk`, moving the nodes in the current temp list to the hit temp list, and displacing them by the layer of the target node.

To simplify, we can always consider that we are in nested mode, with the three cases.

### 4. Staggering

Once the layers are set, if there is one (or more) layer that is too crowded, stagger it into two layers instead.

Example:

- 1, 2, 3, 4, 5, 6, 7, 8

Becomes:

- 1, 3, 5, 7
- 2, 4, 6, 8

Remember that you need to push all the nodes below one level too.

Note: We should have a way to tell the layout that the new second level should be skewed half a node to the right, so they are actually staggered, and connections are made easier and less messy.

### 5. Weighting

Try to align parents to the children.

(How?)

### 6. Fill Node positions in buffer.

Calculate ideal node size and assign the cells in the buffer, leaving space between nodes; probably three cells by default.

### 7. Connections.

Draw down to the next middle from the center of the source node.
If empty, insert a vertical EdgeRef to buffer.

Calculate horizontal and vertical delta between position and target's center (how? we need to keep all positions and sizes. TODO).

Determine the natural movement left vs right, up vs down.

Walk towards the target's center following horizontal natural movement, checking the cell below or above for empty, depending on vertical natural movement. If an Empty is found, try to traverse down or up to the middle level above the target.

If the horizontal middle of the target is passed, and no empties were found, backtrack to last EdgeRef found on up/down check. Then try to move all the cells right for the NodeRef level in that direction, as well as the MiddleLevels above it, checking there is actually space at the end to properly displace them. (how do we know what is a MiddleLevel or a NodeLevel? we should have a per-line reference somewhere. TODO)

If the vertical middle of the target is passed, backtrack in much the same way, except that if going up, the new MiddleLevel should be added at the bottom of the MiddleLevels above the NodeLevel.

Then walk to be above the target's middle.

Finish the connection.


If at any point, cells used by EdgeRefs are encountered, resolve as following:

- If Edge has same Source; merge. (and keep reference for the place where they will unmerge)
- If Edge going oposite (Horizontal vs Vertical) with different source; change to Skip, and skip that cell. If adding that skip would push the patting a level too down for the target, insert another middle level for it to path trough.



### 8. Final draw

Draw in buffer.

Nodes should be:

```
+-------+
| Label |
+-------+
```

The space between the label and the `|` can be ommited if space is constrained.


Edges are defined by type.

- `|`: Vertical.
- `-`: Horizontal.
// TODO add proper unicodes, or preferably ascii, for the other options. Maybe update vertical and horizontal too.


Empty space is simply a blankspace.
