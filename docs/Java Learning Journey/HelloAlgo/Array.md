# 数组 (Array)
数组（array）是一种线性数据结构，其将相同类型的元素存储在连续的内存空间中。我们将元素在数组中的位置称为该元素的索引（index）。图 4-1 展示了数组的主要概念和存储方式。

An array is a linear data structure that stores elements of the same type in contiguous memory space. The position of an element in the array is called its index. Figure 4-1 illustrates the main concepts of an array and how it is stored.

<img width="1422" height="334" alt="image" src="https://github.com/user-attachments/assets/06d23f8c-bc65-4f73-afe4-52088739f243" />

## 4.1.1   数组常用操作 (Common Array Operations)
### 1.   初始化数组 (Initialize the array)
我们可以根据需求选用数组的两种初始化方式：无初始值、给定初始值。在未指定初始值的情况下，大多数编程语言会将数组元素初始化为 0

We can choose between two ways of initializing an array based on requirements: without initial values ​​or with specified initial values. When no initial values ​​are specified, most programming languages ​​initialize array elements to 0.

```java
int[] arr  = new int[5]; //{0,0,0,0,0}
int[] nums = {1,3,2,5,4}
```

### 2. 访问元素 (Access Elements)

数组元素存储在连续的内存空间中，因此可以根据数组的起始地址和元素索引直接计算出目标元素的内存地址，从而快速访问该元素。

Array elements are stored in contiguous memory locations. Therefore, the memory address of any element can be calculated directly from the array's starting address and the element's index, allowing efficient access to that element.

<img width="801" height="322" alt="image" src="https://github.com/user-attachments/assets/95eb532f-a2f4-4bf4-b84c-08d4935364c5" />

元素地址的计算公式如下：

The memory address of an element can be calculated using the following formula:

```text
Element Address = Array Starting Address + Element Size × Element Index

元素地址 = 数组起始地址 + 元素大小 × 元素索引
```

例如，对于一个 `int` 数组，每个元素占用 `4` 个字节。假设数组的起始地址为 `000`，则索引为 `3` 的元素地址为：

For example, in an `int` array, each element occupies `4` bytes. If the starting address of the array is `000`, the address of the element at index `3` is:

```text
000 + 4 × 3 = 012
```

因此，计算机只需进行一次地址计算，便可直接定位到目标元素，而无需依次遍历前面的元素。

Therefore, the computer only needs to perform a single address calculation to locate the target element directly, without traversing the preceding elements.

<img width="1339" height="293" alt="image" src="https://github.com/user-attachments/assets/742bf0a3-6344-4c3e-990d-732593a7a69f" />

#### 为什么数组索引从 0 开始？(Why Do Array Indexes Start at 0?)

数组索引从 `0` 开始，这可能看起来有些不直观，因为人们通常习惯从 `1` 开始计数。然而，从内存地址计算的角度来看，索引本质上表示元素相对于数组起始地址的偏移量（offset）。

Array indexes start at `0`, which may seem counterintuitive because people naturally tend to count from `1`. However, from the perspective of memory address calculation, an index essentially represents the offset of an element from the array's starting address.

第一个元素距离数组起始地址的偏移量为 `0`，因此其索引为 `0`。

The first element has an offset of `0` from the starting address, so its index is naturally `0`.

#### 随机访问 (Random Access)

由于任意元素的地址都可以通过上述公式直接计算，因此数组支持**随机访问（Random Access）**。

Because the address of any element can be calculated directly using the formula above, arrays support **Random Access**.

这里的“随机”并不表示随机选择元素，而是表示可以直接访问任意索引位置的元素，而无需遍历数组中的其他元素。

Here, "random" does not mean randomly selecting an element. Instead, it refers to the ability to directly access an element at any index without traversing other elements in the array.

无论访问的是索引 `3` 还是索引 `3,000,000`，都只需要一次地址计算，因此数组访问元素的时间复杂度为 **O(1)**。

Whether accessing index `3` or index `3,000,000`, only a single address calculation is required. Therefore, accessing an array element has a time complexity of **O(1)**.

Example

```java
/* Random access to an element */
int randomAccess(int[] nums) {
    // Randomly select an index in [0, nums.length)
    int randomIndex = ThreadLocalRandom.current().nextInt(0, nums.length);

    // Access and return the element directly
    return nums[randomIndex];
}
```

### 3.   插入元素 (Insert element)
数组元素在内存中是“紧挨着的”，它们之间没有空间再存放任何数据。如图 4-3 所示，如果想在数组中间插入一个元素，则需要将该元素之后的所有元素都向后移动一位，之后再把元素赋值给该索引。

Array elements are stored contiguously in memory, leaving no space between them to store additional data. As shown in Figure 4-3, if you wish to insert an element into the middle of an array, you must shift all subsequent elements one position to the right before assigning the new element to that index.

<img width="1353" height="806" alt="image" src="https://github.com/user-attachments/assets/a590d598-b679-47f9-aa00-30b59be5885b" />

值得注意的是，由于数组的长度是固定的，因此插入一个元素必定会导致数组尾部元素“丢失”。我们将这个问题的解决方案留在“列表”章节中讨论。

It is worth noting that since the length of an array is fixed, inserting an element will inevitably push the last element out of the array. We will leave the solution to this problem for discussion in the "List" chapter.

```java
/* 在数组的索引 index 处插入元素 num */
void insert(int[] nums, int num, int index) {
    // 把索引 index 以及之后的所有元素向后移动一位
    for (int i = nums.length - 1; i > index; i--) {
        nums[i] = nums[i - 1];
    }
    // 将 num 赋给 index 处的元素
    nums[index] = num;
}
```

### 4.   删除元素 (Delete element)
同理，如图 4-4 所示，若想删除索引i处的元素，则需要把索引i之后的元素都向前移动一位。

Similarly, as shown in Figure 4-4, if you want to delete the element at index i, all elements following index i must be shifted forward by one position.

<img width="1090" height="785" alt="image" src="https://github.com/user-attachments/assets/3b256377-a49d-430f-af90-1ae91fbef895" />

请注意，删除元素完成后，原先末尾的元素变得“无意义”了，所以我们无须特意去修改它。

Note that once the element has been removed, the element originally at the end becomes "meaningless," so there is no need to explicitly modify it.

```java
/* 删除索引 index 处的元素 */
void remove(int[] nums, int index) {
    // 把索引 index 之后的所有元素向前移动一位
    for (int i = index; i < nums.length - 1; i++) {
        nums[i] = nums[i + 1];
    }
}
```

总的来看，数组的插入与删除操作有以下缺点：
- 时间复杂度高：数组的插入和删除平均时间复杂度均为 O(n)，其中 n 为数组长度。
- 丢失元素：由于数组长度不可变，在插入新元素后，超出数组长度范围的元素可能会被覆盖或丢失。
- 内存浪费：可以预先创建一个较大的数组，仅使用前面一部分空间。这样在插入元素时，即使末尾元素被覆盖，也不会影响有效数据，但会造成部分内存空间的浪费。

Overall, array insertion and deletion operations have the following drawbacks:
- High time complexity: The average time complexity of both insertion and deletion operations in an array is O(n), where n is the length of the array.
- Loss of elements: Since the length of an array is fixed, elements that exceed the array's capacity after an insertion may be overwritten or lost.
- Memory wastage: A larger array can be initialized in advance while only a portion of it is used. This prevents meaningful data from being lost during insertions, but  results in unused memory space and therefore wastes memory.

### 5.   遍历数组 (Iterate through the array)
在大多数编程语言中，我们既可以通过索引遍历数组，也可以直接遍历获取数组中的每个元素：

In most programming languages, we can iterate through an array either by using indices or by directly accessing each element:

```java
/* 删除索引 index 处的元素 */
void traverse(int[] nums) {
    int count = 0;
    // 通过索引遍历数组
    for (int i = 0; i < nums.length; i++) {
        count += nums[i];
    }
    // 直接遍历数组元素
    for (int num : nums) {
        count += num;
    }
}
```

### 6.   查找元素 (Find element)
在数组中查找指定元素需要遍历数组，每轮判断元素值是否匹配，若匹配则输出对应索引。

Finding a specific element in an array requires traversing the array; in each iteration, the element's value is checked for a match, and if a match is found, the corresponding index is output.

因为数组是线性数据结构，所以上述查找操作被称为“线性查找”。

```java
/* 在数组中查找指定元素 */
int find(int[] nums, int target) {
    for (int i = 0; i < nums.length; i++) {
        if (nums[i] == target)
            return i;
    }
    return -1;
}
```

Since an array is a linear data structure, the aforementioned search operation is called "linear search."

### 7.   扩容数组 (Resize the array)
在复杂的系统环境中，程序难以保证数组之后的内存空间是可用的，从而无法安全地扩展数组容量。因此在大多数编程语言中，数组的长度是不可变的。

In complex system environments, it is difficult for a program to guarantee that the memory space immediately following an array is available, making it impossible to safely expand the array's capacity. Consequently, in most programming languages, the length of an array is immutable.

如果我们希望扩容数组，则需重新建立一个更大的数组，然后把原数组元素依次复制到新数组。这是一个O(n)的操作，在数组很大的情况下非常耗时。代码如下所示：

If we wish to expand the array, we must create a larger array and then copy the elements from the original array into the new one one by one. This is an O(n) operation, which is very time-consuming when the array is large. The code is shown below:

```java
/* 扩展数组长度 */
int[] extend(int[] nums, int enlarge) {
    // 初始化一个扩展长度后的数组
    int[] res = new int[nums.length + enlarge];
    // 将原数组中的所有元素复制到新数组
    for (int i = 0; i < nums.length; i++) {
        res[i] = nums[i];
    }
    // 返回扩展后的新数组
    return res;
}
```
