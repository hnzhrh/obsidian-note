---
title: N 叉树遍历
tags:
  - fleet-note
  - algorithm/template
date: 2025-01-16
time: 19:55
aliases:
---

# DFS

```java
// N 叉树的遍历框架
void traverse(Node root) {
    if (root == null) {
        return;
    }
    // 前序位置
    for (Node child : root.children) {
        traverse(child);
    }
    // 后序位置
}
```

# BFS

第一种写法：普通

```java
void levelOrderTraverse(Node root) {
    if (root == null) {
        return;
    }
    Queue<Node> q = new LinkedList<>();
    q.offer(root);
    while (!q.isEmpty()) {
        Node cur = q.poll();
        // 访问 cur 节点
        System.out.println(cur.val);

        // 把 cur 的所有子节点加入队列
        for (Node child : cur.children) {
            q.offer(child);
        }
    }
}
```

第二种：记录节点深度

```java
void levelOrderTraverse(Node root) {
    if (root == null) {
        return;
    }
    Queue<Node> q = new LinkedList<>();
    q.offer(root);
    // 记录当前遍历到的层数（根节点视为第 1 层）
    int depth = 1;

    while (!q.isEmpty()) {
        int sz = q.size();
        for (int i = 0; i < sz; i++) {
            Node cur = q.poll();
            // 访问 cur 节点，同时知道它所在的层数
            System.out.println("depth = " + depth + ", val = " + cur.val);

            for (Node child : cur.children) {
                q.offer(child);
            }
        }
        depth++;
    }
}
```

第三种写法：带权重

```java
class State {
    Node node;
    int depth;

    public State(Node node, int depth) {
        this.node = node;
        this.depth = depth;
    }
}

void levelOrderTraverse(Node root) {
    if (root == null) {
        return;
    }
    Queue<State> q = new LinkedList<>();
    // 记录当前遍历到的层数（根节点视为第 1 层）
    q.offer(new State(root, 1));

    while (!q.isEmpty()) {
        State state = q.poll();
        Node cur = state.node;
        int depth = state.depth;
        // 访问 cur 节点，同时知道它所在的层数
        System.out.println("depth = " + depth + ", val = " + cur.val);

        for (Node child : cur.children) {
            q.offer(new State(child, depth + 1));
        }
    }
}
```
# Reference