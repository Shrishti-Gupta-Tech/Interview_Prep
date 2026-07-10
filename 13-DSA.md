# 13. Data Structures & Algorithms — Deep Dive

> **Goal:** Master the patterns behind common problems, know when to apply each technique, and code them fluently in Java.

---

## 13.0 How to Approach DSA in Interviews

### The 5-step framework

1. **Understand** — read carefully. Ask clarifying questions (edge cases, input size, duplicates?).
2. **Examples** — walk through 2-3 examples by hand.
3. **Approach** — brainstorm: brute force first, then optimize.
4. **Complexity** — analyze time & space.
5. **Code** — write clean code, then test.

### Complexity cheat sheet

| Big-O | Name | Example |
|-------|------|---------|
| O(1) | Constant | Array access, HashMap get |
| O(log n) | Logarithmic | Binary search |
| O(n) | Linear | Loop through array |
| O(n log n) | Linearithmic | Merge sort, heap sort |
| O(n²) | Quadratic | Nested loops, bubble sort |
| O(2ⁿ) | Exponential | Recursive Fibonacci |
| O(n!) | Factorial | Permutations |

Aim for the best complexity possible. `n = 10^5` → O(n log n) fine; `n = 10^7` → need O(n) or better.

---

## 13.1 Arrays

### Pattern: Two Pointers

**When:** sorted array; find pair/triplet; palindrome.

**Two Sum (unsorted, HashMap approach) — O(n)**
```java
public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> seen = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        int need = target - nums[i];
        if (seen.containsKey(need)) return new int[]{ seen.get(need), i };
        seen.put(nums[i], i);
    }
    return new int[0];
}
```

**Two Sum (sorted, two pointers) — O(n) space O(1)**
```java
public int[] twoSumSorted(int[] a, int target) {
    int l = 0, r = a.length - 1;
    while (l < r) {
        int sum = a[l] + a[r];
        if (sum == target) return new int[]{l, r};
        if (sum < target) l++; else r--;
    }
    return new int[0];
}
```

**Three Sum — sum to zero — O(n²)**
```java
public List<List<Integer>> threeSum(int[] nums) {
    Arrays.sort(nums);
    List<List<Integer>> res = new ArrayList<>();
    for (int i = 0; i < nums.length - 2; i++) {
        if (i > 0 && nums[i] == nums[i - 1]) continue;   // skip duplicates
        int l = i + 1, r = nums.length - 1;
        while (l < r) {
            int sum = nums[i] + nums[l] + nums[r];
            if (sum == 0) {
                res.add(List.of(nums[i], nums[l], nums[r]));
                while (l < r && nums[l] == nums[l+1]) l++;
                while (l < r && nums[r] == nums[r-1]) r--;
                l++; r--;
            } else if (sum < 0) l++;
            else r--;
        }
    }
    return res;
}
```

### Pattern: Prefix Sum

**When:** range sum queries, subarray sum problems.

```java
int[] prefix = new int[nums.length + 1];
for (int i = 0; i < nums.length; i++)
    prefix[i + 1] = prefix[i] + nums[i];

int rangeSum(int l, int r) { return prefix[r + 1] - prefix[l]; }
```

**Subarray Sum equals K — O(n) using prefix + HashMap**
```java
public int subarraySum(int[] nums, int k) {
    Map<Integer, Integer> cnt = new HashMap<>();
    cnt.put(0, 1);
    int sum = 0, res = 0;
    for (int n : nums) {
        sum += n;
        res += cnt.getOrDefault(sum - k, 0);
        cnt.merge(sum, 1, Integer::sum);
    }
    return res;
}
```

### Kadane's Algorithm — Maximum Subarray Sum

**When:** find max sum of contiguous subarray. **O(n)**.

```java
public int maxSubArray(int[] nums) {
    int max = nums[0], cur = nums[0];
    for (int i = 1; i < nums.length; i++) {
        cur = Math.max(nums[i], cur + nums[i]);   // extend or restart
        max = Math.max(max, cur);
    }
    return max;
}
```

### Product of Array Except Self — O(n) no division

```java
public int[] productExceptSelf(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];
    result[0] = 1;
    for (int i = 1; i < n; i++) result[i] = result[i-1] * nums[i-1];   // left product
    int right = 1;
    for (int i = n - 1; i >= 0; i--) {
        result[i] *= right;
        right *= nums[i];
    }
    return result;
}
```

### Rotate Array

```java
public void rotate(int[] nums, int k) {
    k %= nums.length;
    reverse(nums, 0, nums.length - 1);
    reverse(nums, 0, k - 1);
    reverse(nums, k, nums.length - 1);
}
private void reverse(int[] a, int l, int r) {
    while (l < r) { int t = a[l]; a[l++] = a[r]; a[r--] = t; }
}
```

---

## 13.2 Strings

### Longest Substring Without Repeating Characters — Sliding Window — O(n)

```java
public int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> lastSeen = new HashMap<>();
    int start = 0, max = 0;
    for (int i = 0; i < s.length(); i++) {
        char c = s.charAt(i);
        if (lastSeen.containsKey(c) && lastSeen.get(c) >= start)
            start = lastSeen.get(c) + 1;
        lastSeen.put(c, i);
        max = Math.max(max, i - start + 1);
    }
    return max;
}
```

### Valid Parentheses — Stack — O(n)

```java
public boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    for (char c : s.toCharArray()) {
        if (c == '(') stack.push(')');
        else if (c == '[') stack.push(']');
        else if (c == '{') stack.push('}');
        else if (stack.isEmpty() || stack.pop() != c) return false;
    }
    return stack.isEmpty();
}
```

### Reverse String / Palindrome check

```java
public void reverseString(char[] s) {
    int l = 0, r = s.length - 1;
    while (l < r) { char t = s[l]; s[l++] = s[r]; s[r--] = t; }
}

public boolean isPalindrome(String s) {
    int l = 0, r = s.length() - 1;
    while (l < r) {
        while (l < r && !Character.isLetterOrDigit(s.charAt(l))) l++;
        while (l < r && !Character.isLetterOrDigit(s.charAt(r))) r--;
        if (Character.toLowerCase(s.charAt(l++)) != Character.toLowerCase(s.charAt(r--))) return false;
    }
    return true;
}
```

### Anagram check — count array approach — O(n)

```java
public boolean isAnagram(String s, String t) {
    if (s.length() != t.length()) return false;
    int[] cnt = new int[26];
    for (int i = 0; i < s.length(); i++) {
        cnt[s.charAt(i) - 'a']++;
        cnt[t.charAt(i) - 'a']--;
    }
    for (int c : cnt) if (c != 0) return false;
    return true;
}
```

### Group Anagrams

```java
public List<List<String>> groupAnagrams(String[] strs) {
    Map<String, List<String>> map = new HashMap<>();
    for (String s : strs) {
        char[] arr = s.toCharArray();
        Arrays.sort(arr);
        String key = new String(arr);
        map.computeIfAbsent(key, k -> new ArrayList<>()).add(s);
    }
    return new ArrayList<>(map.values());
}
```

---

## 13.3 Sliding Window

### Template
```java
int l = 0;
for (int r = 0; r < n; r++) {
    // include a[r]

    while (windowConditionBroken()) {
        // shrink from left
        l++;
    }

    // update answer with current window [l, r]
}
```

### Max sum of subarray of size K — Fixed window

```java
public int maxSum(int[] a, int k) {
    int sum = 0;
    for (int i = 0; i < k; i++) sum += a[i];
    int max = sum;
    for (int i = k; i < a.length; i++) {
        sum += a[i] - a[i - k];
        max = Math.max(max, sum);
    }
    return max;
}
```

### Min length subarray with sum ≥ target — Dynamic window

```java
public int minSubArrayLen(int target, int[] nums) {
    int l = 0, sum = 0, min = Integer.MAX_VALUE;
    for (int r = 0; r < nums.length; r++) {
        sum += nums[r];
        while (sum >= target) {
            min = Math.min(min, r - l + 1);
            sum -= nums[l++];
        }
    }
    return min == Integer.MAX_VALUE ? 0 : min;
}
```

### Longest substring with at most K distinct chars

```java
public int lengthOfLongestSubstringKDistinct(String s, int k) {
    Map<Character, Integer> count = new HashMap<>();
    int l = 0, max = 0;
    for (int r = 0; r < s.length(); r++) {
        count.merge(s.charAt(r), 1, Integer::sum);
        while (count.size() > k) {
            char c = s.charAt(l++);
            if (count.merge(c, -1, Integer::sum) == 0) count.remove(c);
        }
        max = Math.max(max, r - l + 1);
    }
    return max;
}
```

---

## 13.4 Binary Search

### Template — closed interval

```java
public int binarySearch(int[] a, int target) {
    int l = 0, r = a.length - 1;
    while (l <= r) {
        int m = l + (r - l) / 2;      // avoid overflow!
        if (a[m] == target) return m;
        if (a[m] < target) l = m + 1;
        else r = m - 1;
    }
    return -1;
}
```

### First / Last occurrence

```java
int firstOccurrence(int[] a, int target) {
    int l = 0, r = a.length - 1, res = -1;
    while (l <= r) {
        int m = l + (r - l) / 2;
        if (a[m] == target) { res = m; r = m - 1; }
        else if (a[m] < target) l = m + 1;
        else r = m - 1;
    }
    return res;
}
```

### Search in Rotated Sorted Array

```java
public int search(int[] a, int target) {
    int l = 0, r = a.length - 1;
    while (l <= r) {
        int m = (l + r) / 2;
        if (a[m] == target) return m;
        if (a[l] <= a[m]) {                // left half sorted
            if (target >= a[l] && target < a[m]) r = m - 1;
            else l = m + 1;
        } else {                            // right half sorted
            if (target > a[m] && target <= a[r]) l = m + 1;
            else r = m - 1;
        }
    }
    return -1;
}
```

### Binary Search on Answer (advanced)

Some problems don't look like binary search but are — you binary search over the answer space.

Example: Koko Eating Bananas — find min eating speed to finish in H hours.
```java
public int minEatingSpeed(int[] piles, int h) {
    int l = 1, r = Arrays.stream(piles).max().getAsInt();
    while (l < r) {
        int m = (l + r) / 2;
        long hours = 0;
        for (int p : piles) hours += (p + m - 1) / m;    // ceil division
        if (hours <= h) r = m;
        else l = m + 1;
    }
    return l;
}
```

---

## 13.5 Linked List

```java
class ListNode {
    int val;
    ListNode next;
    ListNode(int v) { val = v; }
}
```

### Reverse Linked List — Iterative

```java
public ListNode reverse(ListNode head) {
    ListNode prev = null, cur = head;
    while (cur != null) {
        ListNode next = cur.next;
        cur.next = prev;
        prev = cur;
        cur = next;
    }
    return prev;
}
```

### Reverse — Recursive

```java
public ListNode reverseRec(ListNode head) {
    if (head == null || head.next == null) return head;
    ListNode p = reverseRec(head.next);
    head.next.next = head;
    head.next = null;
    return p;
}
```

### Detect Cycle — Floyd's tortoise & hare

```java
public boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true;
    }
    return false;
}
```

### Find start of cycle

```java
public ListNode detectCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next; fast = fast.next.next;
        if (slow == fast) {
            ListNode p = head;
            while (p != slow) { p = p.next; slow = slow.next; }
            return p;
        }
    }
    return null;
}
```

### Merge Two Sorted Lists

```java
public ListNode merge(ListNode a, ListNode b) {
    ListNode dummy = new ListNode(0), tail = dummy;
    while (a != null && b != null) {
        if (a.val <= b.val) { tail.next = a; a = a.next; }
        else                { tail.next = b; b = b.next; }
        tail = tail.next;
    }
    tail.next = (a != null) ? a : b;
    return dummy.next;
}
```

### Find middle (fast/slow)

```java
public ListNode middle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next; fast = fast.next.next;
    }
    return slow;
}
```

### Remove Nth from End — one pass

```java
public ListNode removeNthFromEnd(ListNode head, int n) {
    ListNode dummy = new ListNode(0); dummy.next = head;
    ListNode fast = dummy, slow = dummy;
    for (int i = 0; i <= n; i++) fast = fast.next;
    while (fast != null) { fast = fast.next; slow = slow.next; }
    slow.next = slow.next.next;
    return dummy.next;
}
```

---

## 13.6 Trees

```java
class TreeNode {
    int val;
    TreeNode left, right;
    TreeNode(int v) { val = v; }
}
```

### Traversals — Recursive

```java
void inorder(TreeNode n)   { if (n == null) return; inorder(n.left);  visit(n); inorder(n.right);  }
void preorder(TreeNode n)  { if (n == null) return; visit(n); preorder(n.left); preorder(n.right); }
void postorder(TreeNode n) { if (n == null) return; postorder(n.left); postorder(n.right); visit(n); }
```

### BFS — Level Order Traversal

```java
public List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> res = new ArrayList<>();
    if (root == null) return res;
    Queue<TreeNode> q = new LinkedList<>();
    q.offer(root);
    while (!q.isEmpty()) {
        int size = q.size();
        List<Integer> level = new ArrayList<>();
        for (int i = 0; i < size; i++) {
            TreeNode n = q.poll();
            level.add(n.val);
            if (n.left != null) q.offer(n.left);
            if (n.right != null) q.offer(n.right);
        }
        res.add(level);
    }
    return res;
}
```

### Max Depth

```java
public int maxDepth(TreeNode root) {
    if (root == null) return 0;
    return 1 + Math.max(maxDepth(root.left), maxDepth(root.right));
}
```

### LCA (Binary Tree)

```java
public TreeNode lca(TreeNode root, TreeNode p, TreeNode q) {
    if (root == null || root == p || root == q) return root;
    TreeNode left = lca(root.left, p, q);
    TreeNode right = lca(root.right, p, q);
    if (left != null && right != null) return root;
    return left != null ? left : right;
}
```

### Validate BST

```java
public boolean isValidBST(TreeNode root) {
    return validate(root, Long.MIN_VALUE, Long.MAX_VALUE);
}
private boolean validate(TreeNode n, long min, long max) {
    if (n == null) return true;
    if (n.val <= min || n.val >= max) return false;
    return validate(n.left, min, n.val) && validate(n.right, n.val, max);
}
```

### Serialize / Deserialize Binary Tree

```java
public String serialize(TreeNode root) {
    if (root == null) return "N";
    return root.val + "," + serialize(root.left) + "," + serialize(root.right);
}
public TreeNode deserialize(String data) {
    Deque<String> queue = new ArrayDeque<>(Arrays.asList(data.split(",")));
    return build(queue);
}
private TreeNode build(Deque<String> q) {
    String v = q.poll();
    if ("N".equals(v)) return null;
    TreeNode n = new TreeNode(Integer.parseInt(v));
    n.left = build(q);
    n.right = build(q);
    return n;
}
```

### Diameter of Binary Tree

```java
int diameter = 0;
public int diameterOfBinaryTree(TreeNode root) {
    height(root);
    return diameter;
}
private int height(TreeNode n) {
    if (n == null) return 0;
    int l = height(n.left), r = height(n.right);
    diameter = Math.max(diameter, l + r);
    return 1 + Math.max(l, r);
}
```

---

## 13.7 Heap / Priority Queue

Java `PriorityQueue` is a min-heap by default.

### Top K Frequent Elements

```java
public int[] topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int n : nums) freq.merge(n, 1, Integer::sum);

    // Min-heap by frequency; keep only k
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);
    for (var e : freq.entrySet()) {
        pq.offer(new int[]{ e.getKey(), e.getValue() });
        if (pq.size() > k) pq.poll();
    }
    int[] res = new int[k];
    for (int i = k - 1; i >= 0; i--) res[i] = pq.poll()[0];
    return res;
}
```

### Kth Largest Element

```java
public int findKthLargest(int[] nums, int k) {
    PriorityQueue<Integer> heap = new PriorityQueue<>();   // min-heap of size k
    for (int n : nums) {
        heap.offer(n);
        if (heap.size() > k) heap.poll();
    }
    return heap.peek();
}
```

### Merge K Sorted Lists

```java
public ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> pq = new PriorityQueue<>((a, b) -> a.val - b.val);
    for (ListNode l : lists) if (l != null) pq.offer(l);
    ListNode dummy = new ListNode(0), tail = dummy;
    while (!pq.isEmpty()) {
        ListNode n = pq.poll();
        tail.next = n; tail = n;
        if (n.next != null) pq.offer(n.next);
    }
    return dummy.next;
}
```

---

## 13.8 Graphs

### Representation
```java
// Adjacency list
List<List<Integer>> adj = new ArrayList<>();
for (int i = 0; i < n; i++) adj.add(new ArrayList<>());
adj.get(0).add(1);
```

### BFS

```java
void bfs(int start, List<List<Integer>> adj) {
    int n = adj.size();
    boolean[] visited = new boolean[n];
    Queue<Integer> q = new LinkedList<>();
    q.offer(start); visited[start] = true;
    while (!q.isEmpty()) {
        int u = q.poll();
        // process u
        for (int v : adj.get(u)) {
            if (!visited[v]) { visited[v] = true; q.offer(v); }
        }
    }
}
```

### DFS

```java
void dfs(int u, List<List<Integer>> adj, boolean[] visited) {
    visited[u] = true;
    // process u
    for (int v : adj.get(u)) if (!visited[v]) dfs(v, adj, visited);
}
```

### Number of Islands (2D grid DFS)

```java
public int numIslands(char[][] grid) {
    int count = 0;
    for (int i = 0; i < grid.length; i++)
        for (int j = 0; j < grid[0].length; j++)
            if (grid[i][j] == '1') { dfsIsland(grid, i, j); count++; }
    return count;
}
private void dfsIsland(char[][] g, int i, int j) {
    if (i < 0 || j < 0 || i >= g.length || j >= g[0].length || g[i][j] != '1') return;
    g[i][j] = '0';                                        // mark visited
    dfsIsland(g, i+1, j); dfsIsland(g, i-1, j);
    dfsIsland(g, i, j+1); dfsIsland(g, i, j-1);
}
```

### Topological Sort (Kahn's — BFS-based)

```java
public List<Integer> topoSort(int n, int[][] edges) {
    List<List<Integer>> adj = new ArrayList<>();
    int[] inDeg = new int[n];
    for (int i = 0; i < n; i++) adj.add(new ArrayList<>());
    for (int[] e : edges) { adj.get(e[0]).add(e[1]); inDeg[e[1]]++; }

    Queue<Integer> q = new LinkedList<>();
    for (int i = 0; i < n; i++) if (inDeg[i] == 0) q.offer(i);

    List<Integer> res = new ArrayList<>();
    while (!q.isEmpty()) {
        int u = q.poll(); res.add(u);
        for (int v : adj.get(u)) if (--inDeg[v] == 0) q.offer(v);
    }
    return res.size() == n ? res : new ArrayList<>();     // empty = cycle
}
```

### Dijkstra — Shortest Path (weighted, non-negative)

```java
public int[] dijkstra(int n, List<int[]>[] adj, int src) {
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[src] = 0;
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);   // {node, dist}
    pq.offer(new int[]{src, 0});

    while (!pq.isEmpty()) {
        int[] cur = pq.poll();
        int u = cur[0], d = cur[1];
        if (d > dist[u]) continue;
        for (int[] e : adj[u]) {
            int v = e[0], w = e[1];
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                pq.offer(new int[]{v, dist[v]});
            }
        }
    }
    return dist;
}
```

### Union-Find (Disjoint Set)

For connectivity problems.

```java
class UnionFind {
    int[] parent, rank;
    UnionFind(int n) {
        parent = new int[n]; rank = new int[n];
        for (int i = 0; i < n; i++) parent[i] = i;
    }
    int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);      // path compression
        return parent[x];
    }
    boolean union(int x, int y) {
        int px = find(x), py = find(y);
        if (px == py) return false;
        if (rank[px] < rank[py]) parent[px] = py;
        else if (rank[px] > rank[py]) parent[py] = px;
        else { parent[py] = px; rank[px]++; }
        return true;
    }
}
```

Use for: number of connected components, Kruskal's MST, cycle detection.

---

## 13.9 Dynamic Programming

**When:** overlapping subproblems + optimal substructure.

**Two styles:**
- **Top-down (memoization)** — recursive with cache
- **Bottom-up (tabulation)** — iterative table

### Fibonacci

```java
// Memoization
int fib(int n, Integer[] memo) {
    if (n <= 1) return n;
    if (memo[n] != null) return memo[n];
    return memo[n] = fib(n - 1, memo) + fib(n - 2, memo);
}

// Tabulation with O(1) space
int fibDp(int n) {
    if (n <= 1) return n;
    int a = 0, b = 1;
    for (int i = 2; i <= n; i++) { int c = a + b; a = b; b = c; }
    return b;
}
```

### Climbing Stairs

```java
public int climbStairs(int n) {
    if (n <= 2) return n;
    int a = 1, b = 2;
    for (int i = 3; i <= n; i++) { int c = a + b; a = b; b = c; }
    return b;
}
```

### House Robber (no two adjacent)

```java
public int rob(int[] nums) {
    int prev = 0, cur = 0;
    for (int n : nums) {
        int newCur = Math.max(cur, prev + n);
        prev = cur;
        cur = newCur;
    }
    return cur;
}
```

### Coin Change — minimum coins

```java
public int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1);
    dp[0] = 0;
    for (int i = 1; i <= amount; i++)
        for (int c : coins)
            if (c <= i) dp[i] = Math.min(dp[i], dp[i - c] + 1);
    return dp[amount] > amount ? -1 : dp[amount];
}
```

### Longest Common Subsequence — 2D DP

```java
public int lcs(String a, String b) {
    int m = a.length(), n = b.length();
    int[][] dp = new int[m + 1][n + 1];
    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++)
            dp[i][j] = a.charAt(i - 1) == b.charAt(j - 1)
                ? dp[i - 1][j - 1] + 1
                : Math.max(dp[i - 1][j], dp[i][j - 1]);
    return dp[m][n];
}
```

### 0/1 Knapsack

```java
public int knapsack(int[] wt, int[] val, int W) {
    int n = wt.length;
    int[][] dp = new int[n + 1][W + 1];
    for (int i = 1; i <= n; i++)
        for (int w = 0; w <= W; w++) {
            dp[i][w] = dp[i - 1][w];
            if (wt[i - 1] <= w)
                dp[i][w] = Math.max(dp[i][w], dp[i - 1][w - wt[i - 1]] + val[i - 1]);
        }
    return dp[n][W];
}
```

### Longest Increasing Subsequence — O(n log n)

```java
public int lengthOfLIS(int[] nums) {
    List<Integer> tails = new ArrayList<>();
    for (int n : nums) {
        int i = Collections.binarySearch(tails, n);
        if (i < 0) i = -(i + 1);
        if (i == tails.size()) tails.add(n);
        else tails.set(i, n);
    }
    return tails.size();
}
```

---

## 13.10 Backtracking

**When:** all permutations / combinations / subsets / partitions.

### Template

```
choose → explore → un-choose (backtrack)
```

### Subsets

```java
public List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> res = new ArrayList<>();
    backtrack(nums, 0, new ArrayList<>(), res);
    return res;
}
private void backtrack(int[] nums, int start, List<Integer> cur, List<List<Integer>> res) {
    res.add(new ArrayList<>(cur));
    for (int i = start; i < nums.length; i++) {
        cur.add(nums[i]);
        backtrack(nums, i + 1, cur, res);
        cur.remove(cur.size() - 1);
    }
}
```

### Permutations

```java
public List<List<Integer>> permute(int[] nums) {
    List<List<Integer>> res = new ArrayList<>();
    boolean[] used = new boolean[nums.length];
    backtrack(nums, used, new ArrayList<>(), res);
    return res;
}
private void backtrack(int[] nums, boolean[] used, List<Integer> cur, List<List<Integer>> res) {
    if (cur.size() == nums.length) { res.add(new ArrayList<>(cur)); return; }
    for (int i = 0; i < nums.length; i++) {
        if (used[i]) continue;
        used[i] = true; cur.add(nums[i]);
        backtrack(nums, used, cur, res);
        used[i] = false; cur.remove(cur.size() - 1);
    }
}
```

### N-Queens

```java
public List<List<String>> solveNQueens(int n) {
    List<List<String>> res = new ArrayList<>();
    char[][] board = new char[n][n];
    for (char[] row : board) Arrays.fill(row, '.');
    backtrack(board, 0, res);
    return res;
}
private void backtrack(char[][] b, int row, List<List<String>> res) {
    int n = b.length;
    if (row == n) {
        List<String> sol = new ArrayList<>();
        for (char[] r : b) sol.add(new String(r));
        res.add(sol);
        return;
    }
    for (int col = 0; col < n; col++) {
        if (safe(b, row, col)) {
            b[row][col] = 'Q';
            backtrack(b, row + 1, res);
            b[row][col] = '.';
        }
    }
}
private boolean safe(char[][] b, int row, int col) {
    int n = b.length;
    for (int i = 0; i < row; i++) if (b[i][col] == 'Q') return false;
    for (int i = row - 1, j = col - 1; i >= 0 && j >= 0; i--, j--) if (b[i][j] == 'Q') return false;
    for (int i = row - 1, j = col + 1; i >= 0 && j < n; i--, j++) if (b[i][j] == 'Q') return false;
    return true;
}
```

---

## 13.11 Problem-Solving Patterns Summary

| Pattern | When | Example |
|---------|------|---------|
| **Two Pointers** | sorted array, palindrome | 3Sum, Reverse |
| **Sliding Window** | subarray/substring | Longest substr, Min window |
| **Fast & Slow Pointers** | cycle detection, middle | LinkedList cycle |
| **Merge Intervals** | overlapping ranges | Meeting rooms |
| **Cyclic Sort** | numbers in [1..n] | Missing number |
| **Tree DFS/BFS** | tree traversal | Level order, LCA |
| **Topological Sort** | dependency ordering | Course Schedule |
| **Union-Find** | connectivity | Islands, Kruskal |
| **Backtracking** | all combinations | Subsets, N-Queens |
| **Greedy** | local optimum → global | Activity selection |
| **DP** | overlapping subproblems | LCS, Knapsack |
| **Binary Search on Answer** | monotonic | Koko Bananas |

---

## 13.12 Time Complexity Cheat Sheet

| Data structure | Access | Search | Insert | Delete |
|----------------|--------|--------|--------|--------|
| Array | O(1) | O(n) | O(n) | O(n) |
| Sorted array | O(1) | O(log n) | O(n) | O(n) |
| LinkedList | O(n) | O(n) | O(1) at ends | O(1) at ends |
| HashMap | – | O(1) avg | O(1) avg | O(1) avg |
| TreeMap | – | O(log n) | O(log n) | O(log n) |
| Heap | O(1) peek | – | O(log n) | O(log n) |
| BST (balanced) | – | O(log n) | O(log n) | O(log n) |

| Algorithm | Best | Avg | Worst | Space |
|-----------|------|-----|-------|-------|
| Bubble sort | O(n) | O(n²) | O(n²) | O(1) |
| Quicksort | O(n log n) | O(n log n) | O(n²) | O(log n) |
| Mergesort | O(n log n) | O(n log n) | O(n log n) | O(n) |
| Heapsort | O(n log n) | O(n log n) | O(n log n) | O(1) |
| BFS / DFS | O(V + E) | | | O(V) |
| Dijkstra (heap) | O((V+E) log V) | | | O(V) |

---

## 13.13 Interview Practice List

**Must-do (Blind 75 / NeetCode 150 highlights):**
- Two Sum, 3Sum
- Longest Substring Without Repeating Characters
- Valid Parentheses, Valid Anagram
- Reverse Linked List, Detect Cycle, Merge Two Sorted
- Number of Islands, Rotting Oranges
- Course Schedule (Topological)
- Coin Change, House Robber, LIS, LCS, 0/1 Knapsack
- Word Break
- Trapping Rain Water, Container With Most Water
- LRU Cache (design)
- Merge Intervals
- K largest / most frequent (heap)
- Serialize/Deserialize Binary Tree
- Word Ladder (BFS)
- Longest Palindromic Substring
- Rotate Image, Spiral Matrix

---

## 13.14 Interview Tips

- **Think out loud.** Interviewers care about your process, not just the answer.
- **Start with brute force.** Then optimize. Shows you can reason.
- **Test with examples.** Trace your code by hand.
- **Consider edge cases:** empty input, single element, all duplicates, negatives, overflow.
- **State time/space complexity** for every solution.
- **If stuck**, ask for hints — but only after honest effort.
- **Clean code**: meaningful variable names, small functions, no magic numbers.

---

## 13.15 Cheat Sheet

```
Two pointers: sorted array, palindromes
Sliding window: subarray/substring problems
HashMap: O(1) lookup — most array/string problems
Stack: parentheses, monotonic problems
Heap: top-K, streaming median
DFS/BFS: trees, graphs, grids
Union-Find: connectivity
DP: overlapping subproblems (memoize or tabulate)
Backtracking: permutations, combinations, N-Queens
Binary Search: sorted arrays, and "search on answer"
```

---

## Practical Assignments

1. Solve NeetCode 150 across all categories.
2. Implement common data structures from scratch: dynamic array, linked list, HashMap, LRU cache, min-heap.
3. Learn all Java built-in collections and their operations.
4. Time yourself: aim for 30-45 min per problem in prep mode.
5. Practice on whiteboard (or plain text editor without IDE completion) — many interviews are like this.

Master these and DSA interviews are yours. 🚀