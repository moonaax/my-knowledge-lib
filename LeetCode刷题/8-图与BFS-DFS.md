# 图与 BFS/DFS

## 核心技巧

- BFS：层序遍历、最短路径（无权图）
- DFS：连通分量、拓扑排序、环检测
- 建图：邻接表 `List<List<Integer>>`
- visited 数组防止重复访问

---

## 1. 岛屿数量（#200）

> 计算二维网格中岛屿的数量

```java
public int numIslands(char[][] grid) {
    int count = 0;
    for (int i = 0; i < grid.length; i++) {
        for (int j = 0; j < grid[0].length; j++) {
            if (grid[i][j] == '1') {
                dfs(grid, i, j);
                count++;
            }
        }
    }
    return count;
}

private void dfs(char[][] grid, int i, int j) {
    if (i < 0 || i >= grid.length || j < 0 || j >= grid[0].length || grid[i][j] != '1') return;
    grid[i][j] = '0'; // 标记已访问
    dfs(grid, i + 1, j);
    dfs(grid, i - 1, j);
    dfs(grid, i, j + 1);
    dfs(grid, i, j - 1);
}
```

- 时间 O(m×n)，空间 O(m×n) 递归栈
- 原地修改代替 visited

**📝 解析**

- **思路推导**：遍历网格，遇到 '1' 就 DFS 把整个连通的岛屿标记为 '0'，计数+1。
- **关键点**：原地修改 `grid[i][j]='0'` 代替 visited 数组，节省空间。也可以用 BFS。
- **易错点**：递归深度可能很大（最坏 m×n），可能栈溢出。BFS 更安全。
- **举一反三**：岛屿的最大面积（#695）→ DFS 时计数面积；岛屿周长（#463）→ 计算边界

---

## 2. 腐烂的橘子（#994）

> BFS 求所有橘子腐烂的最短时间

```java
public int orangesRotting(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    Queue<int[]> queue = new LinkedList<>();
    int fresh = 0;
    for (int i = 0; i < m; i++)
        for (int j = 0; j < n; j++) {
            if (grid[i][j] == 2) queue.offer(new int[]{i, j});
            else if (grid[i][j] == 1) fresh++;
        }
    if (fresh == 0) return 0;
    int[][] dirs = {{1,0},{-1,0},{0,1},{0,-1}};
    int minutes = 0;
    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int k = 0; k < size; k++) {
            int[] pos = queue.poll();
            for (int[] d : dirs) {
                int ni = pos[0] + d[0], nj = pos[1] + d[1];
                if (ni >= 0 && ni < m && nj >= 0 && nj < n && grid[ni][nj] == 1) {
                    grid[ni][nj] = 2;
                    fresh--;
                    queue.offer(new int[]{ni, nj});
                }
            }
        }
        minutes++;
    }
    return fresh == 0 ? minutes - 1 : -1;
}
```

- 多源 BFS：所有腐烂橘子同时入队
- 时间 O(m×n)，空间 O(m×n)

**📝 解析**

- **思路推导**：多源 BFS：所有初始腐烂橘子同时入队作为第 0 层，层序扩展。层数就是时间。
- **关键点**：最后检查 `fresh == 0`，如果还有新鲜橘子说明无法全部腐烂，返回 -1。`minutes - 1` 因为最后一层没有新橘子但 minutes 多加了 1。
- **易错点**：初始 fresh=0 时直接返回 0（没有新鲜橘子）。
- **举一反三**：01 矩阵（#542）→ 多源 BFS 求每个 1 到最近 0 的距离

---

## 3. 课程表（#207）

> 判断能否完成所有课程（有向图是否有环）

**思路**：拓扑排序（BFS / Kahn 算法）

```java
public boolean canFinish(int numCourses, int[][] prerequisites) {
    int[] inDegree = new int[numCourses];
    List<List<Integer>> graph = new ArrayList<>();
    for (int i = 0; i < numCourses; i++) graph.add(new ArrayList<>());
    for (int[] p : prerequisites) {
        graph.get(p[1]).add(p[0]);
        inDegree[p[0]]++;
    }
    Queue<Integer> queue = new LinkedList<>();
    for (int i = 0; i < numCourses; i++)
        if (inDegree[i] == 0) queue.offer(i);
    int count = 0;
    while (!queue.isEmpty()) {
        int course = queue.poll();
        count++;
        for (int next : graph.get(course)) {
            if (--inDegree[next] == 0) queue.offer(next);
        }
    }
    return count == numCourses;
}
```

- 时间 O(V+E)，空间 O(V+E)
- count != numCourses 说明有环

**📝 解析**

- **思路推导**：课程依赖 = 有向图，能否完成 = 有向图是否无环 = 拓扑排序是否能处理所有节点。
- **关键点**：Kahn 算法：入度为 0 的节点入队，处理后将邻居入度 -1，新的入度为 0 的入队。最终处理的节点数 = 总数则无环。
- **易错点**：也可以用 DFS 染色法检测环（白/灰/黑三色标记），但 BFS 拓扑排序更直观。
- **举一反三**：课程表 II（#210）→ 返回拓扑排序结果；课程表 III（#630）→ 贪心+堆

---

## 4. 课程表 II（#210）

> 返回拓扑排序的结果

```java
public int[] findOrder(int numCourses, int[][] prerequisites) {
    int[] inDegree = new int[numCourses];
    List<List<Integer>> graph = new ArrayList<>();
    for (int i = 0; i < numCourses; i++) graph.add(new ArrayList<>());
    for (int[] p : prerequisites) {
        graph.get(p[1]).add(p[0]);
        inDegree[p[0]]++;
    }
    Queue<Integer> queue = new LinkedList<>();
    for (int i = 0; i < numCourses; i++)
        if (inDegree[i] == 0) queue.offer(i);
    int[] order = new int[numCourses];
    int idx = 0;
    while (!queue.isEmpty()) {
        int course = queue.poll();
        order[idx++] = course;
        for (int next : graph.get(course))
            if (--inDegree[next] == 0) queue.offer(next);
    }
    return idx == numCourses ? order : new int[0];
}
```

**📝 解析**

- **思路推导**：和课程表 I 完全一样的拓扑排序，只是多了一个 order 数组记录顺序。
- **关键点**：`idx == numCourses` 判断是否有环，有环返回空数组。
- **易错点**：拓扑排序结果不唯一，任何合法顺序都行。
- **举一反三**：火星词典（#269）→ 从单词顺序推导字母顺序，建图后拓扑排序

---

## 5. 被围绕的区域（#130）

> 把被 X 围绕的 O 翻转，边界连通的 O 不翻转

**思路**：从边界 O 开始 DFS 标记，剩下的 O 就是被围绕的

```java
public void solve(char[][] board) {
    int m = board.length, n = board[0].length;
    for (int i = 0; i < m; i++) {
        dfs(board, i, 0);
        dfs(board, i, n - 1);
    }
    for (int j = 0; j < n; j++) {
        dfs(board, 0, j);
        dfs(board, m - 1, j);
    }
    for (int i = 0; i < m; i++)
        for (int j = 0; j < n; j++) {
            if (board[i][j] == 'O') board[i][j] = 'X';
            else if (board[i][j] == '#') board[i][j] = 'O';
        }
}

private void dfs(char[][] board, int i, int j) {
    if (i < 0 || i >= board.length || j < 0 || j >= board[0].length || board[i][j] != 'O') return;
    board[i][j] = '#';
    dfs(board, i + 1, j);
    dfs(board, i - 1, j);
    dfs(board, i, j + 1);
    dfs(board, i, j - 1);
}
```

- 逆向思维：先保护边界连通的，再翻转剩余的

**📝 解析**

- **思路推导**：直接找被围绕的 O 很难判断。逆向思维：从边界的 O 出发 DFS 标记为 '#'（安全的），剩下的 O 就是被围绕的，翻转为 X，最后 '#' 恢复为 O。
- **关键点**：只需要从四条边界出发 DFS，不需要遍历所有格子。
- **易错点**：三步操作的顺序不能乱：先标记边界连通 → 翻转内部 O → 恢复 '#'。
- **举一反三**：飞地的数量（#1020）→ 类似思路，计数而非翻转

---

## 6. 单词接龙（#127）

> 从 beginWord 到 endWord 的最短转换序列长度

**思路**：BFS，每次变一个字母

```java
public int ladderLength(String beginWord, String endWord, List<String> wordList) {
    Set<String> wordSet = new HashSet<>(wordList);
    if (!wordSet.contains(endWord)) return 0;
    Queue<String> queue = new LinkedList<>();
    queue.offer(beginWord);
    int level = 1;
    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            char[] word = queue.poll().toCharArray();
            for (int j = 0; j < word.length; j++) {
                char orig = word[j];
                for (char c = 'a'; c <= 'z'; c++) {
                    if (c == orig) continue;
                    word[j] = c;
                    String next = new String(word);
                    if (next.equals(endWord)) return level + 1;
                    if (wordSet.remove(next)) queue.offer(next);
                }
                word[j] = orig;
            }
        }
        level++;
    }
    return 0;
}
```

- 时间 O(n × L × 26)，n 为单词数，L 为单词长度
- 用 `wordSet.remove` 代替 visited，更简洁

**📝 解析**

- **思路推导**：每个单词是图的节点，只差一个字母的单词之间有边。求最短路径 → BFS。对每个单词的每个位置尝试 26 个字母替换。
- **关键点**：`wordSet.remove(next)` 同时起到 visited 的作用，避免重复访问。比 HashSet visited 更简洁。
- **易错点**：双向 BFS 可以大幅优化：从 beginWord 和 endWord 同时搜索，每次扩展较小的那一端。
- **举一反三**：单词接龙 II（#126）→ 返回所有最短路径，需要记录父节点

---

## 7. 克隆图（#133）

```java
private Map<Node, Node> visited = new HashMap<>();

public Node cloneGraph(Node node) {
    if (node == null) return null;
    if (visited.containsKey(node)) return visited.get(node);
    Node clone = new Node(node.val);
    visited.put(node, clone);
    for (Node neighbor : node.neighbors)
        clone.neighbors.add(cloneGraph(neighbor));
    return clone;
}
```

- DFS + HashMap 记录已克隆节点
- 时间 O(V+E)，空间 O(V)

**📝 解析**

- **思路推导**：DFS 遍历图，用 HashMap 存原节点→克隆节点的映射。遇到已克隆的直接返回，避免死循环。
- **关键点**：先创建克隆节点放入 map，再递归克隆邻居。顺序不能反，否则环形图会死循环。
- **易错点**：node 为 null 时返回 null。BFS 也能做，用队列逐层克隆。
- **举一反三**：复制带随机指针的链表（#138）→ 同样的克隆+映射思路
