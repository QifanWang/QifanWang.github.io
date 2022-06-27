---
layout: post
title:  "二叉树的序列化与反序列化"
categories: algorithm
tags: [leetcode]
toc: true
--- 
We awaken in others the same attitude of mind we hold toword them.
{: .message }

从一道题目看二叉树的序列化与反序列的几种方式。

## Issue

Leetcode上的[一道题目](https://leetcode.cn/problems/serialize-and-deserialize-binary-tree/)要求将一个二叉树序列化成自定义的格式，然后再还原为原来的二叉树。借这道题小小讨论一下不同的二叉树的序列化与反序列化的几种方式。

## Binary Searching Tree

如果二叉树是一个二叉搜索树，那么根据前序遍历的序列是很容易还原二叉树的（后序遍历同理）：确定根节点后根据大小关系将序列分为左右子树，分而治之。但是中序遍历却不行，因为无法确定根节点。

## Complete/Full Binary Tree

如果二叉树拥有全或满的性质，情况为之改变。全二叉树(Complete Binary Tree)为除了最后一层的节点，每层节点均完全填满，且最后一层节点尽可能向左排布(也有可能填满)。由于每层的数量关系已知，因此采取层次遍历的方式序列化比较容易存储与还原。满二叉树(Full Binary Tree)的节点要么有左右儿子，要么为叶子节点。采用前序遍历并为每个节点分配一位指示是否为叶子节点，也可以通过递归的写法进行序列化和还原。

## General Binary Tree

一个普通的二叉树没有太多可以利用的性质，最容易想到的方法是存储其前序与中序遍历序列(后序与中序也可，但前后序不可)。这种方法的关键是利用前序与中序确定根节点，然后切分左右子树，分而治之。思想类似BST解法，但序列化数据的存储空间相当于原节点数乘二。

另一种常见的方法是为空节点分配一种标记，前序遍历的过程中也添加空标记，以此完成序列化；而反序列化则通过递归写法完成。空标记的作用是在递归过程中指示基线条件，这和满二叉树解法相似，只不过这里除了叶子节点和满节点，还有只有左儿子和只有右儿子的节点。当然我们也可在序列化时为每个节点多分配两位，两位刚好表示四种情况，但处理四种情况的逻辑会使代码比较复杂。这里提供前序遍历+空标记写法。

```python
# Definition for a binary tree node.
# class TreeNode(object):
#     def __init__(self, x):
#         self.val = x
#         self.left = None
#         self.right = None

class Codec:

    def serialize(self, root):
        """Encodes a tree to a single string.
        
        :type root: TreeNode
        :rtype: str
        """
        if root is None: return ""

        ret = ""

        def preorder(cur):
            nonlocal ret
            if cur:
                ret += ',' + str(cur.val)
                preorder(cur.left)
                preorder(cur.right)
            else:
                ret += ',#'
        
        ret += str(root.val)
        preorder(root.left)
        preorder(root.right)
        return ret
        

    def deserialize(self, data):
        """Decodes your encoded data to tree.
        
        :type data: str
        :rtype: TreeNode
        """
        if data == '': return None

        eles = data.split(',')
        i, N = 0, len(eles)

        def preorder() -> TreeNode:
            nonlocal i
            if eles[i] == '#':
                i += 1
                return None
            else:
                ret = TreeNode(int(eles[i]))
                i += 1
                ret.left = preorder()
                ret.right = preorder()
                return ret

        return preorder()

# Your Codec object will be instantiated and called as such:
# ser = Codec()
# deser = Codec()
# ans = deser.deserialize(ser.serialize(root))
```

分析一下空间，设二叉树的节点数目为N，其中满节点数目为N2，拥有一个儿子的节点数目为N1，叶子节点数目为N0。那么有，
> N = N0 + N1 + N2

根据边数关系，
> N0 + N1 + N2 - 1 = N1 + 2 * N2

即
> N0 - 1 = N2

空标记将有 2 * N0 + N1 个，即 N0 + N1 + N2 + 1 个，其实也相当于占用一个新的二叉树大小。只不过空标记可能数据占用更小而已。

还有一种解法可以结合空标记与指示位，如下前序遍历序列，符号'表示节点为非叶子节点，符号/表示空标记。无论是空标记还是指示位，抑或是二者结合，本质上只是将序列化递归时的基线条件路径通过序列“记住”，然后反序列化重走一遍递归时可知基线条件的路径而已。

>A'B'DE'G/C'/F

在使用空标记后，也可使用层次遍历的方法序列化与反序列化，如下的写法没有用到递归，而是反序列化时通过双指针不断添加子节点。这个写法参考了外站的一个[讨论](https://leetcode.com/problems/recover-binary-search-tree/discuss/32539/Tree-Deserializer-and-Visualizer-for-Python)。这个写法似乎也是LC官方的二叉树可视化工具的写法，只是LC官方去掉了序列结尾的一串空标记，待考证。

```python
# Definition for a binary tree node.
# class TreeNode(object):
#     def __init__(self, x):
#         self.val = x
#         self.left = None
#         self.right = None

class Codec:

    def serialize(self, root):
        """Encodes a tree to a single string.
        
        :type root: TreeNode
        :rtype: str
        """
        if root is None: return ""

        ret = ""
        q = deque()
        q.append(root)
        while q:
            cur = q.popleft()
            if cur == root:
                ret += str(cur.val)
            else:
                ret += ',' + str(cur.val) if cur else ',#'
            if cur:
                q.append(cur.left)
                q.append(cur.right)
        return ret
        

    def deserialize(self, data):
        """Decodes your encoded data to tree.
        
        :type data: str
        :rtype: TreeNode
        """
        if data == '': return None

        nodes = [None if d == '#' else TreeNode(int(d)) for d in data.split(',')]
        # print(data, nodes)
        j, N = 1, len(nodes)
        for i in range(N):
            if nodes[i]:
                if j < N:
                    nodes[i].left = nodes[j]
                    j += 1
                if j < N:
                    nodes[i].right = nodes[j]
                    j += 1
        return nodes[0]
        
# Your Codec object will be instantiated and called as such:
# ser = Codec()
# deser = Codec()
# ans = deser.deserialize(ser.serialize(root))
```

泛化一下，如果是多叉树的序列化与反序列化，我们可以用某个符号表示前序遍历中子节点列表的的结尾。背后的性质在于多叉树每个节点的子节点是尽可能向左排布的，用该符号可以记录序列化递归过程中每个调用的return（不管是基线条件路径还是递归条件路径），如下前序遍历序列用符号)表示子节点结尾

> ABE)FK)))C)DG)H)I)J)))

## Conclusion
从上面的讨论过程中可以看出，从BST以及一些特殊的二叉树出发，其特殊的性质很容易使人得出序列化与反序列化方法。再思考通用的二叉树，几种写法背后也有特殊二叉树解法的思想。也许思考一个复杂的问题可以从某些特殊情况出发，从特殊到一般，寻找相通点，搭建通往通用解法的桥梁。另外，递归写法中通过特殊符号记录递归过程的思想很值得学习，从动态规划的过程中记录决策路径等其他问题也可以应用这类思想。非递归写法的双指针其实也是记录层次遍历时的队列变化过程。

## Reference
1. [LC 297](https://leetcode.cn/problems/serialize-and-deserialize-binary-tree/)
2. [Tree Deserializer and Visualizer](https://leetcode.com/problems/recover-binary-search-tree/discuss/32539/Tree-Deserializer-and-Visualizer-for-Python)
3. [Geeks for geeks: serialize deserialize binary tree](https://www.geeksforgeeks.org/serialize-deserialize-binary-tree/)