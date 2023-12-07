---
title: 118.杨辉三角
comments: true
---

# 118.杨辉三角
## 题目描述
给定一个非负整数 numRows，生成「杨辉三角」的前 numRows 行。

在「杨辉三角」中，每个数是它左上方和右上方的数的和。

![](images/PascalTriangleAnimated2.gif)

    示例 1:
    输入: numRows = 5
    输出: [[1],[1,1],[1,2,1],[1,3,3,1],[1,4,6,4,1]]

    示例 2:
    输入: numRows = 1
    输出: [[1]]
 
提示:
- 1 <= numRows <= 30

## 答案
时间复杂度为 O(n^2)
```cpp
class Solution {
public:
    vector<vector<int>> generate(int numRows) {
        vector<vector<int>> result;
        for(int i= 0; i < numRows; i++){
            vector<int> tmp;
            for(int j = 0; j < i + 1; j++){
                if(j == 0) {
                    tmp.push_back(1);
                }else if(j == i && i !=0){
                    tmp.push_back(1);
                }else{
                    tmp.push_back(result[i-1][j-1] + result[i-1][j]);
                }
            }
            result.push_back(tmp);
        }
        return result;
    }
};
```