---
layout: post
title: "Two Pointers"
date: 2026-06-03 10:00:00 +0200
categories: dsa algorithms
excerpt: "A practical note about the two pointers pattern in C#."
---

I have been revisiting DSA topics lately and two pointers is one of those patterns that looks simple enough to skip. That is usually a mistake.

The idea is not complicated. We keep two indexes and move them through an array or a string. The useful part is deciding which pointer to move and why that move is safe.

I usually start thinking about two pointers when my first solution has nested loops. If the input has some structure like sorted values or two sides that can be compared there is a good chance one pointer move can remove a lot of unnecessary checks.

## Starting From Both Sides

The most common version starts with one pointer at the beginning and one pointer at the end.

```csharp
var left = 0;
var right = items.Length - 1;

while (left < right)
{
    // use items[left] and items[right]
    // then move left, right, or both
}
```

This shape is useful when the problem has some kind of left/right relationship.

Palindrome is the first example that comes to mind. We do not need to reverse the string. We can compare the first character with the last one and then move inward.

```csharp
public static bool IsPalindrome(string value)
{
    var left = 0;
    var right = value.Length - 1;

    while (left < right)
    {
        if (value[left] != value[right])
        {
            return false;
        }

        left++;
        right--;
    }

    return true;
}
```

For `level` the first check is `l` with `l` and then `e` with `e`. After that we are already in the middle.

The nice part is that we only walk through the string once. There is no extra copy no reverse operation and no second pass.

## Pair Sum In A Sorted Array

This is where the pattern becomes more interesting.

Suppose we have a sorted array and need to know if two numbers add up to a target.

```csharp
var nums = new[] { 3, 5, 7, 11, 18, 21, 29 };
var target = 26;
```

In this case the answer is `true` because `5 + 21 = 26`.

The brute force version checks every pair. It works, but it does too much. Since the array is sorted, we can start from both ends.

If the current sum is too small moving `left` gives us a bigger number.

If the current sum is too large moving `right` gives us a smaller number.

```csharp
public static bool HasPairWithSum(int[] nums, int target)
{
    var left = 0;
    var right = nums.Length - 1;

    while (left < right)
    {
        var sum = nums[left] + nums[right];

        if (sum == target)
        {
            return true;
        }

        if (sum < target)
        {
            left++;
            continue;
        }

        right--;
    }

    return false;
}
```

That is the important detail: every move has a reason. Moving `left` or `right` is not guessing. It is based on the sorted order.

Without sorted input this exact logic would not be valid.

## Moving Through Two Inputs

There is another version where both pointers start at the beginning. Usually this happens when we have two inputs.

For two sorted arrays, the rough shape looks like this:

```csharp
var i = 0;
var j = 0;

while (i < first.Length && j < second.Length)
{
    if (first[i] <= second[j])
    {
        i++;
    }
    else
    {
        j++;
    }
}
```

The exact condition depends on the problem but the idea is the same. Each step moves at least one pointer forward.

A common example is merging two sorted arrays.

```csharp
public static int[] MergeSortedArrays(int[] first, int[] second)
{
    var result = new List<int>(first.Length + second.Length);
    var i = 0;
    var j = 0;

    while (i < first.Length && j < second.Length)
    {
        if (first[i] <= second[j])
        {
            result.Add(first[i]);
            i++;
        }
        else
        {
            result.Add(second[j]);
            j++;
        }
    }

    while (i < first.Length)
    {
        result.Add(first[i]);
        i++;
    }

    while (j < second.Length)
    {
        result.Add(second[j]);
        j++;
    }

    return result.ToArray();
}
```

The last two loops are easy to forget. One array can finish before the other and the remaining values still need to be copied.

## Subsequence

Subsequence is another good example because it does not look like the left/right version.

A substring must be continuous. A subsequence only needs to keep the same order.

For example `git` is a subsequence of `giraffe tea`.

```csharp
public static bool IsSubsequence(string small, string big)
{
    var i = 0;
    var j = 0;

    while (i < small.Length && j < big.Length)
    {
        if (small[i] == big[j])
        {
            i++;
        }

        j++;
    }

    return i == small.Length;
}
```

Here `j` always moves through the bigger string. `i` moves only when we find the next character we need.

If `i` reaches the end of `small` all characters were found in order.

## When I Would Try It

Two pointers is worth trying when:

- the input is sorted
- the problem asks about pairs
- a string can be checked from both sides
- two sorted inputs need to be merged or compared
- the brute force solution has nested loops and feels repetitive

I am keeping these problems as practice:

- [125. Valid Palindrome](https://leetcode.com/problems/valid-palindrome/)
- [167. Two Sum II - Input Array Is Sorted](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted/)
- [88. Merge Sorted Array](https://leetcode.com/problems/merge-sorted-array/)
- [392. Is Subsequence](https://leetcode.com/problems/is-subsequence/)
