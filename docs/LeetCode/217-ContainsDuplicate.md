---
title: 217.存在重复元素
comments: true
---

#  217.存在重复元素
## 题目描述
给你一个整数数组 nums 。如果任一值在数组中出现 至少两次 ，返回 true ；如果数组中每个元素互不相同，返回 false 。
 

    示例 1：
    输入：nums = [1,2,3,1]
    输出：true

    示例 2：
    输入：nums = [1,2,3,4]
    输出：false

    示例 3：
    输入：nums = [1,1,1,3,3,4,3,2,4,2]
    输出：true
 

提示：
- 1 <= nums.length <= 105
- -109 <= nums[i] <= 109

## 答案
时间复杂度为O(N)
```cpp
class Solution {
public:
    bool containsDuplicate(vector<int>& nums) {
        std::unordered_map<int,int> data;
        for(auto value : nums){
            data[value]++;
            if(data[value] == 2) return true;
        }
        return false;
    }
};
```