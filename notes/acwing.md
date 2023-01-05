# 各种排序

基于比较的排序算法，时间复杂度具有下限 $O(n\log n)$

>   可证明，但我不会

*   交换排序:
    *   冒泡排序
    *   快速排序
*   插入排序：
    *   单纯的插入排序
    *   希尔排序
*   选择排序：
    *   单纯的选择排序
    *   堆排序
*   归并排序：
    *   二路归并
    *   多路归并

>   其实写的最多的还是快排和归并

其他的排序算法，可以将时间复杂度降到更低，比如桶排序可以在 $O(n)$ 的时间内完成排序

>   其他的还有：计数排序、基数排序

## 什么是稳定

之前看帖子的时候就经常看到 xxx 排序算法稳定，xxx 排序算法不稳定

如果原始序列中存在两个相等的数 a 和 b，如果在排序结束后 a 和 b 的相对顺序一定可以保持不变，那么这个排序算法就是稳定的 

要说明的是，说一个排序算法是稳定的，是说可以通过一定的手法实现稳定性，不是说瞎 jb 排序就能稳定；说一个算法是不稳定的，是说算法本身导致无论什么手法都可能导致不稳定

## 插入排序

思想是从前向后遍历数组，认为整个数组头部是有序的，每次对于枚举到的位置，将其插入到前面有序的序列中，此时就需要遍历前面所有的位置，为新的元素找到合适的位置

所以就是遍历所有的元素，并为所有元素找到合适的位置，因此排序时间复杂度为 $O(n^2)$

然而在序列长度比较短的时候，使用插入排序其实更方便，主要是写起来比较简单，且如果序列比较短，$O(n^2)$ 和 $O(n\log n)$ 也就没差那么多了

插入排序是稳定，前提是从后向前找位置的时候，仅在枚举到的位置比当前位置更大的时候才将其向后移动

```java
public static void sort(int[] arr) {
    for (int i = 1; i < arr.length; i++) {
        int num = arr[i];
        int idx = i - 1;
        while (idx >= 0 && arr[idx] > num) arr[idx + 1] = arr[idx--];
        arr[++idx] = num;
    }
}
```

### 希尔排序

第一个突破 $O(n^2)$ 的排序算法

也是一种插入排序，更快了，但是不稳定了

先分组，数组下标每隔 gap 分为一组，一组内部进行插入排序，然后不断缩小 gap，当 gap 缩小为 1 时将退化为一般化的插入排序

具体的 gap 的取值，可以看看算法4

## 选择排序

每轮遍历的时候找到当前最小(或者最大)的数，放在序列的最前面(或者最后面)

时间复杂度还是 $O(n^2)$

```java
public static void sort(int[] arr) {
    for (int i = 0; i < arr.length - 1; i++) {
        int min = i;
        for (int j = i + 1; j < arr.length; j++) {
            if (arr[j] < arr[min]) min = j;
        }
        int tmp = arr[i];
        arr[i] = arr[min];
        arr[min] = tmp;
    }
}
```

### 堆排序

满二叉树：二叉树每层的节点个数都达到了最大值

完全二叉树：除了最后一层外，每一层节点数都达到了最大值，最后一层的节点从左向右排列

堆其实就是数组实现的完全二叉树

利用堆找到当前序列中最小(或者最大)的元素，时间复杂度为 $O(n\log n)$

>   思想和选择排序是一样的，都是找到当前最大或最小的元素

堆排序不稳定

堆包含了 push_up 和 push_down 两个操作，下面考虑一个大顶堆

同时考虑一个堆的下标从 1 开始，即如果当前节点为 idx，那么左子节点为 idx << 1，右子节点为 (idx << 1) + 1

*   push_down: 如果当前节点的两个子节点中存在一个比当前节点大的节点，那么需要让当前节点和子节点交换(保证了当前节点的堆的有序性)；然后还需要从当前节点开始向子节点递归的修改，即 push_down 保证整个堆的大顶堆特性

    ```java
    public void push_down(int[] heap, int idx, int size) {
        int t = idx;
        int left = idx << 1;
        int right = (idx << 1) + 1;
        if (left <= size && heap[left] > heap[t]) t = left;
        if (right <= size && heap[right] > heap[t]) t = right;
        if (t != idx) {
            int tmp = heap[idx];
            heap[idx] = heap[t];
            heap[t] = tmp;
            push_down(heap, t, size);
        }
    }
    ```

*   push_up: 如果当前节点的值比父节点更大，那么需要交换当前节点和父节点(保证当前节点大顶堆的性质)，同时还需要让父节点递归的进行更新(保证整个堆的大顶堆特性)

    ```java
    public void push_up(int[] heap, int idx) {
        int parent = idx >> 1;
        if (parent >= 1 && heap[parent] < idx) {
            int tmp = heap[idx];
            heap[idx] = heap[parent];
            heap[parent] = tmp;
            push_up(heap, parent);
        }
    }
    ```

对于堆的各种修改都可以通过这两个操作完成，如果如果向堆中添加一个元素，就是在数组的下一个有效下标处添加元素，并进行 push_up 操作、如果对堆中的某个元素进行更新操作，那么就同时调用 push_down 和 push_up(事实证明，尽管调用了两个函数，但最终最多仅会进行 push_up 和 push_down 两个操作中的一个)...

而至于堆排序，其实只会用到 push_down 操作，借助算法 4 中的思想，即维护一个大顶堆和一个 size，每次交换堆顶和最后一个元素，然后让 size 自减

>   这里暗含了有效下标从 0 开始，即 left = (idx << 1) + 1; right = (idx << 1) + 2

```java
private void heapSort(int[] arr) {
    int size = arr.length - 1;
    for (int i = (size - 1) >> 1; i >= 0; i--) push_down(arr, i, size);
    while  (size > 1) {
        int tmp = arr[0];
        arr[0] = arr[size];
        arr[size] = tmp;
        size--;
    }
}

private void push_down(int[] arr, int idx, int size) {
    int t = idx;
    int left = (t << 1) + 1;
    int right = (t << 1) + 2;
    if (left <= size && arr[left] > arr[t]) t = left;
    if (right <= size && arr[right] > arr[t]) t = right;
    if (t != idx) {
        int tmp = arr[idx];
        arr[idx] = arr[t];
        arr[t] = tmp;
        push_down(arr, t, size);
    }
}
```

最开始，因为数组不是有效的大顶堆，需要先进行构建，构建的方式是从数组中第一个具有子节点的节点开始，下滤，下滤的次数为 $\frac{n}{2}$，且每次下滤的时间复杂度为 $O(\log n)$，因此初始构建堆的时间复杂度为 $O(n\log n)$

然后每轮次，交换堆顶和堆尾部的元素，并修改堆的 size，并针对堆顶下滤，这样每轮的时间开销包括了交换和下滤堆顶，因此每轮的时间复杂度为 $O(\log n)$，一共需要交换 n - 1 次，因此 while 循环的终止条件为 size > 1(第一次交换 size = n, 第 n - 1 次交换 size = n - (n - 1 - 1) = 2)

## 冒泡排序

梦开始的地方

思想是，从头向后遍历数组，如果前一个元素比后一个元素更大就进行交换，那么一次遍历过后，数组中最后一个元素一定是整个序列中的最大的那个；然后重复上面的步骤 n 次就可以排好所有位置

冒泡排序是稳定的：前提是遇到了两个相同的数，就不进行交换

```java
public static void sort(int[] arr) {
    for (int i = arr.length - 1; i > 0; i--) {
        for (int j = 0; j < i; j++) {
            if (arr[j] > arr[j + 1]) {
                int tmp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = tmp;
            }
        }
    }
}
```

优化，如果扫一轮发现没有进行过交换就提前结束

```java
public static void sort(int[] arr) {
    for (int i = arr.length - 1; i > 0; i--) {
        boolean changed = false;
        for (int j = 0; j < i; j++) {
            if (arr[j] > arr[j + 1]) {
                int tmp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = tmp;
                changed = true;
            }
        }
        if (!changed) return;
    }
}
```

>   数组大部分是有序状态了，那么扫几轮就可以提前结束了

## 快速排序

主要思想是找分割点，平均时间复杂度为 $O(n\log)$，而在最坏的情况下时间开销将达到 $O(n^2)$

这里所谓最坏的情况，即为每次选择的数，都不能将整个数组分为两部分，比如如果每次选择的分割点都是当前区间的最后一个字符，而如果原始数组本身已经是有序的了，那么此时快速排序的时间复杂度将达到 $O(n^2)$

所以具体的事件复杂度和选取策略有关，如果选取分割点的手段足够随机，就很难达到 $O(n^2)$ 的运行时间

>   更为具体的，每种随机选择都对应一种最坏的情况，每次选择到最坏的情况的概率为 $\frac{1}{n!}$

```java
public void quick_sort(int[] arr, int l, int r) {
    if (l >= r) return;
    int i = l - 1;
    int j = r + 1;
    int p = arr[l + ((r - l) >> 1)];
    while (i < j) {
        do i++; while(arr[i] < p);
        do j--; while (arr[j] > p);
        if (i < j) {
            int tmp = arr[i];
            arr[i] = arr[j];
            arr[j] = arr[i];
        }
    }
    quick_sort(arr, l, j);
    quick_sort(arr, j + 1, r);
}
```

要注意的是一轮快排结束后 left 到 j 是有序的，j + 1 到 right 是有序的，但并不是说 arr[j] 这个位置恰好就是整个数组中第 j 个元素

## 归并排序

分治的思想，先分开，然后合并的时候认为两部分已经分别排好序了，合并需要的时间为 $O(n + m)$

因为每次从中间分开数组，所以递归深度为 $\log n$，每层递归需要处理的数字个数为 n 个所以时间复杂度为 $O(n\log n)$

归并排序的典型应用：[求逆序对](./一些算法.md#求逆序对)：暴力二维遍历时间复杂度为 $O(n^2)$，在归并排序的合并阶段，一边进行排序，一边进行统计

对于数组进行归并排序是很容易的，但是对于链表而言，就有点麻烦了：[对链表进行归并排序](./一些算法.md#对链表进行归并排序)

## 计数排序

其实就是桶排序，主要应对的是，输入数据的个数较多，且数据的范围相对小($10^5$ 以内没什么问题)，实现 $O(n)$ 级别的排序

核心思路就是开一个大数组，然后将对应的位置加入桶中，输出的时候就是遍历桶

```java
/**
 * @param size 表示输入数据的范围，最大可以给到 10^5，如果再大可能会 MLE
 */
public void bucket_sort(int[] arr, int size) {
    int[] bucket = new int[size + 1];
    for (int num : arr) bucket[num]++;
    int idx = 0;
    for (int i = 0; i < arr.length; i++) {
        while (idx <= size && bucket[idx] == 0) idx++;
        arr[i] = idx;
        bucket[idx]--;
    }
}
```

## 基数排序

按位排序，先按低位排序，然后逐渐遍历到高位，具体的过程可以上网看，不想写了

# 二分

题目是否能不能二分，关键点在性质的二段性，即整个结果集可以分为两部分，一部分满足性质，另一部分不满足性质，一般的，需要求解的就是满足性质的边界点(比如最大的最小值问题)

acwing 中的二分模板和我经常使用的那个是类似的，分为两类，现在考虑一个单调的数组，可能存在重复元素，要在 nums 中找到 target 的左右边界：

*   左边界：

    ```java
    /**
     * 找到 target 的左边界
     * @param arr 数组本身
     * @param left 数组左边界
     * @param right 数组右边界
     */
    public int leftBi(int[] arr, int left, int right, int target) {
        while (left < right) {
            int mid = left + ((right - left) >> 1);
            if (arr[mid] < target) left = mid + 1;
            else right = mid;
        }
        if (arr[left] == target) return left;
        return -1;
    }
    ```

*   右边界：

    ```java
    /**
     * 找到 target 的右边界
     * @param arr 数组本身
     * @param left 数组左边界
     * @param right 数组右边界
     */
    public int rightBi(int[] arr, int left, int right, int target) {
        while (left < right) {
            int mid = left + ((right - left + 1) >> 1);
            if (arr[mid] > target) right = mid - 1;
            else left = mid;
        }
        if (arr[left] == target) return left;
        return -1;
    }
    ```

## 浮点数二分

浮点数的二分不需要考虑边界的问题，所以其实是比正数二分简单一点的，不过一般浮点数的二分存在精度的问题，不一定必须返回一个精确解，一般认为在误差范围内就可以返回了

拿一个题举例：[790. 数的三次方根 | AcWing](https://www.acwing.com/problem/content/description/792/)，就是求解一个数的立方根

```c++
#include <iostream>

using namespace std;

int main() {
    double x;
    cin >> x;
    double left = -100;
    double right = 100;
    while (right - left > 1e-8) {
        double mid = (left + right) / 3;
        double mul = mid * mid * mid;
        if (mul > x) right = mid;
        else if (mul < x) left = mid;
        else printf("%.6f", x);
    }
    printf("%.6f", left);
    return 0;
}
```

一般的，如果需要保留的小数个数为小数点后 6 位, 那么 while 的终止条件需要取到 $10^{-8}$，即精确度比小数个数多两个数量级

# 高精度问题

大数之间的运算，使用数组表示数字的每个位进行运算，将大数的每一位映射到数组的某个位置上

习惯上，更倾向于让数组的小索引保存数字的低位，即数组下标为 0 的位置，保存大数的个位

# 前缀和

现在重新考虑一下前缀和，原数组 arr，前缀和数组 preSum

考虑数组的下标从 1 开始，那么有：preSum[i] = arr[1] + arr[2] + ... + arr[i]

>   在 acwing 这种评测网站中，需要手动处理输入输出，因此，原数组和差分数组都可以在读取的时候就强制使其下标从 1 开始
>
>   下标从 1 开始，方便边界处理

在求前缀和数组时，有：preSum[i] = preSum[i - 1] + arr[i]

>   由于在 c++ 中，新建的数组需要手动初始化，因此，尽管定义了 preSum 数组下标从 1 开始，还是需要手动设置 preSum[0] = 0

当考虑一个区间 [L, R] 内的前缀和时，有 sum[L, R] = preSum[R] - preSum[L - 1]

>   其中 L，R 均大于等于 1

这一点从定义上就能看出来

## 二维前缀和

其实关键还得看二维的

类似的，考虑原矩阵下标从 1 开始，即 x 从 1 -> row；y 从 1 -> col

计算前缀和：preSum\[i][j] = preSum\[i - 1][j] + preSum\[i][j - 1] - preSum\[i - 1][j - 1] +  arr\[i][j]

计算某一个子矩阵(x1,y1) -> (x2,y2)的大小：

area = preSum\[x2][y2] - preSum\[x1 - 1][y2] - preSum\[x2][y1 - 1] + preSum\[x1 - 1][y1 - 1]

# 差分

原数组 arr，差分数组为 diff，那么可以将原数组 arr 看成差分数组 diff 的前缀和数组

如果考虑数组下标从 1 开始，那么有：arr[i] = diff[1] + diff[2] + ... + diff[i]

使用差分数组，优化的其实是区间更新的操作，比如对于原数组，考虑更新区间 [L, R]，增量为 C

那么等效于让 diff[L] += C; diff[R + 1] -= C

简单证明一下：

考虑原数组分为三个区间：[1, L - 1]、[L, R]、[R + 1, N]

首先对于第一个区间，因为运算不涉及到 diff[C]，因此通过前缀和的方式获取到 arr[i] 的时候，和没有进行 +C 操作一致

对于第二个区间，因为区间包含了 L，通过前缀和的方式计算 arr[i] 的时候，会将 C 加到 arr[i] 中

>   arr[i] = diff[1] + diff[2] + ... + diff[L] + diff[L + 1] + ... + diff[i]
>
>   因为 diff[L] 增加了 C，因此 arr[i] 的结果比原来改变了 C

而对于最后一个区间，计算前缀和的时候，因为同时经过了 diff[L] 和 diff[R + 1] 因此，在计算的的时候，先进行了 +C，后进行了 -C 操作，因此 arr[i] 保持不变

初始化的时候，可以认为原来的一维数组全为 0，后来针对区间 [i, i] 进行了 n 次修改，得到了最终的 arr[i]，那么相当于针对差分数组 diff，每次让 diff[i] += arr[i]; diff[i + 1] -= arr[i]

因此初始化差分数组，需要的时间就是遍历一次原数组的时间

## 二维差分

其实一维的差分数组还好说，主要是二维的差分数组还存在一些疑问，不过思想还是一样的，即认为原矩阵 grid\[i][j] 为二维差分矩阵 diff\[i][j] 的前缀和

每次修改一个子矩阵$(x_1, y_1)$->$(x_2, y_2)$内的数字时，仅需要修改 diff 的边界即可

每次的操作：diff\[x1][y1] += C; diff\[x1][y2 + 1] -= C; diff\[x2 + 1][y1] -= C; diff\[x2 + 1][y2 + 1] += C

# 双指针

双指针利用了原有序列的性质，将暴力的两重枚举优化为一次遍历，时间复杂度从 $O(n^2)$ 降低为 $O(n)$

# 离散化

数组的下标肯定都是正数，有的时候输入的范围很大，而数据个数较小，此时如果单纯使用数组存储的话，可能导致 MLE

更有甚者，输入数据的范围包含了负数，显然不能直接作为数组下标使用

此时需要考虑将输入数据离散化

说是离散化，其实就是排序，并去重，这样数字映射后的位置，就是数组中每个数字的下标索引，当使用的时候，通过二分的方式，获取某个元素的下标

举例来说



# dp 问题

通过集合的方式理解 dp

首先，不管是求解种类个数、最值，其实都是从集合中进行各种方案的枚举，然后可能需要计算，所有方案的个数、或者方案的最值

简单的枚举，就是 DFS 暴力，有的时候方案个数是质数级别的，基本上就是超时

这个时候，可以通过 dp，从集合中，枚举一类集合(子集)，子集中包含了某一类的方案，这样，通过枚举子集，可以优化时间复杂度，可能方案的个数是指数级别的，但子集的个数可能就是可枚举的

dp 问题，关键在于：

*   确定状态的定义
*   确定状态转移方程

状态的定义，就是明确子集的定义，而一般 dp[i] 都是一个数，这个数，是集合的属性，属性可能是方案的个数，一类方案的最值...

确定状态转移方程，即计算子集的属性，一般的，一个子集需要被分成若干部分计算，将一个子集划分为若干部分，要求不重不漏，且每种划分需要是可枚举的，这样通过枚举每个子集的划分，就可以得到子集的属性(通过状态转移方程实现状态转移)

再要说明的一点是，并不是所有情况下，对于子集的划分，都需要满足不重复的，如果是计算方案的个数，显然不能让划分重复，但如果是求解最值，就算划分重复了，还是可以从重复的划分中获取到最值

但不管如何，在对子集进行划分的时候，一定需要满足不遗漏原则

子集划分的依据："寻找最后一个不同点"

# OI 中的数据结构

## 单链表

使用数组模拟单链表，初始化大数组，避免初始化结构体，更快

使用两个数组：e、ne 表示节点，其中 e[i] 表示某个节点的 val，ne[i] 表示当前节点的下一个节点

对于链表的尾节点，其 next[i] = -1(反正是一个起到标识作用的非法值)

特别的没有指定链表中第一个节点必须时 e[0]，使用了变量 head，表示头节点

因此一个典型的链表，可以通过两个指针和两个数组表示：

```c++
int head, idx;
int e[], ne[];
```

这个结构和链式前向星的图差不多，默认 head 指向 -1，表示指向尾节点，idx 表示下一个节点存储的位置

典型的结构：

```c++
// 实际的数据范围需要手动测试一下
const int N = (int)1e5 + 10;

int head, idx;
int e[N], ne[N];

// 初始化函数
void init() {
    head = -1;
    idx = 0;
}

// 将新的值作为头节点
void add_head(int val) {
    e[idx] = val;
    ne[idx] = head;
    head = idx++;
}

// 将当前节点添加到链表的第 i 个位置上
// 如果 i 为 1，等效于将新的值作为头节点
void add(int i, int val) {
    if (i == 1) add_head(val);
    else {
        int node = head;
        while (i > 2) {
            node = ne[node];
            i--;
        }
        e[idx] = val;
        ne[idx] = ne[node];
        ne[node] = idx++;
    }
}

// 删除头节点
void remove_head() {
    head = ne[head];
}

// 删除链表的第 i 个位置的元素
// 如果 i 为 1，等效于删除头节点
void remove(int i) {
    if (i == 1) remove_head();
    else {
        int node = head;
        while (i > 2) {
            node = ne[node];
            i--;
        }
        ne[node] = ne[ne[node]];
    }
}
```

## 双链表

也是类似的，不过这里使用了两个哨兵节点 head 和 tail 避免处理边界问题

```c++
const int N = (int)1e5 + 10;

int idx;

int left[N], right[N], e[N];

// 引入头尾哨兵节点
// 头节点即为 idx = 0
// 尾节点即为 idx = 1
void init() {
    idx = 2;
    right[0] = 1;
    left[0] = -1;
    left[1] = 0;
    right[1] = -1;
}

// 让新的值作为头节点
void add_head(int val) {
    e[idx] = val;
    
    right[idx] = right[0];
    left[right[idx]] = idx;
    
    left[idx] = 0;
    right[0] = idx++;
}

// 将新的值作为双向链表的第 i 个元素插入
// 当 i 取 1 等效于 add_head
void add(int i, int val) {
    int node = 0;
    while (i > 1) {
        node = right[node];
        i--;
    }
    
    e[idx] = val;
    
    right[idx] = right[node];
    left[right[idx]] = idx;
    
    left[idx] = node;
    right[node] = idx++;
}

// 删除头节点
void remove_head() {
    right[0] = right[right[0]];
    left[right[0]] = 0;
}

// 删除当前链表中的第 i 个元素插入
// 如果 i 为 1 等效于 remove_head
void remove(int i) {
    int node = 0;
    while (i > 1) {
        node = right[node];
        i--;
    }
    
    right[node] = right[right[node]];
    left[right[node]] = node;
}
```

## 栈

呵呵

```c++
const int N = (int)1e5 + 10;
// 栈顶指针
int t;
// 栈
int st[N];
// 栈的下标从数组下标suo 1 开始
void push(int val) {
    st[++t] = val;
}

int peek() {
    return st[t];
}

int size() {
    return t;
}


void pop() {
    t--;
}
```

## 队列

```c++
const int N = (int)1e5 + 10;

int q[N];
int hh, tt = -1;

void push(int val) {
    q[++tt] = val;
}

int query() {
    return q[hh];
}

int pop() {
    return q[hh++];
}
```

这里初始化 tt 为 -1
