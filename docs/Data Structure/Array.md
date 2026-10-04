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

### 2. 访问元素 (Access elements)
数组元素被存储在连续的内存空间中，这意味着计算数组元素的内存地址非常容易。给定数组内存地址（首元素内存地址）和某个元素的索引，我们可以使用图 4-2 所示的公式计算得到该元素的内存地址，从而直接访问该元素。

Array elements are stored in contiguous memory space, which means calculating the memory address of an array element is very straightforward. Given the array's memory address (the address of the first element) and the index of a specific element, we can use the formula shown in Figure 4-2 to calculate that element's memory address, thereby accessing it directly.

<img width="801" height="322" alt="image" src="https://github.com/user-attachments/assets/95eb532f-a2f4-4bf4-b84c-08d4935364c5" />

<img width="1339" height="293" alt="image" src="https://github.com/user-attachments/assets/742bf0a3-6344-4c3e-990d-732593a7a69f" />

观察图 4-2 ，我们发现数组首个元素的索引为 0，这似乎有些反直觉，因为从1开始计数会更自然。但从地址计算公式的角度看，索引本质上是内存地址的偏移量。首个元素的地址偏移量是0，因此它的索引为0是合理的。

在数组中访问元素非常高效，我们可以在O(1)时间内随机访问数组中的任意一个元素。

Looking at Figure 4-2, we observe that the index of the first array element is 0; this may seem counterintuitive, as counting from 1 feels more natural. However, from the perspective of the address calculation formula, an index is essentially a memory address offset. Since the offset for the first element is 0, assigning it an index of 0 is logical.

Accessing elements in an array is highly efficient; we can randomly access any element in O(1) time.

```java
/* Random access to element *
int randomAccess(int[] nums) {
  // Randomly select a number in the interval [0, nums.length)
  int randomIndex = ThreadRandomLocal.current().nextInt(0, nums.length)
  // Retrieve and return the random element
  int randomNum = nums[randomIndex];
  return randomNum;
}
```

### 3.   插入元素 (Insert element)
数组元素在内存中是“紧挨着的”，它们之间没有空间再存放任何数据。如图 4-3 所示，如果想在数组中间插入一个元素，则需要将该元素之后的所有元素都向后移动一位，之后再把元素赋值给该索引。

Array elements are stored contiguously in memory, leaving no space between them to store additional data. As shown in Figure 4-3, if you wish to insert an element into the middle of an array, you must shift all subsequent elements one position to the right before assigning the new element to that index.



