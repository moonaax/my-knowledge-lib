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
