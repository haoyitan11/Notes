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



