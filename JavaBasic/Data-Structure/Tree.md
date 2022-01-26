# Tree
## Binary Tree

## Trie Tree
The definition of binary-tree's node is
```java
class TreeNode {
    int val;
    TreeNode left, right;
}
```
left and right storage point of child node：

The definition of muti-way tree's node is
```java
class TreeNode {
    int val;
    TreeNode[] children;
}
```

The definition of trie-tree's node is
```java
class TrieNode<V> {
    V val;
    TrieNode<V>[] children = new TrieNode[256];
}
```
