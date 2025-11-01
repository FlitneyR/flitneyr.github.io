---
title: Topological Sort - Picking graphs apart
layout: post
---

In my attempts to make understand game engines, and how they render complex scenes, I've come across the idea of Render Graphs.

Render graphs are a type of dependency graph: an acyclic, directed graph which expresses dependencies between nodes.

Render graphs allow you to represent the rendering work of an engine as distinct stages which depend on each other. This is a very high-level abstraction that makes reasoning about rendering much easier. But you will need to record command buffers at some point, so you need some way of turning this graph into a list of instructions (or multiple, parallel lists), and that's where topological sorting comes in.

Topological sorting is a process to extract nodes in a graph in some order determined by the topological structure of the graph (i.e. the relationships between the nodes). You could sort nodes by any number of metrics, but the key metric we're interested in here is a "dependency order".

> A node A is said to "depend on" a node B, if: \
> It has a direct dependency on node B, or \
> It depends on node C, which depends on node B

We can formalise this with the following statement in set theory. First, let $$N$$ be the set of nodes, and $$E$$ the set of directed edges. And let an edge be represented by a tuple of (source node, destination node).

We can define a relationship $$d$$ which holds for any pair of nodes where the first node depends on the second. We define this relationship as follows

$$ \forall_{ a \in N, b \in N }: d(a, b) \implies (\exists_{ e \in E }: e = (a, b)) \vee (\exists_{ c \in N }: d(a, c) \wedge d(c, b)) $$

This is basically just a transitive relationship. For example in the set of natural numbers:

$$ \forall_{ a, b, c \in \mathbb{N} } : a \le b \wedge b \le c \implies a \le c $$

We could state this explicitly for our relationship $$d$$ as:

$$ \forall_{ a, b, c \in N } : d(a, b) \wedge d(b, c) \implies d(a, c) $$

So sorting by this transitive relationship seems natural, but it will require traversing the graph somehow. We could use the above rules and compare all the nodes, in a brute force algorithm, finding a node with no dependencies, removing it from the graph and adding it to our output, repeating this until the graph is empty. Here's how finding that next node might look in pseudocode:

{% highlight plaintext %}

FUNCTION extractNextNode(graph) DO
    FOREACH node IN graph DO
        IF graph CONTAINS other WHERE node DEPENDS ON other THEN
            NEXT node
        ELSE
            REMOVE node FROM graph
            RETURN node
        ENDIF
    ENDFOREACH

    RETURN NONE
ENDFUNCTION

{% endhighlight %}

This will work. But we can do better. For one, every time we extract a node from the graph, we need to traverse every other node to check if it depends on this node. In the worst case, this algorithm has a time complexity of $$O(n^2)$$. Also, this just finds the next node with no dependencies, but we might want to give nodes some kind of priority.

Here's an alternative algorithm, [Kahn's algorithm](https://en.wikipedia.org/wiki/Topological_sorting#Kahn's_algorithm):

{% highlight plaintext %}

FUNCTION extractNextNode(graph, searchNodes) DO
    IF searchNodes IS EMPTY THEN
        RETURN NONE
    ENDIF

    nextNode = TAKE FROM searchNodes

    FOREACH node IN graph DO
        IF node DEPENDS ON nextNode THEN
            REMOVE node DEPENDENCY ON nextNode
        ENDIF

        IF node HAS NO DEPENDENCIES THEN
            REMOVE node FROM graph
            ADD node TO searchNodes
        ENDIF
    ENDFOREACH

    RETURN nextNode
ENDFUNCTION

{% endhighlight %}

This could improve slightly on the first algorithm. Instead of checking every node against every other node, we keep a cache of nodes that we know to have no dependencies, and select a node from them. This caching of previous work is a kind of dynamic programming. Then when we ripple the effect of removing the node from the graph, by removing dependencies on that node, and then removing any nodes which have no more dependencies and adding them to our cache of "ready" nodes.

In the worst case, where `searchNodes == graph`, this will have a complexity of $$O(n^2)$$. And on a more realistic level, creating the `searchNodes` structure could be expensive (for example, it may incur more cache misses).

So long as `searchNodes` remains relatively small, this algorithm should perform better than the brute force algorithm. A more interesting benefit of this algorithm is the versatility of this `searchNodes` structure. It could be a simple first in first out queue, or random order set. But if we give our nodes a weight or priority, then we could also use a priority heap, to nudge the order of our un-ravelled nodes without needing explicit dependencies. And this requires no real change to the algorithm. Neat!

Now let's look at another dynamic programming approach. To explain this approach, I'll start with a table, showing you how we will build this table, how we will transform this table, and how we will use it to sort nodes.

First the initial state of the table:

{% highlight plaintext %}

| Index | Depth | Dependencies |
| 0     | 0     | _ _ _ _ _    |
| 1     | 0     | 1 _ _ _ _    |
| 2     | 0     | 1 0 _ _ _    |
| 3     | 0     | 0 1 1 _ _    |
| 4     | 0     | 0 0 0 1 _    |

{% endhighlight %}

What does this table say? On the left we have an index for each node. Then we have a weight for each node. Then we have a bit mask of dependencies.

So node 1 has a 1 in the 0th bit of the dependency bit mask, this indicates that node 1 has a dependency on node 0. Node 2 has a 1 in bit 0, indicating it also has a dependency on node 0. Node 3 has a 0 in bit 0, a 1 in bit 1 and a 1 in bit 2. This means that node 3 has a direct dependency on nodes 1 and 2, but not node 0.

Some of the bits in the dependency bit mask are left blank. This represents how a node cannot have a dependency on itself, or a node that comes after it. This is a simple way to prevent cyclic dependencies.

They all have a depth of 0, but we'll get back to that later.

And now the final state of the table:

{% highlight plaintext %}

| Index | Depth | Dependencies |
| 0     | 0     | _ _ _ _ _    |
| 1     | 1     | 1 _ _ _ _    |
| 2     | 1     | 1 0 _ _ _    |
| 3     | 2     | 1 1 1 _ _    |
| 4     | 3     | 1 1 1 1 _    |

{% endhighlight %}

Now, node 3 has a 1 in bit 0 of its dependency mask, and node 4 has a 1 in every bit of its bit mask (except the last, which would imply a self-dependency). This is because node 4 depended on node 3, which depended on nodes 1 and 2, which depended on node 0. So node 4 had an indirect dependency on all the previous nodes.

Now we see that the nodes have different depths, let's take a look at that. To see how we got here, let's walk through the steps to create this table:

{% highlight plaintext %}

| Index | Depth | Dependencies |
| 0     | 0     | _ _ _ _ _    | < working on
| 1     | 0     | 1 _ _ _ _    |
| 2     | 0     | 1 0 _ _ _    |
| 3     | 0     | 0 1 1 _ _    |
| 4     | 0     | 0 0 0 1 _    |

| Index | Depth | Dependencies |
| 0     | 0     | _ _ _ _ _    |
| 1     | 1     | 1 _ _ _ _    | < working on
| 2     | 0     | 1 0 _ _ _    |
| 3     | 0     | 0 1 1 _ _    |
| 4     | 0     | 0 0 0 1 _    |

| Index | Depth | Dependencies |
| 0     | 0     | _ _ _ _ _    |
| 1     | 1     | 1 _ _ _ _    |
| 2     | 1     | 1 0 _ _ _    | < working on
| 3     | 0     | 0 1 1 _ _    |
| 4     | 0     | 0 0 0 1 _    |

| Index | Depth | Dependencies |
| 0     | 0     | _ _ _ _ _    |
| 1     | 1     | 1 _ _ _ _    |
| 2     | 1     | 1 0 _ _ _    |
| 3     | 2     | 1 1 1 _ _    | < working on
| 4     | 0     | 0 0 0 1 _    |

| Index | Depth | Dependencies |
| 0     | 0     | _ _ _ _ _    |
| 1     | 1     | 1 _ _ _ _    |
| 2     | 1     | 1 0 _ _ _    |
| 3     | 2     | 1 1 1 _ _    |
| 4     | 3     | 1 1 1 1 _    | < working on

{% endhighlight %}

At each step we consider one node. For each previous node, if this node depends on a previous node, we perform a bit-wise or to include the previous node's dependencies in this nodes dependencies. We also set the nodes depth to the maximum depth of a depended-on node plus one.

Now that we've built up this table, we can extract each node and add them to a priority heap, with the priority of their depth. This way we ensure that all nodes with a depth less than $$n$$ are selected before all nodes with a depth greater than or equal to $$n$$.

This algorithm has a time complexity of $$O(n^2)$$, but we finish with a very comprehensive view of the dependencies, we get a matrix of all dependencies, both direct and indirect. And we can see if the dependency graph has two parallel paths. i.e. consider this case:

{% highlight plaintext %}

| Index | Depth | Dependencies |
| 0     | 0     | _ _ _ _ _    | < working on
| 1     | 0     | 0 _ _ _ _    |
| 2     | 0     | 1 0 _ _ _    |
| 3     | 0     | 0 0 1 _ _    |
| 4     | 0     | 0 1 0 0 _    |

| Index | Depth | Dependencies |
| 0     | 0     | _ _ _ _ _    |
| 1     | 0     | 0 _ _ _ _    | < working on
| 2     | 0     | 1 0 _ _ _    |
| 3     | 0     | 0 0 1 _ _    |
| 4     | 0     | 0 1 0 0 _    |

| Index | Depth | Dependencies |
| 0     | 0     | _ _ _ _ _    |
| 1     | 0     | 0 _ _ _ _    |
| 2     | 1     | 1 0 _ _ _    | < working on
| 3     | 0     | 0 0 1 _ _    |
| 4     | 0     | 0 1 0 0 _    |

| Index | Depth | Dependencies |
| 0     | 0     | _ _ _ _ _    |
| 1     | 0     | 0 _ _ _ _    |
| 2     | 1     | 1 0 _ _ _    |
| 3     | 2     | 1 0 1 _ _    | < working on
| 4     | 0     | 0 1 0 0 _    |

| Index | Depth | Dependencies |
| 0     | 0     | _ _ _ _ _    |
| 1     | 0     | 0 _ _ _ _    |
| 2     | 1     | 1 0 _ _ _    |
| 3     | 2     | 1 0 1 _ _    |
| 4     | 1     | 0 1 0 0 _    | < working on

{% endhighlight %}

To see if we have any parallel paths, that is to say, paths which do not depend on each other, or formally, for two paths $$A$$ and $$B$$:

$$ \forall_{ a \in A, b \in B } :\ !(d(a, b) \vee d(b, a)) $$
