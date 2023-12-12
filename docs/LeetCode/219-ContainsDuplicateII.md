---
title: 219.存在重复元素II
comments: true
---

#  219.存在重复元素II
## 题目描述
给你一个整数数组 nums 和一个整数 k ，判断数组中是否存在两个 不同的索引 i 和 j ，满足 nums[i] == nums[j] 且 abs(i - j) <= k 。如果存在，返回 true ；否则，返回 false 。

 

    示例 1：
    输入：nums = [1,2,3,1], k = 3
    输出：true

    示例 2：
    输入：nums = [1,0,1,1], k = 1
    输出：true

    示例 3：
    输入：nums = [1,2,3,1,2,3], k = 2
    输出：false
 
提示：
- 1 <= nums.length <= 105
- -109 <= nums[i] <= 109
- 0 <= k <= 105

## 答案
时间复杂度为O(N)
```cpp
class Solution {
public:
    bool containsNearbyDuplicate(vector<int>& nums, int k) {
        std::unordered_map<int,int> data;
        for(int i =0; i < nums.size(); i++){
            if(data.count(nums[i])){
                if(abs(i - data[nums[i]]) <=k) return true;
            }
            data[nums[i]] = i;
            
        }
        return false;
    }
};
```