---
title: 118 Pascal's Triangle
date: 2017-03-05 09:06:33
tags: [array, recurrence]
categories: [leetcode]
---
# 求出帕斯卡三脚形各层的值
<!--more-->
[description](https://leetcode.com/problems/pascals-triangle/?tab=Description)

	public List<List<Integer>> generate(int numRows) {
		List<List<Integer>> result = new ArrayList<>();
		List<Integer> curResult = new ArrayList<>();
		List<Integer> lastResult = new ArrayList<>();
		if(0 == numRows) {
			return result;
		}
		else {
			curResult.add(1);
			result.add(new ArrayList(curResult));
			for(int i = 1; i < numRows; i ++) {
				curResult.clear();
				curResult.add(1);
				for(int j = 1; j < i; j ++) {
					lastResult = result.get(i - 1);
					curResult.add(lastResult.get(j - 1) + lastResult.get(j));
				}
				curResult.add(1);
				result.add(new ArrayList(curResult));
			}
		}
		return result;
	}
	
	public List<List<Integer>> generateCompact(int numRows) {
		List<List<Integer>> allrows = new ArrayList<List<Integer>>();
    	ArrayList<Integer> row = new ArrayList<Integer>();
    	for(int i = 0; i < numRows; i++)
    	{
    		row.add(0, 1); // noted that the new element is insert at the head
    		for(int j = 1; j < row.size() - 1; j++)
    			row.set(j, row.get(j) + row.get(j+1));
    		allrows.add(new ArrayList<Integer>(row));
    	}
    	return allrows;
	}
	
容易看出其本质，下一层的新元素是上一层的相邻元素之和，这可以化简到首先直接复制上一层结果，然后从第二个元素其，在其下标位置插入一个其自身与其前方元素之和的新元素，通过这种递推关系算出每一层的结果。


