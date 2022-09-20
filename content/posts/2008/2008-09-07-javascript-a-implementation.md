+++
title = "Javascript A* Implementation"

[taxonomies]
tags = ["javascript", "algos"]

[extra]
location = "San Diego"
+++

So a little background on my implementation (feel free to skip this)....

In my freshman year at UCSD I was recruited by [CSES](http://cses.ucsd.edu)
to work on a small game project. It was supposed to be generic enough so
that the frontend (my game engine) could connect to a back-end that could
be written in any language as long as it conformed to a set of standards we
had. Things like retrieving player position, setting player position,
getting maps, etc. So long story short, I needed a pathfinding algorithm
for the game engine, and my experiment with the A* algorithm began.

<!-- more -->

## A Little Theory

For those who don't know, A* is a best-first, graph searching algorithm
based on the infamous [Dijkstra's
Algorithm](http://en.wikipedia.org/wiki/Dijkstra%27s_algorithm). The A*
algorithm has been used across the spectrum of applications. The most
popular use is path-finding in games, but I've spotted it being used even
web-based apps like Google Maps or MapQuest (at least I think so).

First of all, an important thing to note about A* is that it uses a
heuristic. This speeds up the running time considerably, but as a
consequence, although guaranteed to find a path, it is not always
necessarily the *best* path.

For the best path, you could use Dijkstra's, but that's not always viable.
Dijkstra's running time is a whopping O(V<sup>2</sup>), where V is the
number of vertices in the graph) compared to A* running time of O(E + V)
(best best case where E is the number of edges and V is the number of
vertices in the graph. Another thing to note is that, A\*'s running time is
a bit more complex because of the heuristic you can use, but for the best
case, with a simple heuristic, you can get a runtime very similar to
BFS([Breadth First Search](http://en.wikipedia.org/wiki/Breadth-first_search)).
Using different heuristics can vary the running time which can increase accuracy
(although with a loss of speed)

The algorithm is given a start and goal coordinate and when run finds a
path from start to end. Obstacles (for example, in a game a large boulder
could be an obstacle) are introduced into the algorithm by removing edges
from the graph, forcing the algorithm to take into account "holes" in the
graph.


## Implementation

Now that we've gotten the theory behind A* out of the way, lets dive into
my implementation.

First off we have the Node object.

``` javascript,linenos
function Node(parentNode, cx, cy, g, h) {
  this.parent = parentNode;
  this.x = cx;
  this.y = cy;
  this.g = g;
  this.h = h;
  this.f = g + h;
};
```

A node represents a certain vertex on a graph. The parent of a Node
represents the vertex in which the algorithm came from to reach this node,
this provides a link to previous node in case we want to backtrack, as well
as an easy way to extract the path the algorithm took. `X` & `Y` are simply
the graph positions. Now `g`, `h`, and `f` are "scores" for the node `g` is
the actual shortest distance from the source node to the current one `h` is
the estimated distance (this is the heuristic) distance from the current
node to the goal `f` is simple the sum of `g` and `h`. `f` is used to
generate a priority for nodes, which has the end result of nodes that are
closer to the goal are picked over others.

Now lets take a look at the core stuff itself. The code that actual finds
the path for us.

``` javascript,linenos
Astar.prototype = {
  // Various Vars
  _open: [],
  _closed: [],
  _path: [],
  _avail: null,
  _sx: 0,
  _sy: 0,
  _dx: 0,
  _dy: 0,

  // Public Functions
  findPath: {},

  // Private Functions
  _CreateNode: {},
  _fillChildren: {},
  _getBestNode: {},
  _isAvailable: {},
  _printArray: {},
  _printNode: {},
  _ValidTile: {}
};
```

Lets take a look at the variables and functions used in the implementation.
`_open`, `_closed`, and `_path` are all used as stacks `_open` holds a list
of nodes that are still "open", which means that a node has not been
visited by the algorithm `_closed` hold a list of nodes that have been
visited by the algorithm `_closed` is actually not used by the algorithm in
any way, but was only there for debugging purposes so that I can see the
nodes the algorithm accessed during tests `_avail` is an array that is used
to quickly determine whether or not a node has been accessed. _sx, _sy,
_dx, and _dy are the source and destination coordinates.

Now for the functions `findPath` is the only public function, and returns
the path the algorithm found `_CreateNode` `_fillChildren` `_getBestNode`
an `_isAvailable` are used by the algorithm to find the path.

The algorithm itself is in findPath and is shown below...
``` javascript,linenos
for(var count = 0; count < 100; count++) {
  bestNode = this._getBestNode();
  if( bestNode == null ) break;
  this._path.push( bestNode );
  if(bestNode.x == this._dx && bestNode.y == this._dy)
    break;
  this._fillChildren( bestNode );
}
```

A simple for loop is all it takes. The reason I used a for loop is to check
against impossible paths. The number of nodes the algorithm accesses should
never be more than the number of vertices in the graph, which in my test
case was 100 (10x10 grid). All this loop does is retrieve the next node,
check if it has reached it's destination and then if not, add the children
of node to the open list, where the process is started all over.

So how is the best node retreived?
``` javascript,linenos
Astar.prototype._getBestNode = function() {
  if(this._open.length > 0) {
    var bestNum = 0;
    var bestF = this._open[0].f;
    // Cycle through open list until the best node is found.
    for( var idx = 1; idx < this._open.length; idx++) {
      var tmpF = this._open[idx].f;
      // Favors the last added node if there are
      // nodes with the same F.
      if(tmpF <= bestF) {
        bestNum = idx;
        bestF = tmpF;
      }
    }
    // Remove node from open list and place into closed
    // list.
    var bestNode = this._open[bestNum];
    this._open.splice(bestNum, 1);
    this._closed.push(bestNode);
    return bestNode;
  }
  return null;
};
```

Since the `_open` list is unordered (a quality that we can easily fix to
improve the runtime even more), I loop through the entire list to find the
node with the lowes `f` score. Because of the nature of how nodes are
added to th `_open` list, this favors nodes that have been last added
when there are nodes with the same `f` score. The node is then removed
from the `_open` list and return to be processed some more.

So far, you've only seen how nodes are picked, but the key to this
algorithm is how the node scores are determined, because the rest is simply
a BFS using these scores as a priority.

``` javascript,linenos
Astar.prototype._fillChildren = function(node) {
  // Top (North) Tile.
  if( this._ValidTile(node.x, node.y-1) )
    this._CreateNode(node, node.x, node.y-1);
  // Right (East) Tile.
  if(this._ValidTile(node.x+1, node.y))
    this._CreateNode(node, node.x+1, node.y);
  // Bottom (South) Tile.
  if(this._ValidTile(node.x, node.y+1))
    this._CreateNode(node, node.x, node.y+1);
  // Left (West) Tile.
  if(this._ValidTile(node.x-1, node.y))
    this._CreateNode(node, node.x-1, node.y);
};

Astar.prototype._CreateNode = function(node, x, y) {
  // Check to see if node is not on closed list
  if(this._isAvailable(x, y)) {
    var tmp = new Node(node, x, y, node.g+1,
        distance(this._dx - x, this._dy - y));
    avail[y*NUM_ROWS + x] = 1;
    this._open.push(tmp);
  }
};
```

`_fillChildren` grabs the children around a specific node and adds them
to the open list. In this case, I only choose the nodes directly above,
below and to the sides. However, this can easily be expanded where all the
surrounding nodes are added. So first a potential node is checked for
validity (is it out of bounds? has it already been added?) and then if it
is valid, the node is created and added to the `_open` list. The `g`
score is incremented because it is one square farther than its parent from
the source node. The `h` score in the algorithm uses the [Manhattan
distances](http://en.wikipedia.org/wiki/Manhattan_distance) formula which
is the sum of the absolute differences of two coordinates Changing this
formula to other ones can speed up the process of pathfinding with worse
results or impede the process with better results.

## References

* [A* Algorithm Wikipedia Entry](http://en.wikipedia.org/wiki/A*_search_algorithm)
* [An excellent page containing tons of information about variations you can make](http://theory.stanford.edu/~amitp/GameProgramming/)
