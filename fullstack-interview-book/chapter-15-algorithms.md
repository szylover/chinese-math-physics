# 第十五章：算法与数据结构

算法题不是为了把你训练成竞赛选手，而是为了考察你是否具备“抽象问题、识别模式、选择数据结构、分析复杂度、稳定表达”的综合能力。对 Android、TypeScript、Node.js 工程师来说，面试官通常不会要求极端技巧，但会非常关注你是否能把经典题做出**工程化表达**：为什么选这个解法、是否能给出更优复杂度、边界条件如何处理、如果输入规模继续扩大怎么办。

本章按模式（pattern）组织 50 道高频 LeetCode 题。建议你复习时不要按题号死记，而要形成“看到题面就能映射到套路”的能力。

> 说明：链表、树、图题默认使用 LeetCode 提供的 `ListNode`、`TreeNode`、`Node` 等结构体定义，代码只展示核心逻辑。

## 面试中的算法表达模板

在白板或在线编程面试里，建议你用下面 5 句话快速起手：

1. **先确认目标**：输入输出是什么，是否允许修改原数组，是否存在重复元素。
2. **给出朴素解**：哪怕是 `O(n^2)`，先证明你能做出来。
3. **识别优化点**：能否用哈希（hash map）、双指针（two pointers）、单调栈（monotonic stack）、动态规划（dynamic programming）降低复杂度。
4. **说明不变量**：窗口里维护什么、栈里存什么、DP 状态表示什么。
5. **补充边界**：空数组、单元素、负数、重复值、溢出、图不连通。

---

## 30 天刷题计划

| 天数 | 主题 | 题目 |
|---|---|---|
| Day 1 | 数组入门 | #1 Two Sum, #238 Product of Array Except Self |
| Day 2 | 双指针 | #11 Container With Most Water, #15 Three Sum |
| Day 3 | 区间与前缀 | #56 Merge Intervals, #560 Subarray Sum Equals K |
| Day 4 | 高级数组 | #42 Trapping Rain Water, #239 Sliding Window Maximum |
| Day 5 | 字符串窗口 | #3 Longest Substring Without Repeating, #76 Minimum Window Substring |
| Day 6 | 字符串基础 | #20 Valid Parentheses, #49 Group Anagrams |
| Day 7 | 字符串进阶 | #5 Longest Palindromic Substring, #8 String to Integer |
| Day 8 | 链表基础 | #206 Reverse Linked List, #21 Merge Two Sorted Lists |
| Day 9 | 链表进阶 | #142 Linked List Cycle II, #138 Copy List with Random Pointer |
| Day 10 | 设计题 | #146 LRU Cache |
| Day 11 | 树遍历 | #102 Level Order, #98 Validate BST |
| Day 12 | 树递归 | #236 LCA, #297 Serialize/Deserialize Binary Tree |
| Day 13 | 图搜索 | #200 Number of Islands, #133 Clone Graph |
| Day 14 | 拓扑与最短路 | #207 Course Schedule, #127 Word Ladder |
| Day 15 | DP 入门 | #70 Climbing Stairs, #198 House Robber |
| Day 16 | DP 经典 | #62 Unique Paths, #322 Coin Change |
| Day 17 | 序列 DP | #300 LIS, #1143 LCS |
| Day 18 | 字符串 DP | #72 Edit Distance, #139 Word Break |
| Day 19 | 回溯入门 | #46 Permutations, #78 Subsets |
| Day 20 | 回溯进阶 | #39 Combination Sum, #79 Word Search |
| Day 21 | 回溯难题 | #51 N-Queens |
| Day 22 | 堆 | #215 Kth Largest, #347 Top K Frequent |
| Day 23 | 数据流 | #295 Find Median from Data Stream |
| Day 24 | 链表 + 堆 | #23 Merge K Sorted Lists |
| Day 25 | 栈 | #155 Min Stack, #739 Daily Temperatures |
| Day 26 | 单调栈 | #84 Largest Rectangle in Histogram |
| Day 27 | 设计基础 | #232 Queue using Stacks, #208 Trie |
| Day 28 | 哈希设计 | #706 Design HashMap |
| Day 29 | 综合复盘 | 树、图、DP 错题回顾 |
| Day 30 | 模拟面试 | 随机抽 5 题，限时讲解 + 手写 |

---

## 一、数组与双指针

### 1. Two Sum（#1）

- **题意**：给定数组和目标值，返回两数下标，使其和等于 target。
- **思路**：边遍历边用哈希表记录“值 -> 下标”。当看到 `num` 时，先查 `target - num` 是否出现过。核心是不回头扫描，把 `O(n^2)` 降成 `O(n)`。

```kotlin
fun twoSum(nums: IntArray, target: Int): IntArray {
    val map = HashMap<Int, Int>()
    for (i in nums.indices) {
        val need = target - nums[i]
        if (map.containsKey(need)) return intArrayOf(map[need]!!, i)
        map[nums[i]] = i
    }
    return intArrayOf()
}
```

```ts
function twoSum(nums: number[], target: number): number[] {
  const map = new Map<number, number>()
  for (let i = 0; i < nums.length; i++) {
    const need = target - nums[i]
    if (map.has(need)) return [map.get(need)!, i]
    map.set(nums[i], i)
  }
  return []
}
```

- **复杂度**：时间 `O(n)`，空间 `O(n)`。
- **面试沟通**：主动说出“如果要求返回所有组合，写法会不同；如果数组已排序，还可以双指针”。

### 2. Three Sum（#15）

- **题意**：找出所有和为 0 的不重复三元组。
- **思路**：先排序，再固定一个数，把问题转为区间上的 Two Sum。关键点是**去重**：固定点去重，左右指针移动后也要去重。

```kotlin
fun threeSum(nums: IntArray): List<List<Int>> {
    nums.sort()
    val res = mutableListOf<List<Int>>()
    for (i in 0 until nums.size - 2) {
        if (i > 0 && nums[i] == nums[i - 1]) continue
        var l = i + 1
        var r = nums.lastIndex
        while (l < r) {
            val sum = nums[i] + nums[l] + nums[r]
            when {
                sum == 0 -> {
                    res.add(listOf(nums[i], nums[l], nums[r]))
                    l++
                    r--
                    while (l < r && nums[l] == nums[l - 1]) l++
                    while (l < r && nums[r] == nums[r + 1]) r--
                }
                sum < 0 -> l++
                else -> r--
            }
        }
    }
    return res
}
```

```ts
function threeSum(nums: number[]): number[][] {
  nums.sort((a, b) => a - b)
  const res: number[][] = []
  for (let i = 0; i < nums.length - 2; i++) {
    if (i > 0 && nums[i] === nums[i - 1]) continue
    let l = i + 1, r = nums.length - 1
    while (l < r) {
      const sum = nums[i] + nums[l] + nums[r]
      if (sum === 0) {
        res.push([nums[i], nums[l], nums[r]])
        l++; r--
        while (l < r && nums[l] === nums[l - 1]) l++
        while (l < r && nums[r] === nums[r + 1]) r--
      } else if (sum < 0) l++
      else r--
    }
  }
  return res
}
```

- **复杂度**：时间 `O(n^2)`，空间 `O(1)`（不算结果集）。
- **面试沟通**：不要只说“排序 + 双指针”，一定补一句“排序是为了剪枝和去重”。

### 3. Container With Most Water（#11）

- **题意**：两条竖线围成最大面积。
- **思路**：面积由短板决定，所以每次移动更短的那一边才可能变大；移动高板没有意义。

```kotlin
fun maxArea(height: IntArray): Int {
    var l = 0
    var r = height.lastIndex
    var ans = 0
    while (l < r) {
        ans = maxOf(ans, minOf(height[l], height[r]) * (r - l))
        if (height[l] < height[r]) l++ else r--
    }
    return ans
}
```

```ts
function maxArea(height: number[]): number {
  let l = 0, r = height.length - 1, ans = 0
  while (l < r) {
    ans = Math.max(ans, Math.min(height[l], height[r]) * (r - l))
    if (height[l] < height[r]) l++
    else r--
  }
  return ans
}
```

- **复杂度**：时间 `O(n)`，空间 `O(1)`。
- **面试沟通**：强调“贪心（greedy）正确性来自短板效应”。

### 4. Trapping Rain Water（#42）

- **题意**：柱状图能接多少雨水。
- **思路**：双指针维护 `leftMax` 与 `rightMax`。哪边更小，就先结算哪边，因为那边的水位已被更小边界决定。

```kotlin
fun trap(height: IntArray): Int {
    var l = 0
    var r = height.lastIndex
    var leftMax = 0
    var rightMax = 0
    var ans = 0
    while (l < r) {
        if (height[l] < height[r]) {
            leftMax = maxOf(leftMax, height[l])
            ans += leftMax - height[l]
            l++
        } else {
            rightMax = maxOf(rightMax, height[r])
            ans += rightMax - height[r]
            r--
        }
    }
    return ans
}
```

```ts
function trap(height: number[]): number {
  let l = 0, r = height.length - 1
  let leftMax = 0, rightMax = 0, ans = 0
  while (l < r) {
    if (height[l] < height[r]) {
      leftMax = Math.max(leftMax, height[l])
      ans += leftMax - height[l]
      l++
    } else {
      rightMax = Math.max(rightMax, height[r])
      ans += rightMax - height[r]
      r--
    }
  }
  return ans
}
```

- **复杂度**：时间 `O(n)`，空间 `O(1)`。
- **面试沟通**：可顺带提“前后缀最大值数组解法是 `O(n)` 空间，双指针进一步优化空间”。

### 5. Merge Intervals（#56）

- **题意**：合并重叠区间。
- **思路**：先按左端点排序，再顺序扫描。当前区间起点小于等于结果尾区间终点时就合并，否则新增新区间。

```kotlin
fun merge(intervals: Array<IntArray>): Array<IntArray> {
    intervals.sortBy { it[0] }
    val res = mutableListOf<IntArray>()
    for (cur in intervals) {
        if (res.isEmpty() || res.last()[1] < cur[0]) res.add(cur.clone())
        else res.last()[1] = maxOf(res.last()[1], cur[1])
    }
    return res.toTypedArray()
}
```

```ts
function merge(intervals: number[][]): number[][] {
  intervals.sort((a, b) => a[0] - b[0])
  const res: number[][] = []
  for (const cur of intervals) {
    if (!res.length || res[res.length - 1][1] < cur[0]) res.push([...cur])
    else res[res.length - 1][1] = Math.max(res[res.length - 1][1], cur[1])
  }
  return res
}
```

- **复杂度**：时间 `O(n log n)`，空间 `O(n)`。
- **面试沟通**：说明排序是主复杂度来源；如果数据流实时到达，就要换成平衡树等结构。

### 6. Product of Array Except Self（#238）

- **题意**：返回除自身外其他元素乘积，不能用除法。
- **思路**：两遍扫描。第一遍存左侧乘积，第二遍用右侧乘积乘进去。

```kotlin
fun productExceptSelf(nums: IntArray): IntArray {
    val res = IntArray(nums.size) { 1 }
    var prefix = 1
    for (i in nums.indices) {
        res[i] = prefix
        prefix *= nums[i]
    }
    var suffix = 1
    for (i in nums.lastIndex downTo 0) {
        res[i] *= suffix
        suffix *= nums[i]
    }
    return res
}
```

```ts
function productExceptSelf(nums: number[]): number[] {
  const res = new Array(nums.length).fill(1)
  let prefix = 1
  for (let i = 0; i < nums.length; i++) {
    res[i] = prefix
    prefix *= nums[i]
  }
  let suffix = 1
  for (let i = nums.length - 1; i >= 0; i--) {
    res[i] *= suffix
    suffix *= nums[i]
  }
  return res
}
```

- **复杂度**：时间 `O(n)`，空间 `O(1)`（不算输出数组）。
- **面试沟通**：说明“结果数组被复用为前缀积容器，因此额外空间可视为常数”。

### 7. Sliding Window Maximum（#239）

- **题意**：输出每个长度为 `k` 的窗口最大值。
- **思路**：单调队列（monotonic deque）存下标，队首始终是当前窗口最大值下标；新元素入队前，把队尾更小元素弹出。

```kotlin
fun maxSlidingWindow(nums: IntArray, k: Int): IntArray {
    val deque = ArrayDeque<Int>()
    val res = IntArray(nums.size - k + 1)
    for (i in nums.indices) {
        while (deque.isNotEmpty() && deque.first() <= i - k) deque.removeFirst()
        while (deque.isNotEmpty() && nums[deque.last()] <= nums[i]) deque.removeLast()
        deque.addLast(i)
        if (i >= k - 1) res[i - k + 1] = nums[deque.first()]
    }
    return res
}
```

```ts
function maxSlidingWindow(nums: number[], k: number): number[] {
  const deque: number[] = []
  const res: number[] = []
  for (let i = 0; i < nums.length; i++) {
    if (deque.length && deque[0] <= i - k) deque.shift()
    while (deque.length && nums[deque[deque.length - 1]] <= nums[i]) deque.pop()
    deque.push(i)
    if (i >= k - 1) res.push(nums[deque[0]])
  }
  return res
}
```

- **复杂度**：时间 `O(n)`，空间 `O(k)`。
- **面试沟通**：一定说“每个元素最多进队出队一次，所以总复杂度不是 `O(nk)` 而是摊还 `O(n)`”。

### 8. Subarray Sum Equals K（#560）

- **题意**：统计和为 `k` 的连续子数组数量。
- **思路**：前缀和（prefix sum）+ 哈希表。若 `pre[j] - pre[i] = k`，则 `pre[i] = pre[j] - k`。扫描到当前位置时，查之前有多少个这样的前缀和。

```kotlin
fun subarraySum(nums: IntArray, k: Int): Int {
    val map = HashMap<Int, Int>()
    map[0] = 1
    var sum = 0
    var ans = 0
    for (num in nums) {
        sum += num
        ans += map.getOrDefault(sum - k, 0)
        map[sum] = map.getOrDefault(sum, 0) + 1
    }
    return ans
}
```

```ts
function subarraySum(nums: number[], k: number): number {
  const map = new Map<number, number>()
  map.set(0, 1)
  let sum = 0, ans = 0
  for (const num of nums) {
    sum += num
    ans += map.get(sum - k) ?? 0
    map.set(sum, (map.get(sum) ?? 0) + 1)
  }
  return ans
}
```

- **复杂度**：时间 `O(n)`，空间 `O(n)`。
- **面试沟通**：提醒面试官“此题不能用滑动窗口，因为有负数，窗口和不单调”。

---

## 二、字符串与滑动窗口

### 9. Longest Substring Without Repeating Characters（#3）

- **题意**：最长无重复字符子串长度。
- **思路**：滑动窗口（sliding window）维护字符最近出现位置。左边界只能右移不能左移。

```kotlin
fun lengthOfLongestSubstring(s: String): Int {
    val map = HashMap<Char, Int>()
    var left = 0
    var ans = 0
    for (right in s.indices) {
        if (map.containsKey(s[right])) left = maxOf(left, map[s[right]]!! + 1)
        map[s[right]] = right
        ans = maxOf(ans, right - left + 1)
    }
    return ans
}
```

```ts
function lengthOfLongestSubstring(s: string): number {
  const map = new Map<string, number>()
  let left = 0, ans = 0
  for (let right = 0; right < s.length; right++) {
    const ch = s[right]
    if (map.has(ch)) left = Math.max(left, map.get(ch)! + 1)
    map.set(ch, right)
    ans = Math.max(ans, right - left + 1)
  }
  return ans
}
```

- **复杂度**：时间 `O(n)`，空间 `O(min(n, charset))`。
- **面试沟通**：要解释为什么 `left = max(left, oldIndex + 1)`，否则会把左边界往回拉。

### 10. Minimum Window Substring（#76）

- **题意**：找出包含目标串全部字符的最短子串。
- **思路**：经典“先扩后缩”。`need` 统计目标字符频次，`window` 统计当前窗口频次；当满足所有字符后尝试收缩。

```kotlin
fun minWindow(s: String, t: String): String {
    val need = HashMap<Char, Int>()
    for (c in t) need[c] = need.getOrDefault(c, 0) + 1
    val window = HashMap<Char, Int>()
    var left = 0
    var valid = 0
    var start = 0
    var len = Int.MAX_VALUE
    for (right in s.indices) {
        val c = s[right]
        if (need.containsKey(c)) {
            window[c] = window.getOrDefault(c, 0) + 1
            if (window[c] == need[c]) valid++
        }
        while (valid == need.size) {
            if (right - left + 1 < len) {
                start = left
                len = right - left + 1
            }
            val d = s[left++]
            if (need.containsKey(d)) {
                if (window[d] == need[d]) valid--
                window[d] = window[d]!! - 1
            }
        }
    }
    return if (len == Int.MAX_VALUE) "" else s.substring(start, start + len)
}
```

```ts
function minWindow(s: string, t: string): string {
  const need = new Map<string, number>()
  for (const ch of t) need.set(ch, (need.get(ch) ?? 0) + 1)
  const window = new Map<string, number>()
  let left = 0, valid = 0, start = 0, len = Infinity
  for (let right = 0; right < s.length; right++) {
    const c = s[right]
    if (need.has(c)) {
      window.set(c, (window.get(c) ?? 0) + 1)
      if (window.get(c) === need.get(c)) valid++
    }
    while (valid === need.size) {
      if (right - left + 1 < len) {
        start = left
        len = right - left + 1
      }
      const d = s[left++]
      if (need.has(d)) {
        if (window.get(d) === need.get(d)) valid--
        window.set(d, window.get(d)! - 1)
      }
    }
  }
  return len === Infinity ? "" : s.slice(start, start + len)
}
```

- **复杂度**：时间 `O(m+n)`，空间 `O(charset)`。
- **面试沟通**：强调“valid 统计的是满足频次要求的字符种类数，而不是总字符数”。

### 11. Valid Parentheses（#20）

- **题意**：判断括号是否有效匹配。
- **思路**：栈（stack）保存尚未匹配的左括号。遇到右括号时检查栈顶是否为对应左括号。

```kotlin
fun isValid(s: String): Boolean {
    val map = mapOf(')' to '(', ']' to '[', '}' to '{')
    val stack = ArrayDeque<Char>()
    for (c in s) {
        if (c in map.values) stack.addLast(c)
        else if (stack.isEmpty() || stack.removeLast() != map[c]) return false
    }
    return stack.isEmpty()
}
```

```ts
function isValid(s: string): boolean {
  const map = new Map([[')', '('], [']', '['], ['}', '{']])
  const stack: string[] = []
  for (const ch of s) {
    if ([...map.values()].includes(ch)) stack.push(ch)
    else if (!stack.length || stack.pop() !== map.get(ch)) return false
  }
  return stack.length === 0
}
```

- **复杂度**：时间 `O(n)`，空间 `O(n)`。
- **面试沟通**：可顺带指出 TS 实战里应避免循环中频繁展开 `map.values()`，面试时关注思路即可。

### 12. Group Anagrams（#49）

- **题意**：把变位词（anagrams）分组。
- **思路**：把排序后的字符串作为 key。若要求更优常数，可用 26 个字母频次编码成 key。

```kotlin
fun groupAnagrams(strs: Array<String>): List<List<String>> {
    val map = HashMap<String, MutableList<String>>()
    for (s in strs) {
        val key = s.toCharArray().sorted().joinToString("")
        map.computeIfAbsent(key) { mutableListOf() }.add(s)
    }
    return map.values.toList()
}
```

```ts
function groupAnagrams(strs: string[]): string[][] {
  const map = new Map<string, string[]>()
  for (const s of strs) {
    const key = [...s].sort().join('')
    if (!map.has(key)) map.set(key, [])
    map.get(key)!.push(s)
  }
  return [...map.values()]
}
```

- **复杂度**：时间约 `O(n * k log k)`，空间 `O(nk)`。
- **面试沟通**：如果字符集固定为小写字母，主动补充“频次数组 key 更稳定”会加分。

### 13. Longest Palindromic Substring（#5）

- **题意**：求最长回文子串。
- **思路**：中心扩展（expand around center）。每个位置既可能是奇数中心，也可能是偶数中心。

```kotlin
fun longestPalindrome(s: String): String {
    var start = 0
    var end = 0
    fun expand(l0: Int, r0: Int) {
        var l = l0
        var r = r0
        while (l >= 0 && r < s.length && s[l] == s[r]) { l--; r++ }
        if (r - l - 2 > end - start) {
            start = l + 1
            end = r - 1
        }
    }
    for (i in s.indices) {
        expand(i, i)
        expand(i, i + 1)
    }
    return s.substring(start, end + 1)
}
```

```ts
function longestPalindrome(s: string): string {
  let start = 0, end = 0
  const expand = (l0: number, r0: number) => {
    let l = l0, r = r0
    while (l >= 0 && r < s.length && s[l] === s[r]) { l--; r++ }
    if (r - l - 2 > end - start) {
      start = l + 1
      end = r - 1
    }
  }
  for (let i = 0; i < s.length; i++) {
    expand(i, i)
    expand(i, i + 1)
  }
  return s.slice(start, end + 1)
}
```

- **复杂度**：时间 `O(n^2)`，空间 `O(1)`。
- **面试沟通**：若面试官追问更优解，可提 Manacher，但一般工程面试中心扩展已足够。

### 14. String to Integer (atoi)（#8）

- **题意**：模拟字符串转 32 位有符号整数。
- **思路**：跳过前导空格，处理正负号，连续读取数字，并在乘 10 之前做溢出判断。

```kotlin
fun myAtoi(s: String): Int {
    var i = 0
    while (i < s.length && s[i] == ' ') i++
    var sign = 1
    if (i < s.length && (s[i] == '+' || s[i] == '-')) sign = if (s[i++] == '-') -1 else 1
    var ans = 0
    while (i < s.length && s[i].isDigit()) {
        val d = s[i] - '0'
        if (ans > Int.MAX_VALUE / 10 || (ans == Int.MAX_VALUE / 10 && d > 7))
            return if (sign == 1) Int.MAX_VALUE else Int.MIN_VALUE
        ans = ans * 10 + d
        i++
    }
    return ans * sign
}
```

```ts
function myAtoi(s: string): number {
  let i = 0
  while (i < s.length && s[i] === ' ') i++
  let sign = 1
  if (i < s.length && (s[i] === '+' || s[i] === '-')) sign = s[i++] === '-' ? -1 : 1
  let ans = 0
  while (i < s.length && /\d/.test(s[i])) {
    const d = s.charCodeAt(i) - 48
    if (ans > Math.trunc(2147483647 / 10) || (ans === 214748364 && d > 7))
      return sign === 1 ? 2147483647 : -2147483648
    ans = ans * 10 + d
    i++
  }
  return ans * sign
}
```

- **复杂度**：时间 `O(n)`，空间 `O(1)`。
- **面试沟通**：这题重点不是 API，而是“状态机思维 + 边界处理”。

---

## 三、链表

### 15. Reverse Linked List（#206）

- **题意**：反转单链表。
- **思路**：用 `prev / curr / next` 三指针原地反转。

```kotlin
fun reverseList(head: ListNode?): ListNode? {
    var prev: ListNode? = null
    var curr = head
    while (curr != null) {
        val next = curr.next
        curr.next = prev
        prev = curr
        curr = next
    }
    return prev
}
```

```ts
function reverseList(head: ListNode | null): ListNode | null {
  let prev: ListNode | null = null
  let curr = head
  while (curr) {
    const next = curr.next
    curr.next = prev
    prev = curr
    curr = next
  }
  return prev
}
```

- **复杂度**：时间 `O(n)`，空间 `O(1)`。
- **面试沟通**：可补充递归版，但要指出递归会消耗调用栈。

### 16. Merge Two Sorted Lists（#21）

- **题意**：合并两个有序链表。
- **思路**：虚拟头节点（dummy head）减少边界分支。

```kotlin
fun mergeTwoLists(list1: ListNode?, list2: ListNode?): ListNode? {
    val dummy = ListNode(0)
    var tail = dummy
    var l1 = list1
    var l2 = list2
    while (l1 != null && l2 != null) {
        if (l1.`val` < l2.`val`) { tail.next = l1; l1 = l1.next }
        else { tail.next = l2; l2 = l2.next }
        tail = tail.next!!
    }
    tail.next = l1 ?: l2
    return dummy.next
}
```

```ts
function mergeTwoLists(list1: ListNode | null, list2: ListNode | null): ListNode | null {
  const dummy = new ListNode(0)
  let tail = dummy, l1 = list1, l2 = list2
  while (l1 && l2) {
    if (l1.val < l2.val) { tail.next = l1; l1 = l1.next }
    else { tail.next = l2; l2 = l2.next }
    tail = tail.next
  }
  tail.next = l1 ?? l2
  return dummy.next
}
```

- **复杂度**：时间 `O(m+n)`，空间 `O(1)`。
- **面试沟通**：强调“dummy head 是链表题常见技巧，可减少头节点特殊处理”。

### 17. Linked List Cycle II（#142）

- **题意**：找出环的入口节点。
- **思路**：快慢指针（fast/slow pointers）先判断相遇；相遇后从头和相遇点同步前进，再次相遇即入口。

```kotlin
fun detectCycle(head: ListNode?): ListNode? {
    var slow = head
    var fast = head
    while (fast?.next != null) {
        slow = slow?.next
        fast = fast.next?.next
        if (slow == fast) {
            var p1 = head
            var p2 = slow
            while (p1 != p2) {
                p1 = p1?.next
                p2 = p2?.next
            }
            return p1
        }
    }
    return null
}
```

```ts
function detectCycle(head: ListNode | null): ListNode | null {
  let slow = head, fast = head
  while (fast?.next) {
    slow = slow!.next
    fast = fast.next.next
    if (slow === fast) {
      let p1 = head, p2 = slow
      while (p1 !== p2) {
        p1 = p1!.next
        p2 = p2!.next
      }
      return p1
    }
  }
  return null
}
```

- **复杂度**：时间 `O(n)`，空间 `O(1)`。
- **面试沟通**：最好能口述数学推导：`head 到入口 = 相遇点到入口`。

### 18. LRU Cache（#146）

- **题意**：实现最近最少使用缓存（Least Recently Used）。
- **思路**：哈希表 + 双向链表（doubly linked list）。哈希表 `O(1)` 找节点，链表 `O(1)` 调整最近访问顺序。

```kotlin
class LRUCache(private val capacity: Int) {
    private inner class Node(var key: Int, var value: Int) {
        var prev: Node? = null
        var next: Node? = null
    }
    private val map = HashMap<Int, Node>()
    private val head = Node(0, 0)
    private val tail = Node(0, 0)
    init { head.next = tail; tail.prev = head }
    private fun remove(node: Node) { node.prev!!.next = node.next; node.next!!.prev = node.prev }
    private fun addFirst(node: Node) {
        node.next = head.next; node.prev = head
        head.next!!.prev = node; head.next = node
    }
    fun get(key: Int): Int {
        val node = map[key] ?: return -1
        remove(node); addFirst(node)
        return node.value
    }
    fun put(key: Int, value: Int) {
        if (map.containsKey(key)) {
            val node = map[key]!!
            node.value = value
            remove(node); addFirst(node)
            return
        }
        val node = Node(key, value)
        map[key] = node
        addFirst(node)
        if (map.size > capacity) {
            val last = tail.prev!!
            remove(last)
            map.remove(last.key)
        }
    }
}
```

```ts
class LRUNode {
  key: number
  value: number
  prev: LRUNode | null = null
  next: LRUNode | null = null
  constructor(key: number, value: number) { this.key = key; this.value = value }
}
class LRUCache {
  private map = new Map<number, LRUNode>()
  private head = new LRUNode(0, 0)
  private tail = new LRUNode(0, 0)
  constructor(private capacity: number) {
    this.head.next = this.tail
    this.tail.prev = this.head
  }
  private remove(node: LRUNode) {
    node.prev!.next = node.next
    node.next!.prev = node.prev
  }
  private addFirst(node: LRUNode) {
    node.next = this.head.next
    node.prev = this.head
    this.head.next!.prev = node
    this.head.next = node
  }
  get(key: number): number {
    const node = this.map.get(key)
    if (!node) return -1
    this.remove(node); this.addFirst(node)
    return node.value
  }
  put(key: number, value: number): void {
    if (this.map.has(key)) {
      const node = this.map.get(key)!
      node.value = value
      this.remove(node); this.addFirst(node)
      return
    }
    const node = new LRUNode(key, value)
    this.map.set(key, node)
    this.addFirst(node)
    if (this.map.size > this.capacity) {
      const last = this.tail.prev!
      this.remove(last)
      this.map.delete(last.key)
    }
  }
}
```

- **复杂度**：`get/put` 均为时间 `O(1)`，空间 `O(capacity)`。
- **面试沟通**：设计题必须先说接口、数据结构、复杂度，再写代码。

### 19. Copy List with Random Pointer（#138）

- **题意**：复制带随机指针的链表。
- **思路**：哈希映射“旧节点 -> 新节点”，两遍扫描：第一遍建点，第二遍连 `next/random`。

```kotlin
fun copyRandomList(node: Node?): Node? {
    if (node == null) return null
    val map = HashMap<Node, Node>()
    var cur: Node? = node
    while (cur != null) {
        map[cur] = Node(cur.`val`)
        cur = cur.next
    }
    cur = node
    while (cur != null) {
        map[cur]!!.next = map[cur.next]
        map[cur]!!.random = map[cur.random]
        cur = cur.next
    }
    return map[node]
}
```

```ts
function copyRandomList(head: Node | null): Node | null {
  if (!head) return null
  const map = new Map<Node, Node>()
  let cur: Node | null = head
  while (cur) {
    map.set(cur, new Node(cur.val))
    cur = cur.next
  }
  cur = head
  while (cur) {
    map.get(cur)!.next = cur.next ? map.get(cur.next)! : null
    map.get(cur)!.random = cur.random ? map.get(cur.random)! : null
    cur = cur.next
  }
  return map.get(head)!
}
```

- **复杂度**：时间 `O(n)`，空间 `O(n)`。
- **面试沟通**：可以补充 `O(1)` 额外空间的“节点穿插复制”解法，体现深度。

---

## 四、树与图

### 20. Binary Tree Level Order Traversal（#102）

- **题意**：按层遍历二叉树。
- **思路**：广度优先搜索（BFS），队列按层处理。

```kotlin
fun levelOrder(root: TreeNode?): List<List<Int>> {
    if (root == null) return emptyList()
    val res = mutableListOf<List<Int>>()
    val q = ArrayDeque<TreeNode>()
    q.addLast(root)
    while (q.isNotEmpty()) {
        val size = q.size
        val level = mutableListOf<Int>()
        repeat(size) {
            val node = q.removeFirst()
            level.add(node.`val`)
            node.left?.let { q.addLast(it) }
            node.right?.let { q.addLast(it) }
        }
        res.add(level)
    }
    return res
}
```

```ts
function levelOrder(root: TreeNode | null): number[][] {
  if (!root) return []
  const res: number[][] = []
  const q: TreeNode[] = [root]
  while (q.length) {
    const size = q.length
    const level: number[] = []
    for (let i = 0; i < size; i++) {
      const node = q.shift()!
      level.push(node.val)
      if (node.left) q.push(node.left)
      if (node.right) q.push(node.right)
    }
    res.push(level)
  }
  return res
}
```

- **复杂度**：时间 `O(n)`，空间 `O(n)`。
- **面试沟通**：说清楚 DFS 也能做，但层序天然对应 BFS。

### 21. Validate Binary Search Tree（#98）

- **题意**：判断是否为合法 BST。
- **思路**：递归传上下界，所有左子树节点都必须 `< root`，右子树都必须 `> root`。

```kotlin
fun isValidBST(root: TreeNode?): Boolean {
    fun dfs(node: TreeNode?, low: Long, high: Long): Boolean {
        if (node == null) return true
        if (node.`val`.toLong() <= low || node.`val`.toLong() >= high) return false
        return dfs(node.left, low, node.`val`.toLong()) &&
                dfs(node.right, node.`val`.toLong(), high)
    }
    return dfs(root, Long.MIN_VALUE, Long.MAX_VALUE)
}
```

```ts
function isValidBST(root: TreeNode | null): boolean {
  const dfs = (node: TreeNode | null, low: number, high: number): boolean => {
    if (!node) return true
    if (node.val <= low || node.val >= high) return false
    return dfs(node.left, low, node.val) && dfs(node.right, node.val, high)
  }
  return dfs(root, -Infinity, Infinity)
}
```

- **复杂度**：时间 `O(n)`，空间 `O(h)`。
- **面试沟通**：不要犯“只比较父子节点”的错误，那样会漏掉跨层约束。

### 22. Lowest Common Ancestor of a Binary Tree（#236）

- **题意**：求二叉树两个节点的最近公共祖先。
- **思路**：后序递归。若左右子树分别找到一个目标，则当前节点就是答案。

```kotlin
fun lowestCommonAncestor(root: TreeNode?, p: TreeNode?, q: TreeNode?): TreeNode? {
    if (root == null || root == p || root == q) return root
    val left = lowestCommonAncestor(root.left, p, q)
    val right = lowestCommonAncestor(root.right, p, q)
    return when {
        left != null && right != null -> root
        left != null -> left
        else -> right
    }
}
```

```ts
function lowestCommonAncestor(root: TreeNode | null, p: TreeNode | null, q: TreeNode | null): TreeNode | null {
  if (!root || root === p || root === q) return root
  const left = lowestCommonAncestor(root.left, p, q)
  const right = lowestCommonAncestor(root.right, p, q)
  if (left && right) return root
  return left ?? right
}
```

- **复杂度**：时间 `O(n)`，空间 `O(h)`。
- **面试沟通**：面试里最好口头说明递归返回值的含义：返回“当前子树里找到的目标节点或公共祖先”。

### 23. Serialize and Deserialize Binary Tree（#297）

- **题意**：序列化与反序列化二叉树。
- **思路**：前序遍历 + 空节点占位，例如 `#`。反序列化时按相同顺序消费 token。

```kotlin
class Codec {
    fun serialize(root: TreeNode?): String {
        val res = mutableListOf<String>()
        fun dfs(node: TreeNode?) {
            if (node == null) { res.add("#"); return }
            res.add(node.`val`.toString())
            dfs(node.left); dfs(node.right)
        }
        dfs(root)
        return res.joinToString(",")
    }
    fun deserialize(data: String): TreeNode? {
        val q = ArrayDeque(data.split(","))
        fun dfs(): TreeNode? {
            val cur = q.removeFirst()
            if (cur == "#") return null
            val node = TreeNode(cur.toInt())
            node.left = dfs()
            node.right = dfs()
            return node
        }
        return dfs()
    }
}
```

```ts
class Codec {
  serialize(root: TreeNode | null): string {
    const res: string[] = []
    const dfs = (node: TreeNode | null) => {
      if (!node) { res.push('#'); return }
      res.push(String(node.val))
      dfs(node.left); dfs(node.right)
    }
    dfs(root)
    return res.join(',')
  }
  deserialize(data: string): TreeNode | null {
    const arr = data.split(',')
    let i = 0
    const dfs = (): TreeNode | null => {
      const cur = arr[i++]
      if (cur === '#') return null
      const node = new TreeNode(Number(cur))
      node.left = dfs()
      node.right = dfs()
      return node
    }
    return dfs()
  }
}
```

- **复杂度**：时间 `O(n)`，空间 `O(n)`。
- **面试沟通**：设计题要先约定协议格式，再谈可逆性和边界。

### 24. Number of Islands（#200）

- **题意**：统计网格中岛屿数量。
- **思路**：扫描网格，遇到陆地就 DFS/BFS 淹没整座岛，并计数。

```kotlin
fun numIslands(grid: Array<CharArray>): Int {
    val m = grid.size
    val n = grid[0].size
    fun dfs(i: Int, j: Int) {
        if (i !in 0 until m || j !in 0 until n || grid[i][j] != '1') return
        grid[i][j] = '0'
        dfs(i + 1, j); dfs(i - 1, j); dfs(i, j + 1); dfs(i, j - 1)
    }
    var ans = 0
    for (i in 0 until m) for (j in 0 until n) {
        if (grid[i][j] == '1') { ans++; dfs(i, j) }
    }
    return ans
}
```

```ts
function numIslands(grid: string[][]): number {
  const m = grid.length, n = grid[0].length
  const dfs = (i: number, j: number) => {
    if (i < 0 || i >= m || j < 0 || j >= n || grid[i][j] !== '1') return
    grid[i][j] = '0'
    dfs(i + 1, j); dfs(i - 1, j); dfs(i, j + 1); dfs(i, j - 1)
  }
  let ans = 0
  for (let i = 0; i < m; i++) {
    for (let j = 0; j < n; j++) {
      if (grid[i][j] === '1') { ans++; dfs(i, j) }
    }
  }
  return ans
}
```

- **复杂度**：时间 `O(mn)`，空间 `O(mn)`（递归栈最坏）。
- **面试沟通**：记得问“是否允许原地修改输入网格”，这体现工程习惯。

### 25. Course Schedule（#207）

- **题意**：判断课程是否能全部修完，本质是有向图是否有环。
- **思路**：拓扑排序（topological sort），统计入度。不断弹出入度为 0 的节点。

```kotlin
fun canFinish(numCourses: Int, prerequisites: Array<IntArray>): Boolean {
    val graph = Array(numCourses) { mutableListOf<Int>() }
    val indegree = IntArray(numCourses)
    for ((a, b) in prerequisites) {
        graph[b].add(a)
        indegree[a]++
    }
    val q = ArrayDeque<Int>()
    for (i in 0 until numCourses) if (indegree[i] == 0) q.addLast(i)
    var count = 0
    while (q.isNotEmpty()) {
        val cur = q.removeFirst()
        count++
        for (next in graph[cur]) {
            if (--indegree[next] == 0) q.addLast(next)
        }
    }
    return count == numCourses
}
```

```ts
function canFinish(numCourses: number, prerequisites: number[][]): boolean {
  const graph: number[][] = Array.from({ length: numCourses }, () => [])
  const indegree = new Array(numCourses).fill(0)
  for (const [a, b] of prerequisites) {
    graph[b].push(a)
    indegree[a]++
  }
  const q: number[] = []
  indegree.forEach((v, i) => { if (v === 0) q.push(i) })
  let count = 0
  while (q.length) {
    const cur = q.shift()!
    count++
    for (const next of graph[cur]) if (--indegree[next] === 0) q.push(next)
  }
  return count === numCourses
}
```

- **复杂度**：时间 `O(V+E)`，空间 `O(V+E)`。
- **面试沟通**：如果面试官追问，也能用 DFS 染色法判环。

### 26. Word Ladder（#127）

- **题意**：单词接龙最短转换长度。
- **思路**：这是最短路径，优先想到 BFS。每次尝试把每个位置替换成 26 个字母生成邻居。

```kotlin
fun ladderLength(beginWord: String, endWord: String, wordList: List<String>): Int {
    val dict = wordList.toMutableSet()
    if (endWord !in dict) return 0
    val q = ArrayDeque<Pair<String, Int>>()
    q.addLast(beginWord to 1)
    while (q.isNotEmpty()) {
        val (word, step) = q.removeFirst()
        if (word == endWord) return step
        val arr = word.toCharArray()
        for (i in arr.indices) {
            val old = arr[i]
            for (c in 'a'..'z') {
                arr[i] = c
                val next = String(arr)
                if (next in dict) {
                    dict.remove(next)
                    q.addLast(next to step + 1)
                }
            }
            arr[i] = old
        }
    }
    return 0
}
```

```ts
function ladderLength(beginWord: string, endWord: string, wordList: string[]): number {
  const dict = new Set(wordList)
  if (!dict.has(endWord)) return 0
  const q: Array<[string, number]> = [[beginWord, 1]]
  while (q.length) {
    const [word, step] = q.shift()!
    if (word === endWord) return step
    const arr = word.split('')
    for (let i = 0; i < arr.length; i++) {
      const old = arr[i]
      for (let code = 97; code <= 122; code++) {
        arr[i] = String.fromCharCode(code)
        const next = arr.join('')
        if (dict.has(next)) {
          dict.delete(next)
          q.push([next, step + 1])
        }
      }
      arr[i] = old
    }
  }
  return 0
}
```

- **复杂度**：时间约 `O(N * L * 26)`，空间 `O(N)`。
- **面试沟通**：大数据量时可主动提双向 BFS（bidirectional BFS）优化。

### 27. Clone Graph（#133）

- **题意**：深拷贝无向图。
- **思路**：DFS/BFS 都行，关键是哈希表避免重复克隆与死循环。

```kotlin
fun cloneGraph(node: Node?): Node? {
    if (node == null) return null
    val map = HashMap<Node, Node>()
    fun dfs(cur: Node): Node {
        if (map.containsKey(cur)) return map[cur]!!
        val copy = Node(cur.`val`)
        map[cur] = copy
        for (nei in cur.neighbors) copy.neighbors.add(dfs(nei))
        return copy
    }
    return dfs(node)
}
```

```ts
function cloneGraph(node: Node | null): Node | null {
  if (!node) return null
  const map = new Map<Node, Node>()
  const dfs = (cur: Node): Node => {
    if (map.has(cur)) return map.get(cur)!
    const copy = new Node(cur.val)
    map.set(cur, copy)
    for (const nei of cur.neighbors) copy.neighbors.push(dfs(nei))
    return copy
  }
  return dfs(node)
}
```

- **复杂度**：时间 `O(V+E)`，空间 `O(V)`。
- **面试沟通**：深拷贝题一定强调“节点身份”而不是值，值可能重复。

---

## 五、动态规划

### 28. Climbing Stairs（#70）

- **题意**：每次爬 1 或 2 阶，求到顶方案数。
- **思路**：`dp[i] = dp[i-1] + dp[i-2]`，本质是斐波那契。

```kotlin
fun climbStairs(n: Int): Int {
    if (n <= 2) return n
    var a = 1
    var b = 2
    for (i in 3..n) {
        val c = a + b
        a = b
        b = c
    }
    return b
}
```

```ts
function climbStairs(n: number): number {
  if (n <= 2) return n
  let a = 1, b = 2
  for (let i = 3; i <= n; i++) {
    const c = a + b
    a = b
    b = c
  }
  return b
}
```

- **复杂度**：时间 `O(n)`，空间 `O(1)`。
- **面试沟通**：DP 面试一定先说状态、转移、初始化。

### 29. Coin Change（#322）

- **题意**：最少硬币数凑成金额。
- **思路**：完全背包（complete knapsack）的一维 DP。`dp[i]` 表示凑到 `i` 的最少硬币数。

```kotlin
fun coinChange(coins: IntArray, amount: Int): Int {
    val dp = IntArray(amount + 1) { amount + 1 }
    dp[0] = 0
    for (i in 1..amount) {
        for (coin in coins) if (i >= coin) dp[i] = minOf(dp[i], dp[i - coin] + 1)
    }
    return if (dp[amount] > amount) -1 else dp[amount]
}
```

```ts
function coinChange(coins: number[], amount: number): number {
  const dp = new Array(amount + 1).fill(amount + 1)
  dp[0] = 0
  for (let i = 1; i <= amount; i++) {
    for (const coin of coins) if (i >= coin) dp[i] = Math.min(dp[i], dp[i - coin] + 1)
  }
  return dp[amount] > amount ? -1 : dp[amount]
}
```

- **复杂度**：时间 `O(amount * n)`，空间 `O(amount)`。
- **面试沟通**：可对比 DFS 回溯会超时，DP 适合“最优子结构 + 重叠子问题”。

### 30. Longest Increasing Subsequence（#300）

- **题意**：最长严格递增子序列长度。
- **思路**：面试中先给 `O(n^2)` DP，再提 `O(n log n)` 的 patience sorting 思路。这里写高频优化版。

```kotlin
fun lengthOfLIS(nums: IntArray): Int {
    val tails = mutableListOf<Int>()
    for (num in nums) {
        var l = 0
        var r = tails.size
        while (l < r) {
            val m = (l + r) ushr 1
            if (tails[m] < num) l = m + 1 else r = m
        }
        if (l == tails.size) tails.add(num) else tails[l] = num
    }
    return tails.size
}
```

```ts
function lengthOfLIS(nums: number[]): number {
  const tails: number[] = []
  for (const num of nums) {
    let l = 0, r = tails.length
    while (l < r) {
      const m = (l + r) >> 1
      if (tails[m] < num) l = m + 1
      else r = m
    }
    if (l === tails.length) tails.push(num)
    else tails[l] = num
  }
  return tails.length
}
```

- **复杂度**：时间 `O(n log n)`，空间 `O(n)`。
- **面试沟通**：一定说明 `tails[i]` 不是实际答案，而是“长度为 i+1 的递增子序列的最小结尾”。

### 31. Edit Distance（#72）

- **题意**：把 word1 变成 word2 的最少操作数。
- **思路**：二维 DP，三种操作：插入、删除、替换。

```kotlin
fun minDistance(word1: String, word2: String): Int {
    val m = word1.length
    val n = word2.length
    val dp = Array(m + 1) { IntArray(n + 1) }
    for (i in 0..m) dp[i][0] = i
    for (j in 0..n) dp[0][j] = j
    for (i in 1..m) for (j in 1..n) {
        dp[i][j] = if (word1[i - 1] == word2[j - 1]) dp[i - 1][j - 1]
        else minOf(dp[i - 1][j], dp[i][j - 1], dp[i - 1][j - 1]) + 1
    }
    return dp[m][n]
}
```

```ts
function minDistance(word1: string, word2: string): number {
  const m = word1.length, n = word2.length
  const dp = Array.from({ length: m + 1 }, (_, i) => new Array(n + 1).fill(0))
  for (let i = 0; i <= m; i++) dp[i][0] = i
  for (let j = 0; j <= n; j++) dp[0][j] = j
  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      dp[i][j] = word1[i - 1] === word2[j - 1]
        ? dp[i - 1][j - 1]
        : Math.min(dp[i - 1][j], dp[i][j - 1], dp[i - 1][j - 1]) + 1
    }
  }
  return dp[m][n]
}
```

- **复杂度**：时间 `O(mn)`，空间 `O(mn)`。
- **面试沟通**：若面试官追问，能提滚动数组优化到 `O(n)` 空间。

### 32. Longest Common Subsequence（#1143）

- **题意**：求两个字符串最长公共子序列长度。
- **思路**：`dp[i][j]` 表示前 `i` 和前 `j` 的答案。字符相等取左上 + 1，否则取上或左最大值。

```kotlin
fun longestCommonSubsequence(text1: String, text2: String): Int {
    val m = text1.length
    val n = text2.length
    val dp = Array(m + 1) { IntArray(n + 1) }
    for (i in 1..m) for (j in 1..n) {
        dp[i][j] = if (text1[i - 1] == text2[j - 1]) dp[i - 1][j - 1] + 1
        else maxOf(dp[i - 1][j], dp[i][j - 1])
    }
    return dp[m][n]
}
```

```ts
function longestCommonSubsequence(text1: string, text2: string): number {
  const m = text1.length, n = text2.length
  const dp = Array.from({ length: m + 1 }, () => new Array(n + 1).fill(0))
  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      dp[i][j] = text1[i - 1] === text2[j - 1]
        ? dp[i - 1][j - 1] + 1
        : Math.max(dp[i - 1][j], dp[i][j - 1])
    }
  }
  return dp[m][n]
}
```

- **复杂度**：时间 `O(mn)`，空间 `O(mn)`。
- **面试沟通**：注意“子序列”不是“子串”，不要求连续。

### 33. Word Break（#139）

- **题意**：判断字符串能否被字典拆分。
- **思路**：`dp[i]` 表示前 `i` 个字符是否可拆分。枚举最后一刀位置 `j`。

```kotlin
fun wordBreak(s: String, wordDict: List<String>): Boolean {
    val set = wordDict.toHashSet()
    val dp = BooleanArray(s.length + 1)
    dp[0] = true
    for (i in 1..s.length) {
        for (j in 0 until i) {
            if (dp[j] && s.substring(j, i) in set) {
                dp[i] = true
                break
            }
        }
    }
    return dp[s.length]
}
```

```ts
function wordBreak(s: string, wordDict: string[]): boolean {
  const set = new Set(wordDict)
  const dp = new Array(s.length + 1).fill(false)
  dp[0] = true
  for (let i = 1; i <= s.length; i++) {
    for (let j = 0; j < i; j++) {
      if (dp[j] && set.has(s.slice(j, i))) {
        dp[i] = true
        break
      }
    }
  }
  return dp[s.length]
}
```

- **复杂度**：时间 `O(n^3)`（切片可能复制），空间 `O(n)`。
- **面试沟通**：可补充 Trie 优化或限制最大单词长度进行剪枝。

### 34. House Robber（#198）

- **题意**：相邻房屋不能同时偷，求最大收益。
- **思路**：`dp[i] = max(dp[i-1], dp[i-2] + nums[i])`。

```kotlin
fun rob(nums: IntArray): Int {
    var prev2 = 0
    var prev1 = 0
    for (num in nums) {
        val cur = maxOf(prev1, prev2 + num)
        prev2 = prev1
        prev1 = cur
    }
    return prev1
}
```

```ts
function rob(nums: number[]): number {
  let prev2 = 0, prev1 = 0
  for (const num of nums) {
    const cur = Math.max(prev1, prev2 + num)
    prev2 = prev1
    prev1 = cur
  }
  return prev1
}
```

- **复杂度**：时间 `O(n)`，空间 `O(1)`。
- **面试沟通**：说明状态压缩来自“当前只依赖前两项”。

### 35. Unique Paths（#62）

- **题意**：机器人从左上到右下，只能向右或向下。
- **思路**：每格路径数 = 上方 + 左方。

```kotlin
fun uniquePaths(m: Int, n: Int): Int {
    val dp = IntArray(n) { 1 }
    for (i in 1 until m) {
        for (j in 1 until n) dp[j] += dp[j - 1]
    }
    return dp[n - 1]
}
```

```ts
function uniquePaths(m: number, n: number): number {
  const dp = new Array(n).fill(1)
  for (let i = 1; i < m; i++) {
    for (let j = 1; j < n; j++) dp[j] += dp[j - 1]
  }
  return dp[n - 1]
}
```

- **复杂度**：时间 `O(mn)`，空间 `O(n)`。
- **面试沟通**：可补充组合数学解法 `C(m+n-2, m-1)`。

---

## 六、回溯

### 36. Permutations（#46）

- **题意**：返回全排列。
- **思路**：回溯（backtracking）路径中放已选元素，用 `used` 数组记录状态。

```kotlin
fun permute(nums: IntArray): List<List<Int>> {
    val res = mutableListOf<List<Int>>()
    val path = mutableListOf<Int>()
    val used = BooleanArray(nums.size)
    fun dfs() {
        if (path.size == nums.size) { res.add(path.toList()); return }
        for (i in nums.indices) if (!used[i]) {
            used[i] = true; path.add(nums[i])
            dfs()
            path.removeAt(path.lastIndex); used[i] = false
        }
    }
    dfs()
    return res
}
```

```ts
function permute(nums: number[]): number[][] {
  const res: number[][] = []
  const path: number[] = []
  const used = new Array(nums.length).fill(false)
  const dfs = () => {
    if (path.length === nums.length) { res.push([...path]); return }
    for (let i = 0; i < nums.length; i++) if (!used[i]) {
      used[i] = true; path.push(nums[i])
      dfs()
      path.pop(); used[i] = false
    }
  }
  dfs()
  return res
}
```

- **复杂度**：时间 `O(n * n!)`，空间 `O(n)`（递归栈不算结果）。
- **面试沟通**：回溯题务必说明“做选择、递归、撤销选择”三步。

### 37. Subsets（#78）

- **题意**：返回所有子集。
- **思路**：每个元素都有“选/不选”两种状态。常见写法是 for-loop 枚举下一层起点。

```kotlin
fun subsets(nums: IntArray): List<List<Int>> {
    val res = mutableListOf<List<Int>>()
    val path = mutableListOf<Int>()
    fun dfs(start: Int) {
        res.add(path.toList())
        for (i in start until nums.size) {
            path.add(nums[i]); dfs(i + 1); path.removeAt(path.lastIndex)
        }
    }
    dfs(0)
    return res
}
```

```ts
function subsets(nums: number[]): number[][] {
  const res: number[][] = []
  const path: number[] = []
  const dfs = (start: number) => {
    res.push([...path])
    for (let i = start; i < nums.length; i++) {
      path.push(nums[i]); dfs(i + 1); path.pop()
    }
  }
  dfs(0)
  return res
}
```

- **复杂度**：时间 `O(n * 2^n)`，空间 `O(n)`。
- **面试沟通**：区别排列与子集：排列关注顺序和 `used`，子集关注起点和去重。

### 38. Combination Sum（#39）

- **题意**：数组元素可重复使用，求和为 target 的组合。
- **思路**：排序后回溯。因为可重复使用当前数字，递归时仍传 `i` 而非 `i+1`。

```kotlin
fun combinationSum(candidates: IntArray, target: Int): List<List<Int>> {
    candidates.sort()
    val res = mutableListOf<List<Int>>()
    val path = mutableListOf<Int>()
    fun dfs(start: Int, remain: Int) {
        if (remain == 0) { res.add(path.toList()); return }
        for (i in start until candidates.size) {
            if (candidates[i] > remain) break
            path.add(candidates[i])
            dfs(i, remain - candidates[i])
            path.removeAt(path.lastIndex)
        }
    }
    dfs(0, target)
    return res
}
```

```ts
function combinationSum(candidates: number[], target: number): number[][] {
  candidates.sort((a, b) => a - b)
  const res: number[][] = []
  const path: number[] = []
  const dfs = (start: number, remain: number) => {
    if (remain === 0) { res.push([...path]); return }
    for (let i = start; i < candidates.length; i++) {
      if (candidates[i] > remain) break
      path.push(candidates[i])
      dfs(i, remain - candidates[i])
      path.pop()
    }
  }
  dfs(0, target)
  return res
}
```

- **复杂度**：指数级，取决于解空间。
- **面试沟通**：要说明“排序的意义是剪枝，而不是为了输出有序”。

### 39. N-Queens（#51）

- **题意**：放置 n 个皇后，互不攻击。
- **思路**：按行放置，列、主对角线、副对角线用集合判冲突。

```kotlin
fun solveNQueens(n: Int): List<List<String>> {
    val res = mutableListOf<List<String>>()
    val cols = HashSet<Int>()
    val diag1 = HashSet<Int>()
    val diag2 = HashSet<Int>()
    val board = Array(n) { CharArray(n) { '.' } }
    fun dfs(r: Int) {
        if (r == n) { res.add(board.map { String(it) }); return }
        for (c in 0 until n) {
            if (c in cols || r - c in diag1 || r + c in diag2) continue
            cols.add(c); diag1.add(r - c); diag2.add(r + c); board[r][c] = 'Q'
            dfs(r + 1)
            cols.remove(c); diag1.remove(r - c); diag2.remove(r + c); board[r][c] = '.'
        }
    }
    dfs(0)
    return res
}
```

```ts
function solveNQueens(n: number): string[][] {
  const res: string[][] = []
  const cols = new Set<number>(), diag1 = new Set<number>(), diag2 = new Set<number>()
  const board = Array.from({ length: n }, () => new Array(n).fill('.'))
  const dfs = (r: number) => {
    if (r === n) { res.push(board.map(row => row.join(''))); return }
    for (let c = 0; c < n; c++) {
      if (cols.has(c) || diag1.has(r - c) || diag2.has(r + c)) continue
      cols.add(c); diag1.add(r - c); diag2.add(r + c); board[r][c] = 'Q'
      dfs(r + 1)
      cols.delete(c); diag1.delete(r - c); diag2.delete(r + c); board[r][c] = '.'
    }
  }
  dfs(0)
  return res
}
```

- **复杂度**：指数级。
- **面试沟通**：重点不是背答案，而是说明“状态约束如何设计”。

### 40. Word Search（#79）

- **题意**：判断网格中是否存在某单词路径。
- **思路**：DFS + 回溯。每次匹配一个字符，并临时标记访问。

```kotlin
fun exist(board: Array<CharArray>, word: String): Boolean {
    val m = board.size
    val n = board[0].size
    fun dfs(i: Int, j: Int, k: Int): Boolean {
        if (k == word.length) return true
        if (i !in 0 until m || j !in 0 until n || board[i][j] != word[k]) return false
        val ch = board[i][j]
        board[i][j] = '#'
        val found = dfs(i + 1, j, k + 1) || dfs(i - 1, j, k + 1) ||
                dfs(i, j + 1, k + 1) || dfs(i, j - 1, k + 1)
        board[i][j] = ch
        return found
    }
    for (i in 0 until m) for (j in 0 until n) if (dfs(i, j, 0)) return true
    return false
}
```

```ts
function exist(board: string[][], word: string): boolean {
  const m = board.length, n = board[0].length
  const dfs = (i: number, j: number, k: number): boolean => {
    if (k === word.length) return true
    if (i < 0 || i >= m || j < 0 || j >= n || board[i][j] !== word[k]) return false
    const ch = board[i][j]
    board[i][j] = '#'
    const found = dfs(i + 1, j, k + 1) || dfs(i - 1, j, k + 1) ||
      dfs(i, j + 1, k + 1) || dfs(i, j - 1, k + 1)
    board[i][j] = ch
    return found
  }
  for (let i = 0; i < m; i++) for (let j = 0; j < n; j++) if (dfs(i, j, 0)) return true
  return false
}
```

- **复杂度**：时间最坏 `O(mn * 4^L)`，空间 `O(L)`。
- **面试沟通**：强调“访问标记必须回溯恢复，否则会污染其他路径”。

---

## 七、堆与优先队列

### 41. Kth Largest Element in an Array（#215）

- **题意**：找数组第 k 大元素。
- **思路**：维护大小为 `k` 的最小堆（min heap），堆顶即第 k 大。

```kotlin
fun findKthLargest(nums: IntArray, k: Int): Int {
    val pq = java.util.PriorityQueue<Int>()
    for (num in nums) {
        pq.offer(num)
        if (pq.size > k) pq.poll()
    }
    return pq.peek()
}
```

```ts
class MinHeap {
  data: number[] = []
  push(x: number) {
    this.data.push(x)
    let i = this.data.length - 1
    while (i > 0) {
      const p = (i - 1) >> 1
      if (this.data[p] <= this.data[i]) break
      ;[this.data[p], this.data[i]] = [this.data[i], this.data[p]]
      i = p
    }
  }
  pop(): number {
    const top = this.data[0], last = this.data.pop()!
    if (this.data.length) {
      this.data[0] = last
      let i = 0
      while (true) {
        let s = i, l = i * 2 + 1, r = i * 2 + 2
        if (l < this.data.length && this.data[l] < this.data[s]) s = l
        if (r < this.data.length && this.data[r] < this.data[s]) s = r
        if (s === i) break
        ;[this.data[i], this.data[s]] = [this.data[s], this.data[i]]
        i = s
      }
    }
    return top
  }
  peek() { return this.data[0] }
  size() { return this.data.length }
}
function findKthLargest(nums: number[], k: number): number {
  const heap = new MinHeap()
  for (const num of nums) {
    heap.push(num)
    if (heap.size() > k) heap.pop()
  }
  return heap.peek()
}
```

- **复杂度**：时间 `O(n log k)`，空间 `O(k)`。
- **面试沟通**：可补充 quickselect 平均 `O(n)`。

### 42. Merge K Sorted Lists（#23）

- **题意**：合并 k 个有序链表。
- **思路**：最小堆保存每个链表当前头节点。

```kotlin
fun mergeKLists(lists: Array<ListNode?>): ListNode? {
    val pq = java.util.PriorityQueue<ListNode>(compareBy { it.`val` })
    for (node in lists) if (node != null) pq.offer(node)
    val dummy = ListNode(0)
    var tail = dummy
    while (pq.isNotEmpty()) {
        val node = pq.poll()
        tail.next = node
        tail = tail.next!!
        node.next?.let { pq.offer(it) }
    }
    return dummy.next
}
```

```ts
function mergeKLists(lists: Array<ListNode | null>): ListNode | null {
  const arr = lists.filter(Boolean) as ListNode[]
  arr.sort((a, b) => a.val - b.val)
  const dummy = new ListNode(0)
  let tail = dummy
  while (arr.length) {
    const node = arr.shift()!
    tail.next = node
    tail = tail.next
    if (node.next) {
      arr.push(node.next)
      arr.sort((a, b) => a.val - b.val)
    }
  }
  return dummy.next
}
```

- **复杂度**：Kotlin 堆解法时间 `O(N log k)`；上面 TS 为便于展示使用数组模拟，面试中建议实现堆以达到 `O(N log k)`。
- **面试沟通**：主动说明“k 路归并（k-way merge）是外排序与日志合并的常见模式”。

### 43. Top K Frequent Elements（#347）

- **题意**：返回出现频率最高的 k 个元素。
- **思路**：先计数，再用大小为 `k` 的最小堆按频率筛选。

```kotlin
fun topKFrequent(nums: IntArray, k: Int): IntArray {
    val count = HashMap<Int, Int>()
    for (num in nums) count[num] = count.getOrDefault(num, 0) + 1
    val pq = java.util.PriorityQueue<Int>(compareBy { count[it]!! })
    for (num in count.keys) {
        pq.offer(num)
        if (pq.size > k) pq.poll()
    }
    return IntArray(k) { pq.poll() }.also { it.reverse() }
}
```

```ts
function topKFrequent(nums: number[], k: number): number[] {
  const count = new Map<number, number>()
  for (const num of nums) count.set(num, (count.get(num) ?? 0) + 1)
  return [...count.entries()]
    .sort((a, b) => b[1] - a[1])
    .slice(0, k)
    .map(([num]) => num)
}
```

- **复杂度**：Kotlin 堆解法时间 `O(n log k)`；TS 示例排序版时间 `O(n log n)`。
- **面试沟通**：如果语言标准库堆不方便写，先给排序版，再说明堆可优化。

### 44. Find Median from Data Stream（#295）

- **题意**：数据流中动态求中位数。
- **思路**：大根堆存较小一半，小根堆存较大一半，保持两边数量平衡。

```kotlin
class MedianFinder {
    private val small = java.util.PriorityQueue<Int>(compareByDescending { it })
    private val large = java.util.PriorityQueue<Int>()
    fun addNum(num: Int) {
        if (small.isEmpty() || num <= small.peek()) small.offer(num) else large.offer(num)
        if (small.size > large.size + 1) large.offer(small.poll())
        if (large.size > small.size) small.offer(large.poll())
    }
    fun findMedian(): Double {
        return if (small.size > large.size) small.peek().toDouble()
        else (small.peek() + large.peek()) / 2.0
    }
}
```

```ts
class MedianFinder {
  private nums: number[] = []
  addNum(num: number): void {
    this.nums.push(num)
    this.nums.sort((a, b) => a - b)
  }
  findMedian(): number {
    const n = this.nums.length
    return n % 2 ? this.nums[n >> 1] : (this.nums[n / 2 - 1] + this.nums[n / 2]) / 2
  }
}
```

- **复杂度**：Kotlin 双堆版插入 `O(log n)`、查询 `O(1)`；TS 示例排序版插入 `O(n log n)`。
- **面试沟通**：一定说明最优设计是双堆；如果现场手写堆太慢，可先讲思路再写简化版。

---

## 八、栈与单调栈

### 45. Daily Temperatures（#739）

- **题意**：每一天距离下一个更高温度还有几天。
- **思路**：单调递减栈保存下标。当前温度更高时，弹出并结算。

```kotlin
fun dailyTemperatures(temperatures: IntArray): IntArray {
    val stack = ArrayDeque<Int>()
    val res = IntArray(temperatures.size)
    for (i in temperatures.indices) {
        while (stack.isNotEmpty() && temperatures[i] > temperatures[stack.last()]) {
            val idx = stack.removeLast()
            res[idx] = i - idx
        }
        stack.addLast(i)
    }
    return res
}
```

```ts
function dailyTemperatures(temperatures: number[]): number[] {
  const stack: number[] = []
  const res = new Array(temperatures.length).fill(0)
  for (let i = 0; i < temperatures.length; i++) {
    while (stack.length && temperatures[i] > temperatures[stack[stack.length - 1]]) {
      const idx = stack.pop()!
      res[idx] = i - idx
    }
    stack.push(i)
  }
  return res
}
```

- **复杂度**：时间 `O(n)`，空间 `O(n)`。
- **面试沟通**：说明栈里存下标而不是温度，是为了直接计算距离。

### 46. Largest Rectangle in Histogram（#84）

- **题意**：柱状图最大矩形面积。
- **思路**：单调递增栈。某柱子出栈时，说明它的左右边界都确定了。

```kotlin
fun largestRectangleArea(heights: IntArray): Int {
    val arr = intArrayOf(0) + heights + intArrayOf(0)
    val stack = ArrayDeque<Int>()
    var ans = 0
    for (i in arr.indices) {
        while (stack.isNotEmpty() && arr[i] < arr[stack.last()]) {
            val h = arr[stack.removeLast()]
            val w = i - stack.last() - 1
            ans = maxOf(ans, h * w)
        }
        stack.addLast(i)
    }
    return ans
}
```

```ts
function largestRectangleArea(heights: number[]): number {
  const arr = [0, ...heights, 0]
  const stack: number[] = []
  let ans = 0
  for (let i = 0; i < arr.length; i++) {
    while (stack.length && arr[i] < arr[stack[stack.length - 1]]) {
      const h = arr[stack.pop()!]
      const w = i - stack[stack.length - 1] - 1
      ans = Math.max(ans, h * w)
    }
    stack.push(i)
  }
  return ans
}
```

- **复杂度**：时间 `O(n)`，空间 `O(n)`。
- **面试沟通**：补零哨兵（sentinel）是常见技巧，可统一处理栈清空逻辑。

### 47. Min Stack（#155）

- **题意**：支持 `push/pop/top/getMin` 且最小值查询 `O(1)`。
- **思路**：一个普通栈，一个同步最小栈。

```kotlin
class MinStack {
    private val stack = ArrayDeque<Int>()
    private val minStack = ArrayDeque<Int>()
    fun push(`val`: Int) {
        stack.addLast(`val`)
        if (minStack.isEmpty() || `val` <= minStack.last()) minStack.addLast(`val`)
    }
    fun pop() {
        val x = stack.removeLast()
        if (x == minStack.last()) minStack.removeLast()
    }
    fun top(): Int = stack.last()
    fun getMin(): Int = minStack.last()
}
```

```ts
class MinStack {
  private stack: number[] = []
  private minStack: number[] = []
  push(val: number): void {
    this.stack.push(val)
    if (!this.minStack.length || val <= this.minStack[this.minStack.length - 1]) this.minStack.push(val)
  }
  pop(): void {
    const x = this.stack.pop()!
    if (x === this.minStack[this.minStack.length - 1]) this.minStack.pop()
  }
  top(): number { return this.stack[this.stack.length - 1] }
  getMin(): number { return this.minStack[this.minStack.length - 1] }
}
```

- **复杂度**：各操作均摊 `O(1)`。
- **面试沟通**：如果值允许重复，最小栈也必须保留重复值。

---

## 九、设计题

### 48. Implement Trie（#208）

- **题意**：实现前缀树（Trie）。
- **思路**：每个节点维护 26 个子节点和单词结束标记。

```kotlin
class Trie {
    private class Node {
        val children = arrayOfNulls<Node>(26)
        var isWord = false
    }
    private val root = Node()
    fun insert(word: String) {
        var cur = root
        for (c in word) {
            val idx = c - 'a'
            if (cur.children[idx] == null) cur.children[idx] = Node()
            cur = cur.children[idx]!!
        }
        cur.isWord = true
    }
    private fun find(prefix: String): Node? {
        var cur = root
        for (c in prefix) {
            cur = cur.children[c - 'a'] ?: return null
        }
        return cur
    }
    fun search(word: String) = find(word)?.isWord == true
    fun startsWith(prefix: String) = find(prefix) != null
}
```

```ts
class TrieNode {
  children = new Map<string, TrieNode>()
  isWord = false
}
class Trie {
  private root = new TrieNode()
  insert(word: string): void {
    let cur = this.root
    for (const ch of word) {
      if (!cur.children.has(ch)) cur.children.set(ch, new TrieNode())
      cur = cur.children.get(ch)!
    }
    cur.isWord = true
  }
  private find(prefix: string): TrieNode | null {
    let cur: TrieNode | null = this.root
    for (const ch of prefix) {
      cur = cur.children.get(ch) ?? null
      if (!cur) return null
    }
    return cur
  }
  search(word: string): boolean { return this.find(word)?.isWord === true }
  startsWith(prefix: string): boolean { return this.find(prefix) !== null }
}
```

- **复杂度**：插入、查询、前缀判断均为 `O(L)`。
- **面试沟通**：可联系搜索建议、敏感词过滤、自动补全等真实业务场景。

### 49. Design HashMap（#706）

- **题意**：不使用内置哈希表，实现基本 `put/get/remove`。
- **思路**：数组桶（bucket）+ 链地址法（separate chaining）。

```kotlin
class MyHashMap {
    private val size = 1009
    private val buckets = Array(size) { mutableListOf<Pair<Int, Int>>() }
    private fun hash(key: Int) = key % size
    fun put(key: Int, value: Int) {
        val bucket = buckets[hash(key)]
        for (i in bucket.indices) if (bucket[i].first == key) {
            bucket[i] = key to value; return
        }
        bucket.add(key to value)
    }
    fun get(key: Int): Int = buckets[hash(key)].firstOrNull { it.first == key }?.second ?: -1
    fun remove(key: Int) { buckets[hash(key)].removeIf { it.first == key } }
}
```

```ts
class MyHashMap {
  private size = 1009
  private buckets: Array<Array<[number, number]>> = Array.from({ length: 1009 }, () => [])
  private hash(key: number): number { return key % this.size }
  put(key: number, value: number): void {
    const bucket = this.buckets[this.hash(key)]
    for (let i = 0; i < bucket.length; i++) {
      if (bucket[i][0] === key) { bucket[i][1] = value; return }
    }
    bucket.push([key, value])
  }
  get(key: number): number {
    const item = this.buckets[this.hash(key)].find(([k]) => k === key)
    return item ? item[1] : -1
  }
  remove(key: number): void {
    const bucket = this.buckets[this.hash(key)]
    const idx = bucket.findIndex(([k]) => k === key)
    if (idx !== -1) bucket.splice(idx, 1)
  }
}
```

- **复杂度**：平均时间 `O(1)`，最坏退化到 `O(n)`。
- **面试沟通**：可补充负载因子（load factor）、扩容、冲突解决策略。

### 50. Implement Queue using Stacks（#232）

- **题意**：用两个栈实现队列。
- **思路**：输入栈负责入队，输出栈负责出队；只有输出栈为空时才搬运。

```kotlin
class MyQueue {
    private val input = ArrayDeque<Int>()
    private val output = ArrayDeque<Int>()
    private fun move() {
        if (output.isEmpty()) while (input.isNotEmpty()) output.addLast(input.removeLast())
    }
    fun push(x: Int) { input.addLast(x) }
    fun pop(): Int { move(); return output.removeLast() }
    fun peek(): Int { move(); return output.last() }
    fun empty(): Boolean = input.isEmpty() && output.isEmpty()
}
```

```ts
class MyQueue {
  private input: number[] = []
  private output: number[] = []
  private move() {
    if (!this.output.length) while (this.input.length) this.output.push(this.input.pop()!)
  }
  push(x: number): void { this.input.push(x) }
  pop(): number { this.move(); return this.output.pop()! }
  peek(): number { this.move(); return this.output[this.output.length - 1] }
  empty(): boolean { return this.input.length === 0 && this.output.length === 0 }
}
```

- **复杂度**：均摊时间 `O(1)`，空间 `O(n)`。
- **面试沟通**：强调“单次搬运可能是 `O(n)`，但每个元素最多搬运一次，所以均摊 `O(1)`”。

---

## 算法面试高频追问

1. **为什么这个解法是最优的？**  
   回答时从信息利用率出发：哈希把查找从线性降为常数；双指针利用有序性；单调栈利用“下一个更大/更小”结构； DP 利用重叠子问题。

2. **如果输入扩大 100 倍怎么办？**  
   看时间复杂度是否还能接受，是否能用空间换时间，是否能做预处理、缓存、并行、外存分块。

3. **代码写完后如何自测？**  
   给出最小输入、空输入、重复值、负数、全相同、严格递增/递减、极值溢出等测试用例。

4. **如何把算法题答得像工程师？**  
   不只给答案，要说明约束、错误输入、可读性、模块拆分、命名和测试策略。

## 本章要点

- 高频算法题本质是模式识别，不是题海战术。
- 数组题重点掌握哈希、双指针、前缀和、单调队列。
- 字符串题重点掌握滑动窗口、栈、中心扩展。
- 链表题重点掌握 dummy head、快慢指针、原地反转。
- 树图题重点掌握 DFS、BFS、拓扑排序、序列化协议。
- 动态规划一定先说状态定义、转移方程、初始化，再考虑优化。
- 回溯题核心是路径、选择列表、结束条件、撤销选择。
- 设计题要先讲数据结构与复杂度，再落代码实现。

## 延伸阅读

- LeetCode 官方题解与 Discuss 高赞解法  
- 《算法导论》关于图、堆、动态规划的基础章节  
- 《剑指 Offer》用于中文面试场景复习  
- NeetCode / labuladong 的模式化刷题总结  
- 用 Kotlin 与 TypeScript 各自手写一遍本章高频模板，形成语言肌肉记忆
