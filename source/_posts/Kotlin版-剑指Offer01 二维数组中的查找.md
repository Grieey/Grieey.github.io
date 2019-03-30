---
title: "Kotlin版-剑指Offer01 二维数组中的查找"
date: 2018-11-07T21:18:46+08:00
tags: 
    - 算法
    - Kotlin
categories: 
    - 算法

---

> 题目：在一个二维数组中，每一行都按照从左到右递增的顺序排序，每一列都按照从上到下递增的顺序排序，请完成一个函数，输入这样的一个二维数组和一个整数，判断数组中是否含有该整数。

- 思路：因为是有序的二维数组，那么利用有序的特性，可以在行列上移动着查找，当目标数值大于了本行的最后一列的数，那说明这个说只会在下列之后，反之亦然。
- 核心代码

```kotlin
private fun solution(array: Array<IntArray>, target:Int): Boolean{

    if (!array.isNotEmpty()
        || target > array.last().last()
        || target < array.first().first()){
        return false
    }

    var row = array.size - 1
    var col = 0
    val maxCol = array.first().size

    while (row > -1 && col > -1 && col < maxCol){
        val now = array[row][col]
        when {
            now < target -> col++
            now > target -> row--
            else -> return true
        }
    }

    return false
}
```

