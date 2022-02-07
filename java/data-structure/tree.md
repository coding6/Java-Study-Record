# Tree
## Binary Tree
### OverView

### Traversal method
* Preorder Traversal

```java
//递归
public List<Integer> preorderTraversal(TreeNode root) {
    List<Integer> result = new ArrayList();
    dfs(root, result);
    return result;
}
public void dfs(TreeNode node, List<Integer> list) {
    if (node == null) {
        return;
    }
    list.add(node.val);
    dfs(node.left, list);
    dfs(node.right, list);
}
//递归不使用辅助函数
List<Integer> res = new ArrayList();
public List<Integer> preorderTraversal(TreeNode root) {
    if (root == null) {
        return res;
    }
    res.add(root.val);
    res.addAll(preorderTraversal(root.left));
    res.addAll(preorderTraversal(root.right));  
}
//非递归
public List<Integer> preorderTraversal(TreeNode root) {
    List<Integer> res = new ArrayList();
    LinkedList<TreeNode> stack = new LinkedList();
    TreeNode node = root;
    while (stack.size() != 0 || node != null) {
        if (node != null) {
            res.add(node.val);
            stack.add(node);
            node = node.left;
        } else {
            node = stack.removeLast();
            node = node.right;
        }
    }
    return res;
}
```
* InOrder Traversal

```java
//递归
public List<Integer> inorderTraversal(TreeNode root) {
    List<Integer> result = new ArrayList();
    dfs(root, result);
    return result;
}
public void dfs(TreeNode node, List<Integer> list) {
    if (node == null) {
        return;
    }
    dfs(node.left, list);
    list.add(node.val);
    dfs(node.right, list);
}
//非递归
public List<Integer> inorderTraversal(TreeNode root) {
    List<Integer> res = new ArrayList();
    LinkedList<TreeNode> stack = new LinkedList();
    TreeNode node = root;
    while (stack.size() != 0 || node != null) {
        if (node != null) {
            stack.add(node);
            node = node.left;
        } else {
            node = stack.removeLast();
            res.add(node.val);
            node = node.right;
        }
    }
    return res;
}
```
* PostOrder Traversal

```java
//递归
public List<Integer> postorderTraversal(TreeNode root) {
    List<Integer> result = new ArrayList();
    dfs(root, result);
    return result;
}
public void dfs(TreeNode node, List<Integer> list) {
    if (node == null) {
        return;
    }
    dfs(node.left, list);
    list.add(node.val);
    dfs(node.right, list);
}
//非递归
public List<Integer> postorderTraversal(TreeNode root) {
    //后续遍历的非递归稍微复杂一些，前序遍历是中左右，如果前序遍历倒过来就是右左中，这时我们只需要把右左变为左右即可
    LinkedList<Integer> res = new LinkedList();
    LinkedList<TreeNode> stack = new LinkedList();
    TreeNode node = root;
    while (stack.size() != 0 || node != null) {
        if (node != null) {
            stack.add(node);
            res.addFirst(node.val);
            node = node.right;
        } else {
            node = stack.removeLast();
            node = node.left;
        }
    }
    return res;
}
```
递归思想在二叉树中运用的淋漓尽致，不论是什么题目，都是在前中后序遍历的不同节点做一些骚操作，
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
