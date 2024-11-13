---
date: '2022-09-03T14:38:42+07:00'
draft: false
title: 'Disjoin-Set/Union-Find notes'
tags: ['dsa']
---
## 1. Definition
- *set* is a collection of unique elements.
the word *disjoin* means *non-overlapping* – two sets are considered as disjoining if they have no element in common
- *disjoin-set* contains a collection of *non-overlapping* sets
- it has two method
    - `find(x)`: return the **representative element** of the set that contains of x
        - if u and v are in the same set, find(u) = find(v)
    - `union(u, v)`: merge two sets that contain u and v
        - after merging, find(v) = find(u)
## 2. Implementations
View the visualization of the data structure here.

- tree is a connected graph with no cycles
- forest is a graph with each connected-component as a tree
![](tree-and-forest.png 'A tree and a forest')

Disjoin-Set is usually implemented as a forest.\
The below visualizations are taken from [here](https://www.cs.usfca.edu/~galles/visualization/DisjointSets.html).
### 2.0 Rank
Each node will have a corresponding rank number to represent the depth from this node.
### 2.1 Make Set
![](create-forest.png 'create a forest that has 15 trees (set)')
Initially, we create a forest that has n set, each set has just one element.\
For example, in the above image, we created a forest with 15 sets, each with just one element.\
Each node now will have the same rank: 0
### 2.2 Find
Initially, find(1) = 1, find(2) = 2,…

Because in this implementation, find will return the root of the tree representing a set, at first each set has just one element.
### 2.3 Union
Algorithm of `union(u, v)`:

1. root1 = find(u), root2 = find(v)
2. if root1.rank < root2.rank:
    - hang root1 under root2: root1.parent = root2
3. if root2.rank < root1.rank:
    - hang root2 under root1: root2.parent = root1
4. if root2.rank == root1.rank:
    - root1.parent = root2 (or root2.parent = root1)
    - root1.rank += 1 (root2.rank += 1)

**union(0,1)**
At first, ranks[0] = ranks[1] = 0 , they have same rank, so we can hang 0 under 1 or 1 under 0, both ways are fine.
![](merge-0-1.png 'merge 0 and 1')
after merging: find(0) = find(1) = 1. ranks[0] = 0, ranks[1] = 0 + 1.

**union(2,0)**
find(2) = 2, find(0) = 1. ranks[1] = 1, ranks[2] = 0. rank of 2 is smaller than rank of 1, we will hang 2 under 1.
![](merge-2-0.png 'merge 2 and 0')
After merging: find(0) = find(2) = find(1) = 1. ranks[0] = ranks[2] = 0, ranks[1] = 1.

**union(3,4)**
find(3) = 3, find(4) = 4. ranks[3] = 0, ranks[4] = 0. we can hang 3 under 4 or 4 under 3, both ways are fine.
![](merge-3-4.png 'merge 3 and 4')
after merging, find(3) = find(4) = 4. rank(3) = 0, rank(4) = 0 + 1.

**union(2,3)**
find(2) = 1, find(3) = 4. ranks[1] = 1, ranks[4] = 1. we can either hang 4 under 1 or 1 under 4.
![](union-2-3.png 'union(2,3)')
after merging, find(0) = find(2) = find(3) = 0, find(4) = 1, find(1) = 1 + 1.

This is an [implementation in go](https://github.com/khanh1998/leetcode/blob/master/DisjoinSet/implementation.go).

## 3. Minimum spanning tree
> A **minimum spanning tree** (**MST**) or **minimum weight spanning tree** is a subset of the edges of a [connected](https://en.wikipedia.org/wiki/Connected_graph), edge-weighted undirected graph that connects all the [vertices](https://en.wikipedia.org/wiki/Vertex_(graph_theory)) together, without any [cycles](https://en.wikipedia.org/wiki/Cycle_(graph_theory)) and with the minimum possible total edge weight.[\[1\]](https://en.wikipedia.org/wiki/Minimum_spanning_tree#cite_note-Numpy_and_Scipy_Documentation_%E2%80%94_Numpy_and_Scipy_documentation-1) That is, it is a [spanning tree](https://en.wikipedia.org/wiki/Spanning_tree) whose sum of edge weights is as small as possible.[\[2\]](https://en.wikipedia.org/wiki/Minimum_spanning_tree#cite_note-NetworkX_2.6.2_documentation-2) More generally, any edge-weighted undirected graph (not necessarily connected) has a **minimum spanning forest**, which is a union of the minimum spanning trees for its [connected components](https://en.wikipedia.org/wiki/Connected_component_(graph_theory)).
>
> -[**Wikipedia**](https://en.wikipedia.org/wiki/Minimum_spanning_tree)

![](mst.png 'MST')

For example, if we want to find MST of the graph on the above image, here is the algorithm:

1. `n` is the number of edges, and `v` is the number of nodes.
2. Init a Disjoin set with `v` sets corresponding with `v` nodes of the graph. In other words, if we have 10 nodes, we will have 10 sets in the Disjoin set.
    - Our target is to merge these `v` sets into just one single set representing that all nodes are connected.
3. Sort the edges in the graph in accessing order of weight. You probably can use a priority queue for this purpose, so you can always get the shortest edge first, and then the longer ones.
4. iterates through every edge in the priority queue:
    - call `e1` and `e2` are two nodes that make up the edge
    - if `find(e1) == find(e2)`, it means both node `e1` and node `e2` are connected already, so we won’t need this edge
        - skips this edge
    - if `find(e1) != find(e2)`, `e1` and `e2` now are in the different set, in other words, they are not connected.
        - `union(e1, e2)`
    - we can early stop the algorithm by checking if all `v` nodes are connected.

Here is a similar issue that can be solved using Disjoin set:
https://leetcode.com/problems/min-cost-to-connect-all-points/
https://github.com/khanh1998/Gen-5/blob/main/Homework/quoc_khanh/lesson_15/1584.MinCosttoConnectAllPoints.py

## 4. References
https://en.wikipedia.org/wiki/Disjoint-set_data_structure
https://www.cs.usfca.edu/~galles/visualization/DisjointSets.html
https://courses.cit.cornell.edu/info2950_2012sp/graphs3.pdf