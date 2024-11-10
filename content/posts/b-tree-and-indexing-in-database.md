---
date: '2022-01-09T09:02:34+07:00'
draft: false
title: 'B tree, B+ tree, and indexing in database'
---
## B tree and B+ tree
B tree is a self-balance tree. I can see that the B tree and AVL tree have a thing in common – it is all self-balance. But the difference is, each node in the AVL tree store exactly one value, and have at most two children. Each node of the B tree contains an array of at most **N** values and has at most **N + 1** children.

I am not gonna go into details about the [B tree](https://www.cs.usfca.edu/~galles/visualization/BTree.html) or [B+ tree](https://www.cs.usfca.edu/~galles/visualization/BPlusTree.html), you can read it on Wikipedia.

Here are two very good demos on B tree and B+ tree, you can try them to understand how those trees work.

## How does data get stored on a disk?
The basic unit of a disk is a block. The size of a block is usually 512MB, which means that when you read, you have to read a whole block of 512MB. If you write, you have to write to a 512MB block.

512MB is really large, so the database usually stores multiple records in the same block.

## Index
Database stores data in disk, so how do they find a specific record? Well, they have to scan through every record in the table to find your expected record, and IO operations are expensive. But that is when the database doesn’t have an index.

In simple words, an index is a way to map from a record identifier to its location on a disk. When using create a table, it usually comes with an index on the ID field, so that index is the mapping from ID to its record location on disk. Because the database stores multiple rows in a disk block, so, the location here is not exactly the location of the expected record, but the location of the block that contains the record. The database will read the whole block and return to you your desired record.

## So, where does the indexes get stored?
Well, on disk. Basically, indexes are additional data structures, it requires more space on the disk to store them.

So when a database wants to use an index, it has to read indexes from the disk.

## Is it slow?
Yes, at first. The only needed block will be loaded to RAM, not all blocks of the index. But when a block gets accessed frequently, it will be cached, which could increase the performance.

I have thought why wouldn’t load the whole index to RAM so we can access it quickly. Because the Index could be really big, and the idea of bringing the whole index to RAM is not possible.

## Why B tree is a good fit for the database index?
Why don’t use something like an AVL tree?

As I mentioned before, where we store data to disk, the basic unit is 512MB, so we want to put data to block as much as possible.

So each block now will store a list of mapping from Identifier to record block location.

## Why B+ tree even better than Btree?
The problem is IO operations are expensive and slow, so we want to avoid them, and the B+ tree helps us do that.

B+ tree only stores key (or identifier) in a non-leaf node, which means one block now can store more mapping than before. B tree stores a list of pairs of a key and a location on a non-leaf node, it stores fewer mapping items per disk block.

Let say, when using the B tree, we can store 100 mapping per disk block, but the B+ tree could possibly store 150 mapping per disk block.

What does it mean? Each node in the B tree can have at most 101 children, but the B+ tree has at most 151 children. Each node in the B+ tree can store more information make the tree shorter when compared with the B tree.

In other words, in my example, read a block in B tree has time complexity O(log(n/100)), while the B+ tree has smaller time complexity O(log(n/150)). Where n = number of rows.

## Disadvantages of B+ tree
When you use the B+ tree, even you found the key that you want, you still have to travel to the leaf node to take the value – location of the disk block contains the desired record.

## References
[https://stackoverflow.com/a/870324](https://stackoverflow.com/a/870324)

[https://stackoverflow.com/a/1130](https://stackoverflow.com/a/1130)

