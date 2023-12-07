---
title: 119.杨辉三角 II
comments: true
---

# 119.杨辉三角 II
## 题目描述
给定一个非负索引 rowIndex，返回「杨辉三角」的第 rowIndex 行。

在「杨辉三角」中，每个数是它左上方和右上方的数的和。
![](images/PascalTriangleAnimated2.gif)

    示例 1:
    输入: rowIndex = 3  
    输出: [1,3,3,1]

    示例 2:
    输入: rowIndex = 0
    输出: [1]
    
    示例 3:
    输入: rowIndex = 1
    输出: [1,1]
 
提示:
- 0 <= rowIndex <= 33


你可以优化你的算法到 O(rowIndex) 空间复杂度吗？


## 答案
时间复杂度为O(n^2)
```cpp
class Solution {
public:
    vector<int> getRow(int rowIndex) {
        vector<vector<int>> result(rowIndex + 1);
        for(int index = 0; index <= rowIndex; index++){
           result[index].resize(index + 1);
           result[index][0] = result[index][index] = 1;
           for(int j=1 ; j < index;j++){
                result[index][j] = result[index-1][j-1] + result[index-1][j]; 
           }
        }
        return result[rowIndex];
    }
};
```
进阶写法：
```cpp
class Solution {
public:
    vector<int> getRow(int rowIndex) {
        vector<int> result(rowIndex + 1,1);

        for(int index = 0; index <= rowIndex; index++){
            int pre = result[0];
           for(int j=1 ; j < index;j++){
               int tmp = result[j];
                result[j] = pre + result[j];
                pre = tmp; 
           }
        }
        return result;
    }
};
```