---
Number: 0006
Category: Standards Track
Status: Draft
Author: Ke Wang
Organization: Nervos Foundation
Created: 2018-12-01
---

# Complete Binary Merkle Tree

## 概述

CKB 使用 Complete Binary Merkle Tree 来为静态数据生成 Merkle Root 及 Merkle Proof，如计算Transactions Root。它是一棵完全二叉树，节点排列方式与完全二叉树相同，相比于其它的 Merkle Tree，Complete Binary Merkle Tree具有最少的Hash计算量及最小的proof size。

## 数据结构

6个item(item的Hash为[T0, T1, T2, T3, T4, T5])与7个item(item的hash为[T0, T1, T2, T3, T4, T5, T6])生成的Tree的结构如下所示：

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

构建的Tree可以存在一个数组中，上面的两棵tree用数组表示分别为：

```
[B0, B1, B2, B3, B4, T0, T1, T2, T3, T4, T5]
[B0, B1, B2, B3, B4, B5, T0, T1, T2, T3, T4, T5, T6]
```

假设有n个item，则数组的大小为2n-1，item i 在数组中的下标为（下标从0开始）i+n-1。对于下标为i的节点，其父节点下标为(i-1)/2，兄弟节点下标为(i+1)^1-1，子节点的下标为2i+1、2i+2。

此外，我们规定对于只有0个item的情况，生成的tree只有0个node，其root为H256::zero。

## Merkle Proof

Merkle Proof 能为一个或多个item提供存在性证明，Proof中应只包含从叶子节点到根节点路径中无法直接计算的节点，如在6个item的Merkle Tree中为[T1, T4]生成的Proof中应只包含[T5, T0, B3]。

### Proof 结构

Proof 结构体的schema形式为：

```
table Proof {
  // size of items in the tree
  size: uint32;
  // nodes on the path which can not be calculated, in descending order by index
  nodes: [H256];
}
```

### Proof 生成算法

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
    int sibling = calculate_sibling(index);

    if(sibling == queue.front()) {
      queue.pop_front();
    } else {
      nodes.push_back(tree[sibling]);
    }

    int parent = calculate_parent(index);
    if(parent != 0) {
      queue.push_back(parent);
    }
  }

  return Proof::new(size, nodes);
}
```

### Proof 校验算法

```c++
bool validate_proof(Proof proof, Hash root, Item items[]) {
  if(proof.size = 0) {
    return root == H256::zero;
  }

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
    int sibling = calculate_sibling(index1);

    if(sibling == index2) {
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

  return false;
}
```
