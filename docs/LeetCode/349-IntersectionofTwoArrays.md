---
title: 349.两个数组的交集
comments: true
---

#  349.两个数组的交集
## 题目描述
给定两个数组 nums1 和 nums2 ，返回 它们的交集 。输出结果中的每个元素一定是 唯一 的。我们可以 不考虑输出结果的顺序 。


    示例 1：
    输入：nums1 = [1,2,2,1], nums2 = [2,2]
    输出：[2]

    示例 2：
    输入：nums1 = [4,9,5], nums2 = [9,4,9,8,4]
    输出：[9,4]
    解释：[4,9] 也是可通过的
 

提示：

1 <= nums1.length, nums2.length <= 1000

0 <= nums1[i], nums2[i] <= 1000

## 答案
时间复杂度为O(m * n)
```cpp
class Solution {
public:
    vector<int> intersection(vector<int>& nums1, vector<int>& nums2) {
        vector<int> result;
        for(auto left : nums1){
            for(auto right : nums2){
                if(left == right && find(result.begin(),result.end(),left) == result.end()){
                    result.push_back(left);
                }
            }
        }
        return result;
    }
};
```