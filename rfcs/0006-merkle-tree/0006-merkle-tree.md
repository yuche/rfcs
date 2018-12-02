---
Number: 0006
Category: Standards Track
Status: Draft
Author: Ke Wang
Organization: Nervos Foundation
Created: 2018-12-01
---

# Complete Binary Merkle Tree

## Introduction

CKB uses Complete Binary Merkle Tree(CBMT) to generate Merkle Root and Merkle Proof for a static list of items, such as Transactions Root. Basically, CBMT is a complete binary tree, so it has the same node order with the complete binary tree. Compare with other Merkle trees, the hash computation of CBMT is minimal, as well as the proof size.

## Tree Struct

Tree with 6 items([T0, T1, T2, T3, T4, T5]) and tree with items([T0, T1, T2, T3, T4, T5, T6]) is shown below:

```
        with 6 items                       with 7 items

              B0                                 B0
             /  \                               /  \
           /      \                           /      \
         /          \                       /          \
       /              \                   /              \
      B1              B2                 B1              B2
     /  \            /  \               /  \            /  \
    /    \          /    \             /    \          /    \
   /      \        /      \           /      \        /      \  
  B3      B4      TO      T1         B3      B4      B5      T0
 /  \    /  \                       /  \    /  \    /  \
T2  T3  T4  T5                     T1  T2  T3  T4  T5  T6
```

The tree can be stored in an array, so the two trees above can be stored as:

```
[B0, B1, B2, B3, B4, T0, T1, T2, T3, T4, T5]
[B0, B1, B2, B3, B4, B5, T0, T1, T2, T3, T4, T5, T6]
```

Suppose we have n items, the size of array would be 2n-1, the index of item i(start at 0) is i+n-1. For node at i, the index of its parent is (i-1)/2, the index of its sibling is (i+1)^1-1 and the indexes of its children are [2i+1, 2i+2].

## Merkle Proof

Merkle Proof can provide a proof for existence of one or more items. Incalculable nodes that on the path from leaves to root should be included in the proof. In the tree with 6 items above, in the proof for [T1, T4], only nodes [T5, B3] should be included.

### Proof Sturct

```c
sturct Proof {
  // size of items in the tree
  int size;
  // nodes on the path
  Hash nodes[];
}
```

### Algorithm of proof generation

```c++
Proof gen_proof(Hash tree[], int indexes[]) {
  Hash nodes[];
  Queue queue;
  
  int size = len(tree) >> 1 + 1;
  desending_sort(indexes);

  for index in indexes {
    queue.push_back(index + size - 1);
  }

  while(queue is not empty) {
    int index = queue.pop_front();
    int subling = calculate_subling(index);

    if(subling == queue.front()) {
      queue.pop_front();
    } else {
      nodes.push_back(tree[subling]);
    }

    int parent = calculate_parent(index);
    if(parent != 0) {
      queue.push_back(parent);
    }
  }

  return Proof::new(size, nodes);
}
```

### Algorithm of validation

```c++
bool validate_proof(Proof proof, Hash root, Item items[]) {
  Queue queue;
  desending_sort_by_item_index(items);
  
  for item in items {
    queue.push_back((item.hash(), item.index() + Proof.size - 1));
  }
  
  int i = 0;
  while(queue is not empty) {
    Hash hash, hash1, hash2;
    int index1, index2;

    (hash1, index1) = queue.pop_front();
    (hash2, index2) = queue.front();
    int subling = calculate_subling(index1);

    if(subling == index2) {
      queue.pop_front();
      hash = merge(hash2, hash1);
    } else {
      hash2 = proof.nodes[i++];

      if(is_left_node(index1)) {
        hash = merge(hash1, hash2);
      } else {
        hash = merge(hash2, hash1);
      }
    }

    int parent = calculate_parent(index);
    if(parent == 0) {
      return root == hash;
    }
    queue.push_back((hash, parent))
  }
}
```