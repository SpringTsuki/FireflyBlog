---
title: LeetCode 第一周
published: 2026-09-21
description: 是时候开始写算法题了！
tags: [Python, LeetCode, 算法]
category: Python
---

## 9.21 Day 1

### Q1：两数之和

#### 题目

给定一个整数数组 `nums` 和一个整数目标值 `target`，请在该数组中找出和为目标值 `target` 的两个整数，并返回它们的数组下标。

你可以假设每种输入只会对应一个答案，并且你不能使用两次相同的元素。

示例 1：

> 输入：nums = [2,7,11,15], target = 9
> 输出：[0,1]
> 解释：因为 nums[0] + nums[1] == 9 ，返回 [0, 1] 。

示例 2：

> 输入：nums = [3,2,4], target = 6
> 输出：[1,2]

示例 3：

> 输入：nums = [3,3], target = 6
> 输出：[0,1]

#### 我的第一反应

一开始我混淆了“元素值”和“数组下标”：`for num in nums` 取得的是元素本身，而不是下标。如果需要根据位置访问元素，可以使用 `range(len(nums))`；如果需要同时取得下标和值，可以使用 `enumerate(nums)`。

#### 暴力解法

首先是用了暴力穷举法：时间复杂度：O(n²)，最坏情况下需要比较大约 n(n-1)/2 次。

额外空间复杂度：O(1)，result 只保存固定的两个下标，不随数组长度增长。

```python
from typing import List


class Solution:
    def twoSum(self, nums: List[int], target: int) -> List[int]:
        """
        枚举数组中所有不同的元素组合，找到和为 target 的两个元素。

        时间复杂度：O(n²)
        空间复杂度：O(1)
        """

        # 第一个下标的范围是 0 到 len(nums) - 2
        for first_index in range(len(nums) - 1):

            # 从 first_index 的后一位开始，避免：
            # 1. 重复使用同一个元素
            # 2. 重复检查相同的元素组合
            for second_index in range(first_index + 1, len(nums)):

                # 如果两个元素之和等于目标值，则返回它们的下标
                if nums[first_index] + nums[second_index] == target:
                    return [first_index, second_index]

        # 根据题目条件，每组输入一定存在答案。
        # 保留空列表返回值，可以使函数在其他情况下也有明确返回值。
        return []
```

#### 本地练习版本

```python
def two_sum(nums, target):
    for first_index in range(len(nums) - 1):
        for second_index in range(first_index + 1, len(nums)):
            if nums[first_index] + nums[second_index] == target:
                return [first_index, second_index]

    return []


nums = [2, 7, 11, 15]
target = 9
print(two_sum(nums, target))
```

这里使用 `return` 直接结束函数。如果只写 `break`，它只会退出内层循环，外层循环仍会继续执行。

#### 哈希表优化

更高效的做法是使用字典（哈希表）。对于当前数字 `num`，先计算还需要哪个数字才能得到目标值，再检查该数字是否已经出现过。

```python
from typing import List


class Solution:
    def twoSum(self, nums: List[int], target: int) -> List[int]:
        """
        使用哈希表记录已经遍历过的数字及其下标。

        对于当前数字 num，计算：
            complement = target - num

        如果 complement 已经出现过，说明：
            complement + num == target

        时间复杂度：O(n)
        空间复杂度：O(n)
        """

        # 用于保存“数字: 下标”
        # 例如 {2: 0} 表示数字 2 出现在下标 0
        num_to_index = {}

        # enumerate() 同时提供当前元素的下标和值
        for current_index, current_num in enumerate(nums):

            # 计算与当前数字相加后能够得到 target 的补数
            complement = target - current_num

            # 如果补数之前出现过，直接返回补数和当前数字的下标
            if complement in num_to_index:
                return [num_to_index[complement], current_index]

            # 当前没有找到答案，将当前数字及其下标保存起来，
            # 供后面的元素查询
            num_to_index[current_num] = current_index

        # 根据题目条件，每组输入一定存在答案
        return []
```

#### 复杂度

| 解法 | 时间复杂度 | 额外空间复杂度 |
| --- | --- | --- |
| 暴力枚举 | `O(n²)` | `O(1)` |
| 哈希表 | `O(n)` | `O(n)` |

#### 容易踩坑的地方

- `for num in nums` 得到的是元素值，而不是下标。
- 第二个下标应从第一个下标的后一位开始，避免重复使用同一个元素。
- `break` 只退出当前这一层循环；在函数中找到答案后可以直接 `return`。

### Q2：两数相加

#### 题目

给定两个非空链表，分别表示两个非负整数。数字按照逆序存储，每个节点只能保存一位数字。将两个数相加，并以相同形式返回表示结果的链表。

你可以假设除了数字 0 之外，这两个数都不会以 0 开头。

```
Eg:
2->4->3
5->6->4

Return:
7->0->8
```

#### 链表基础

在开始之前，先学习一下链表的基本操作。Python 标准库没有直接提供这种算法题中使用的单链表节点，通常需要自行定义：

```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val      # 当前节点保存的值
        self.next = next    # 指向下一个节点
```

学习链表时，可以先记住：

```python
current.val # 表示当前节点的值。
current.next # 表示下一个节点。
current = current.next # 表示移动到下一个节点。
```

#### 实现思路

计算当前总值：链表一的当前节点值 + 链表二的当前节点值 + 上一位的进位。进位初始为 `0`。

然后对计算总值进行处理：

使用 `total % 10` 得到当前位，并将它保存到新的结果节点中。

使用 `total // 10` 得到进位，并在下一轮计算中继续使用。

只要任意一条链表还有节点，或者仍然存在进位，就要继续计算。`dummy` 是方便构建结果的虚拟头节点，不属于最终答案，因此最后返回 `dummy.next`，也就是真正的结果链表头节点。

```python
# class ListNode:
#     def __init__(self, val=0, next=None):
#         self.val = val
#         self.next = next

class Solution:
    def addTwoNumbers(self, l1: ListNode | None, l2: ListNode | None) -> ListNode | None:
        current1 = l1
        current2 = l2

        dummy = ListNode()
        tail = dummy

        carry = 0

        while current1 is not None or current2 is not None or carry != 0:
            # 如果链表已经结束，将该位置看作0
            value1 = current1.val if current1 is not None else 0
            value2 = current2.val if current2 is not None else 0

            # 计算当前位数的数字（包括进位）
            total = value1 + value2 + carry

            # 取得当前位。总值大于等于 10 时，只保留个位
            current_digit = total % 10

            # 得到进位数字
            carry = total // 10

            # 将最终结果插入链表，指针进入下一位
            tail.next = ListNode(current_digit)
            tail = tail.next

            # 输入链表尚未结束时，才移动对应指针
            if current1 is not None:
                current1 = current1.next

            if current2 is not None:
                current2 = current2.next

        # 得到结果链表
        return dummy.next

        # 以下为本地测试的打印环境，无需将其复制到 leetcode 中
        # result_head = dummy.next

        # current = result_head

        # while current is not None:
        #     print(current.val)
        #     current = current.next
```

#### 复杂度

- 时间复杂度：`O(max(m, n))`，其中 `m` 和 `n` 分别是两条链表的长度。
- 额外辅助空间复杂度：`O(1)`。
- 结果链表本身需要 `O(max(m, n))` 的空间，这是返回结果必须占用的空间。

#### 容易踩坑的地方

- 链表是逆序存储的，头节点代表个位。
- 判断是否进位时应是“大于等于 10”，不是“大于 10”。
- 不能一直覆盖同一个结果节点；每一位都要创建并连接一个新节点。
- `dummy = ListNode()` 是创建节点对象，`dummy = ListNode` 保存的只是类本身。
- 两条链表长度可能不同，缺失的那一位应当视为 `0`。
- 两条链表都结束后仍可能存在最后一个进位。

### Q3：无重复字符的最长子串

#### 题目

给定一个字符串 `s`，找出其中不含重复字符的最长子串，并返回它的长度。

Eg1:

> 输入: s = "abcabcbb"
> 输出: 3
> 解释: 因为无重复字符的最长子串是 "abc"，所以其长度为 3。注意 "bca" 和 "cab" 也是正确答案。

Eg2:

> 输入: s = "bbbbb"
> 输出: 1
> 解释: 因为无重复字符的最长子串是 "b"，所以其长度为 1。

#### 字符串基础

在开始之前，首先学习一下字符串的处理吧！

Python 字符串本身就是一个可以遍历、按下标访问的字符序列：

```python
s = "abcabcbb"
```

1. 按下标访问字符

```python
s[0]  # "a"
s[1]  # "b"
s[2]  # "c"
```

2. 遍历字符串中的字符

```python
for char in s:
    print(char)
```

3. 同时获取下标和字符

可以使用 `enumerate()`：

```python
for index, char in enumerate(s):
    print(index, char)
```

然而，这题需要留意的是，**遇到重复字符时不能直接归零**。

假设字符串是：

```text
a b c a d
```

读取前三个字符时，当前没有重复字符的部分是：

```text
a b c
```

当读取到第二个 `a` 时，出现了重复字符。

如果直接将当前长度归零，就会丢掉仍然有效的部分：

```text
b c a
```

虽然两个 `a` 不能同时存在，但移除左边旧的 `a` 后：

```text
b c a
```

仍然是一个连续且没有重复字符的子串。

所以遇到重复字符时，不应该直接清空全部内容，而应该：

> 从当前子串的左侧逐步移除字符，直到重复字符不再重复。

#### 滑动窗口

这里可以使用滑动窗口：用两个位置表示当前正在观察的连续范围。

```text
a b c a b c b b
↑     ↑
左    右
```

这个连续范围通常叫作“窗口”。右边界负责加入新字符；出现重复时，左边界不断向右移动，直到窗口重新满足“没有重复字符”这一条件。

以 `abcabcbb` 的前四个字符为例：

| `right` | 新字符 | 是否重复 | `left` 的变化 | 当前窗口 | 历史最大长度 |
| ---: | :---: | :---: | :---: | :---: | ---: |
| 0 | `a` | 否 | 0 | `a` | 1 |
| 1 | `b` | 否 | 0 | `ab` | 2 |
| 2 | `c` | 否 | 0 | `abc` | 3 |
| 3 | `a` | 是 | 0 → 1 | `bca` | 3 |

核心不变量是：`left` 到 `right` 之间始终是一个没有重复字符的连续子串。

#### 最终实现

```python
def study(s):
    characters = set()
    left = 0
    max_length = 0

    for right in range(len(s)):

        # 如果右侧的新字符已经在窗口中出现重复
        # 就不断从窗口左侧移除字符
        while s[right] in characters:
            characters.remove(s[left])
            left += 1

        # 重复已经消失，将当前字符加入窗口
        characters.add(s[right])

        # 根据 left 和 right 计算当前窗口长度
        current_length = right - left + 1

        # 比较当前长度和历史最大长度
        max_length = max(max_length, current_length)

    return max_length
```

#### 复杂度

- 时间复杂度：`O(n)`。虽然代码中存在 `while`，但每个字符最多被加入和移出集合各一次。
- 空间复杂度：`O(min(n, 字符集大小))`，集合只保存当前窗口内不重复的字符。

#### 容易踩坑的地方

- 子串必须连续，不能重新排列字符。
- 字符串本身可以遍历，不必先转换成列表。
- 遇到重复字符时不能直接把当前长度归零。
- `current_length = right - left + 1` 中的 `+ 1` 不能遗漏，因为左右边界都包含在窗口内。
- `current_length = +1` 是把变量赋值为 `1`；累加应写成 `+= 1`。
- 集合与字符不能直接比较，应使用 `s[right] in characters` 判断字符是否存在。

#### 建议测试用例

```python
""
"b"
"bbbbb"
"abcabcbb"
"abba"
"pwwkew"
```

其中 `abba` 很适合检查左边界是否只向右移动，以及重复字符是否被正确移出窗口。

## 今日复盘

今天的三道题分别接触了三种常见思路：

1. 使用哈希表减少重复查找。
2. 使用虚拟头节点构建新链表，并正确处理进位。
3. 使用滑动窗口维护一个连续且满足条件的范围。

比记住最终代码更重要的是理解每段循环正在维护什么条件。以后遇到新题时，可以先写出最直接的解法，再观察其中哪些工作被重复执行，最后尝试用合适的数据结构或算法模式进行优化。
