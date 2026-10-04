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

#### Why Do Array Indexes Start at 0?（为什么数组索引从 0 开始？）

数组索引从 `0` 开始，这可能看起来有些不直观，因为人们通常习惯从 `1` 开始计数。然而，从内存地址计算的角度来看，索引本质上表示元素相对于数组起始地址的偏移量（offset）。

Array indexes start at `0`, which may seem counterintuitive because people naturally tend to count from `1`. However, from the perspective of memory address calculation, an index essentially represents the offset of an element from the array's starting address.

第一个元素距离数组起始地址的偏移量为 `0`，因此其索引为 `0`。

The first element has an offset of `0` from the starting address, so its index is naturally `0`.

#### Random Access（随机访问）

由于任意元素的地址都可以通过上述公式直接计算，因此数组支持**随机访问（Random Access）**。

Because the address of any element can be calculated directly using the formula above, arrays support **Random Access**.

这里的“随机”并不表示随机选择元素，而是表示可以直接访问任意索引位置的元素，而无需遍历数组中的其他元素。

Here, "random" does not mean randomly selecting an element. Instead, it refers to the ability to directly access an element at any index without traversing other elements in the array.

无论访问的是索引 `3` 还是索引 `3,000,000`，都只需要一次地址计算，因此数组访问元素的时间复杂度为 **O(1)**。

Whether accessing index `3` or index `3,000,000`, only a single address calculation is required. Therefore, accessing an array element has a time complexity of **O(1)**.

#### Example

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



