---
title: 169.多数元素
comments: true
---

# 169.多数元素
## 题目描述
给定一个大小为 n 的数组 nums ，返回其中的多数元素。多数元素是指在数组中出现次数 大于 ⌊ n/2 ⌋ 的元素。

你可以假设数组是非空的，并且给定的数组总是存在多数元素。

 
    示例 1：
    输入：nums = [3,2,3]
    输出：3

    示例 2：
    输入：nums = [2,2,1,1,1,2,2]
    输出：2
 
提示：
- n == nums.length
- 1 <= n <= 5 * 104
- -109 <= nums[i] <= 109
 

进阶：尝试设计时间复杂度为 O(n)、空间复杂度为 O(1) 的算法解决此问题。

## 答案
时间复杂度为O(N)
### 哈希法
```cpp
class Solution {
public:
    int majorityElement(vector<int>& nums) {
        int result;
        std::unordered_map<int, int> data;
        for(auto value : nums){
            data[value]++;
            if(data[value] > nums.size() /2) result = value;
        }
        return result;
    }
};
```
### 排序法
```cpp
class Solution {
public:
    int majorityElement(vector<int>& nums) {
        sort(nums.begin(),nums.end());
        return nums[nums.size() / 2];
    }
};
```
### 投票法
```cpp
class Solution {
public:
    int majorityElement(vector<int>& nums) {
        int count = 0, result = 0;
        for(auto value: nums){
            if(value == result) count++;
            else if(count) count--;
            else{
                result = value;
                count++;
            }
        }
        return result;
    }
};
```