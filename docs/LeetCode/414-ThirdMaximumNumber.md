---
title: 414.第三大的数
comments: true
---

#  414.第三大的数
## 题目描述
给你一个非空数组，返回此数组中 第三大的数 。如果不存在，则返回数组中最大的数。

    示例 1：
    输入：[3, 2, 1]
    输出：1
    解释：第三大的数是 1 。
    
    示例 2：
    输入：[1, 2]
    输出：2
    解释：第三大的数不存在, 所以返回最大的数 2 。
    
    示例 3：
    输入：[2, 2, 3, 1]
    输出：1
    解释：注意，要求返回第三大的数，是指在所有不同数字中排第三大的数。
    此 例中存在两个值为 2 的数，它们都排第二。在所有不同数字中排第三大的数为 1 。
 

提示：

1 <= nums.length <= 104

-231 <= nums[i] <= 231 - 1
 

进阶：你能设计一个时间复杂度 O(n) 的解决方案吗？

## 答案
时间复杂度为 O(N)
```cpp
class Solution {
public:
    int thirdMax(vector<int>& nums) {
        set<int> data(nums.begin(),nums.end());
        int first =  numeric_limits<int>::min();
        int second = numeric_limits<int>::min();
        int third =numeric_limits<int>::min();
        for(auto value : data){
            if( value > first){
                third = second;
                second = first;
                first = value;
            }else if(value > second){
                third = second;
                second = value;
            }else if( value > third){
                third = value;
            }

        } 
        if(data.size() < 3) return first;
        return third;
    }
};
```