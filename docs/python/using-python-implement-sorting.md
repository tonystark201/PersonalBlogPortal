## 🥇 排序算法：让这个世界变得更有秩序

> Everything begins in barrenness and ends in entropy.(世界生于虚无而毁于熵增)

在本文中，我们主要讨论排序算法如何使无序的记录有序。

__排序：根据一个或多个关键字的大小，按升序或降序排列一组记录的操作__

 <div align=center>
 <img width="800" src="/python/img/summary-of-the-sorting-algorithm.png"/>
 </div>

### 🎯 O(n²) sorting algorithms

#### 🍔 Bubble sort

> 如果数组的长度为n，则将进行n-1循环比较。 每个循环都会比较相邻的元素。 如果第一个大于第二个，则交换两个。

```python
def bubble_sort(arr: list):
    """
    1. loop count = len(listing)-1
    2. Each cycle compares and exchanges the internal elements pairwise
    """
    print(f"before:{arr}")
    for i in range(1, len(arr)):
        for j in range(0, len(arr) - i):
            if arr[j] > arr[j + 1]:
                arr[j], arr[j + 1] = arr[j + 1], arr[j]
    print(f"after:{arr}")
```

#### 🍔 Selection Sort

> 首先找到未排序序列中的最小元素，并将其存储在已排序序列的开头。 然后继续从剩余的未排序元素中寻找最小的元素，然后将其放在排序序列的末尾。

```python
def selection_sort(arr: list):

    print(f"before:{arr}")
    for i in range(0, len(arr) - 1):
        minidx = i
        for j in range(i + 1, len(arr)):
            if arr[j] < arr[minidx]:
                minidx = j
        if i != minidx:
            arr[i], arr[minidx] = arr[minidx], arr[i]
    print(f"after:{arr}")
```

#### 🍔 Insert Sort

> 待排序的第一个序列的第一个元素被认为是有序序列，第二个元素到最后一个元素被认为是未排序的序列。 从头到尾扫描未排序的序列，将每个元素插入到有序序列的适当位置。
>
> 有序序列的正确位置是什么？ 
>
> 提取需要插入的第一个元素，并与从后到前有序序列中的元素进行比较。 如果有序序列中的元素大于要插入的元素，则将该元素向后移动一个位置（该位置是提取后第一个需要插入的元素剩余时的空缺）。 直到一个元素（有序序列）小于需要插入的元素，并将元素放在元素（有序序列）的后面。

```python
def insert_sort(arr: list):
    """
    sequence: [1,2,8,4,7,6,9]:
       index:  0,1,2,3,4,5,6

    ordered: 1,2,8
    unordered: 4,7,6,9

    extract 4 (idx=3), and the vacancy 4 leftover will be filled by 8(idx=2)
    each loop, push the larger one in ordered sequence back to vacancy,
    until find a position element(first one of unordered) insert in
    """

    print(f"before:{arr}")
    for i in range(0, len(arr)):
        preidx = i - 1
        current = arr[i]
        while preidx > 0 and arr[preidx] > current:
            arr[preidx + 1] = arr[preidx]
            preidx -= 1
        arr[preidx + 1] = current
    print(f"after:{arr}")
```

### 🎯  O(n log n) sorting algorithms

#### 🍔 Quick Sort

> Quicksort (sometimes called partition-exchange sort) is an efficient sorting algorithm. Developed by British computer scientist Tony Hoare in 1959 and published in 1961, and it is still a commonly used algorithm for sorting. When implemented well, it can be about two or three times faster than its main competitors, merge sort and heapsort. — — Qouted from wiki

为什么快速排序这么快？ [这是答案](https://stackoverflow.com/questions/4289024/why-is-quicksort-faster-in-average-than-others)。

快速排序是分治思想在排序算法中的典型应用。 本质上，快速排序应该被认为是一种基于冒泡排序的递归分治法。选择一个好的主元至关重要，因为它会影响快速排序算法的性能。 一般来说，随机枢轴将提供 O(nlogn) 和最坏情况 O(n²)。 最坏情况的概率非常小，这意味着随机选择枢轴就足够了。

计算步骤如下：

1. 从序列中选择一个元素并将其称为枢轴。
2. 重新排序。所有小于pivot的元素放在pivot前面，所有大于pivot的元素放在pivot后面（相同的数字可以到任何一边）。 分区退出后，枢轴处于序列的正确位置。 这称为分区操作。
3. 递归地对小于主元的元素的子序列和大于主元的元素的子序列进行排序。

```python
def quick_sort(arr: list, low, high):
    """
    choosing a good pivot is critical
    partition1: first element as pivot
    partition2: last element as pivot
    partition3: random pivot
    partition4: first element as pivot and use two pointer
    """
    if len(arr) == 1:
        return arr
    if low < high:
        pi = partition1(arr, low, high)
        quick_sort(arr, low, pi - 1)
        quick_sort(arr, pi + 1, high)
    return arr


def partition1(arr, low, high):
    """
    pivot is the first element
    single pointer to smaller
    """
    pivot = arr[low]
    i = low + 1
    for j in range(low + 1, high + 1):
        if arr[j] <= pivot:
            arr[i], arr[j] = arr[j], arr[i]
            i = i + 1
    arr[low], arr[i - 1] = arr[i - 1], arr[low]
    return i - 1


def partition2(arr, low, high):
    """
    pivot the last element,
    single pointer to smaller
    """
    i = low - 1
    pivot = arr[high]
    for j in range(low, high):
        if arr[j] <= pivot:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]
    arr[i + 1], arr[high] = arr[high], arr[i + 1]
    return i + 1


def partition3(arr, low, high):
    """
    first or last element as pivot,quick sort perform
    badly on already sorted (or almost sorted) arrays.

    random pivot
    single pointer to larger
    """
    idx = random.randrange(low, high)
    arr[low], arr[idx] = arr[idx], arr[low]  # change pivot to first element
    return partition1(arr, low, high)


def partition4(arr, low, high):
    """
    pivot is the first element
    double pointers, one point to smaller,and one point to larger
    """
    p_small, p_large = low + 1, high
    pivot = arr[low]
    while True:
        while p_small <= p_large and arr[p_large] >= pivot:
            p_large -= 1
        while p_small <= p_large and arr[p_small] <= pivot:
            p_small += 1

        if p_small <= p_large:
            arr[p_small], arr[p_large] = arr[p_large], arr[p_small]
        else:
            break
    arr[low], arr[p_large] = arr[p_large], arr[low]
    return p_large
```

#### 🍔 Merged Sort

> 归并排序是一种基于归并操作的有效排序算法。 与快速排序一样，该算法也是分治思想的典型应用。

```python
def merge_sort(arr: list):
    """
    recursion from head to tail and iteration from tail to head
    """
    if len(arr) < 2:
        return arr
    middle = len(arr) // 2
    left, right = arr[0:middle], arr[middle:]
    return merge(merge_sort(left), merge_sort(right))


def merge(left, right):
    result = []
    while left and right:
        if left[0] <= right[0]:
            result.append(left.pop(0))
        else:
            result.append(right.pop(0))
    while left:
        result.append(left.pop(0))
    while right:
        result.append(right.pop(0))
    return result
```

#### 🍔 Heap Sort

> 堆排序是指利用堆的数据结构设计的一种排序算法。 堆是一棵除底层外完全填满的二叉树，子节点的值总是小于（或大于）其父节点。

```python
def heapify(arr, n, i):
    root = i
    l = 2 * i + 1
    r = 2 * i + 2
    if l < n and arr[i] < arr[l]:
        root = l
    if r < n and arr[root] < arr[r]:
        root = r
    if root != i:
        arr[root], arr[i] = arr[i], arr[root]
        heapify(arr, n, root)


def heap_sort(arr):
    n = len(arr)
    for i in range(n // 2, -1, -1):
        heapify(arr, n, i)

    for i in range(n - 1, 0, -1):
        arr[i], arr[0] = arr[0], arr[i]
        heapify(arr, i, 0)

    return arr
```

### 🎯  O(n) Algorithms

这三种排序算法都使用了桶，但是桶的使用有明显的区别。

#### 🍔 Counting Sort

> 计数排序的核心是将输入的数据值转换成键值存储在附加空间中。 作为一种线性时间复杂度的排序，计数排序要求输入数据必须是具有一定范围的整数。

```python
def counting_sort(arr):
    """use array as bucket"""
    counts = [0] * (max(arr) + 1)
    for num in arr:  # Counting - O(n)
        counts[num] += 1

    result = []
    for num, count in enumerate(counts):
        if count == 0:
            continue
        result.append([num] * count)
    return sum(result, [])
```

#### 🍔 Bucket Sort

> 桶排序是计数排序的升级版。 为了让桶排序更高效，我们需要做两件事，第一是尽量增加桶的数量，第二是使用的映射函数可以将输入的 N 个数据平均分配到 K 个桶中。
>
> 同时，对于桶中元素的排序，排序算法的选择对性能至关重要。

```python
def bucket_sort(arr):
    """
    sorted is a built-in sorting algorithm in python,
    which can  be replaced by a stable sorting algorithm
    """

    max_, min_ = max(arr), min(arr)
    stride = (max_ - min_) / len(arr)
    buckets = [[] for i in range(len(arr) + 1)]
    for num in arr:
        buckets[int((num - min_) // stride)].append(num)
    return sum([sorted(bucket) for bucket in buckets], [])
```

#### 🍔 Radix Sort

> 基数排序是一种非比较整数排序算法。 其原理是将整数按位数切割成不同的数，然后根据每个位数进行比较。

```python
def radix_sort(arr):
    digit = 0

    def find_max_digit(max_):
        digit = 1
        while True:
            max_ = max_ / 10
            if max_ > 1:
                digit += 1
            else:
                break
        return digit

    def single_swap(arr, digit):
        buckets = [[] for _ in range(10)]
        for num in arr:  # put in bucket
            t = int((num / 10 ** digit) % 10)
            buckets[t].append(num)
        return sum(buckets, [])  # pop from bucket

    max_digit = find_max_digit(max(arr))
    while digit < max_digit:
        arr = single_swap(arr, digit)
        digit = digit + 1
    return arr
```

### 🎯  Summary

排序算法是每个计算机初学者都必须掌握的基础，对于时间复杂度较低的排序算法，都使用到了桶的概念，这是一种空间换时间的思路，这也是计算机工程中比较普遍的思路之一。时间和空间具有对等效应，也就是说牺牲空间，势必会提升时间效率，牺牲时间势必会降低对空间的消耗。

从代码的角度来看，比较容易理解的是冒泡排序、选择排序、插入排序，因为这三种算法都是基于嵌套循环来实现，时间复杂度较高。相对来说，桶排序和计数排序也比较容易理解，把桶作为排序的辅助工具，简化了排序的流程。由于合并排序和堆排序等都使用了迭代，理解起来并不太容易。

