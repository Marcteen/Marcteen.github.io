---
title: 75 Sort Colors
date: 2017-03-04 23:34:40
tags: [partition]
categories: [leetcode]
---
# 将只有三个（少量）值的数组进行排序，借助类似于快排的方式
<!--more-->
[description](https://leetcode.com/problems/sort-colors/?tab=Solutions)

	public void sortColors(int[] nums) {
        if(null == nums || nums.length < 2)
            return;
        int zero = 0, second = nums.length - 1;
        for(int i = 0; i <= second; i ++) {/*It handle nums[0] for potential 2, make it position available and correct*/
        	while(2 == nums[i] && second > i)/*nums[second] may be two, too. This loop ensures it swap to the farthest
            non-two element!!!*/
                swap(nums, i, second --);
            while(0 == nums[i] && zero < i)
                swap(nums, i , zero ++);
            
        }
    }
    
    public void swap(int[] nums, int i, int j) {
        nums[i] ^= nums[j];
        nums[j] ^= nums[i];
        nums[i] ^= nums[j];
    }
    
  每一轮先处理2，再处理零，因为有可能将后面的0调换到当前位置，若后处理2，这个被调换来的0会被错过，值得注意的是，不可能出现在处理0时将前面的2换过来而出现错过2的情况，因为之前的2必然已经被交换到后面；而后面的0还没有被交换到前面，所以要后处理2.
  
* 注意交换一定要进行到真正有效为止。

