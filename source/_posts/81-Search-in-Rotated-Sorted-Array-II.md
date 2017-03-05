---
title: 81 Search in Rotated Sorted Array II
date: 2017-03-04 23:13:34
tags: [binary_search]
categories: [leetcode]
---
# 在包含重复元素并被以一定支点旋转的有序序列中查找元素
<!--more-->
[description](https://leetcode.com/problems/search-in-rotated-sorted-array-ii/?tab=Description)

	public boolean search(int[] nums, int target) {
        if(null == nums || 0 == nums.length)
            return false;
        int low = 0, high = nums.length - 1, mid = 0;
        while(low <= high) {
            mid = low + (high - low) / 2;
            if(nums[mid] == target)
                return true;
            /*for 1121111 and 222221222, the euqlity will make it wrong*/
            if(nums[low] < nums[mid] && nums[mid] < nums[high]) {// into a regular BS
                if(nums[mid] > target)
                    high = mid - 1;
                else
                    low = mid + 1;
            }
            /*Here we pass the [low]<[mid]<[high], and we must take care of all the remaing case, except both equal,
            there are cases consist of equal and other inversed unequal*/
            /* When [mid] > [low], the left part must be in order. When [mid] == [low], we skip the [mid] == [low]
            case and leave it for the final branch; if [mid] > [high], it still the same with last "when"; if [mid] < [high],
            we have [low]=[mid]<[high], it must be a regular BS case, otherwise, mid can't be equal to low, but it should
            have same shrink patter with next if-else branch.*/
            else if(nums[mid] > nums[low] || nums[mid] > nums[high]) {
                if(nums[mid] > target && target >= nums[low])// the left side is sure to be in order, and target is there
                        high = mid - 1;
                    else
                        low = mid + 1;
            /* When [mid] < [high], the right part must be in order. When [mid] == [high], we skip the [mid] == [low]
            case and leave it for the final branch; if [mid] < [low], it still the same with last "when"; if [mid] > [low],
            we have [low]<[mid]=[high], it must be a regular BS case, otherwise, mid can't be equal to high, but it should
            have same shrink patter with last if-else branch.*/
            } else if(nums[mid] < nums[low] || nums[mid] < nums[high]) {// the right side is sure in order
                if(nums[mid] < target && target <= nums[high])
                    low = mid + 1;
                else
                    high = mid - 1;
            } else /* here both equal, so shrink it, both high -- and low ++ work fine(because no
                    info to determine which is better!!!!!!!!!!).*/
                low ++;
        }
        return false; // when low == high, must not found to reach here;
    }
    
   * 因为存在重复元素，可能出现三个标志的值相等，此时不能判断如何移动，只能单步收缩
   * 如上一条所所属，除了三标志位呈严格有序关系，需要小心处理除三者相等外的分支，最终归纳为上面两种情况，主要按照指针行为进行组合。


