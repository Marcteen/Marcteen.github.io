---
title: 11 Container With Most Water
date: 2017-03-04 17:51:18
tags: [dp]
categories: [leetcode]
---
# 给定一个值序列作为y序列，找出任意两个纵线与x轴所形成桶的最大容积
<!--more-->
[description](https://leetcode.com/problems/container-with-most-water/?tab=Description)

	public int maxArea(int[] height) {
    	int head = 0, tail = height.length - 1;
    	int maxArea = 0;
    	int area = 0;
    	while(tail > head) {
    		area = (tail - head) * Math.min(height[head], height[tail]);
    		if(area > maxArea)
    			maxArea = area;
    		/*As Buckets effect reveals, the capacity of a bucket depends
    		 * on the shortest board, so , we HAVE to move the shortest side.*/
    		if(height[head] < height[tail])
    			head ++;
    		else
    			tail --;
    	}
    	return maxArea;
    }
    
这里借助了动态规划的思路，以收缩桶的方式进行，，最开始from与to分别是数列首位，假设from边比较短，这时桶的容积会被其限制，为了将容积增加，要么扩展底边，要么提高最矮边，但是最矮边不一定会被提高，那么就需要进行比较选择比较大的值。

Consider it as a dynamic programming problem and you get the clear proof.
	assume h[from] < h[to], then the largest volume we can get, using 'from' as one side, is (to - from + 1) * h[from]. then we can say: opt[from][to] = max(opt[from + 1][to], (to - from + 1) * h[from]).
	
反映到收缩桶的边上，每次收缩必须抛弃最小的边，否则底边减小了，而矮边没有被提高的可能，桶的没有可能更大。

