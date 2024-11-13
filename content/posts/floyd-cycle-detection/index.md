---
date: '2022-09-17T15:07:25+07:00'
draft: false
title: 'Floyd’s cycle detection'
tags: ['dsa']
math: true
---
![](linked-list.png 'a linked list')
If we use two pointers `slow` and `fast`, the first one moves forward one step at a time, and the second one moves forward two steps at a time. Eventually, the two pointers will meet somewhere. In the above example, they meet at number 9 (or `G`).

`G` is where two-pointers fast and slow meet.\
`E` is the entrance of the loop.\
`S` is the first node of the linked list.\
`F` is the distance from the chain’s beginning to the loop’s entrance (from `S` to `E`). (eg: 1 – 3 – 5)\
`a` is the distance from the loop’s entrance to the point where two pointers meet (from `E` to `G`). (eg: 5 – 6 – 7 – 8 – 9)\
`C` is the length of the loop. (eg: the loop is 5 – 6 – 7 – 8 – 9 – 10, the length is 6 )\
The total distance that the `slow pointer` has moved is $F + a + xC$. $x$ is the number of times the slow pointer completes the loop.\
The total distance that the `fast pointer` has moved is $F + a + yC$. Likewise, $y$ is the number of times the fast pointer completes the loop.

Since the fast pointer moves two steps at a time, its distance will be double the slow pointer’s distance.\
\\[2*(F + a + xC) = F + a + yC\\]
\\[2F + 2a + 2xC = F + a + yC\\]
\\[F + a + 2xC = yC\\]
\\[F + a = yC – 2xC\\]
\\[F + a = (y – 2x)C\\]
Denote $n = y – 2x, n \ge 0$
\\[F + a = nC\\]
$nC$ means to complete the loop (length is `C`) `n` times

In other words, if we have two pointers `a` and `b`, `a` starts at beginning of the chain (node `S`), `b` starts at where `slow` and `fast` pointers meet (node `G`), and both two new pointers move one step at a time. The first pointer moves a distance equal to $F + a$, and the second pointer moves a distance equal to $nC$, they will meet at `G`.

\\[F + a= (n – 1)C + C\\]
\\[F + a= (n – 1)C + (C – a) + a\\]
\\[F = (n – 1)C + (C – a)\\]

This means, that if pointer a starts from the beginning of the chain (node `S`), pointer `b` starts from where two pointers slow and fast meet (node `G`). They eventually meet at node `E`, the entrance of the loop.
![](image.png 'Leetcode: 287. Find the Duplicate Number')
