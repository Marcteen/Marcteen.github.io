---
title: 6 ZigZag Conversion
date: 2017-03-04 17:21:48
tags: [string]
categories: [leetcode]
---
# 将字符串按照指定的层数组织之字形，然后按照“层序”输出结果
<!--more-->
[description](https://leetcode.com/problems/zigzag-conversion/?tab=Description)

	public static String convert(String s, int numRows) {
		if(1 == numRows){
			return s;
		}
		else {
			StringBuffer result = new StringBuffer();
			
			for(int j = 0; j < s.length(); j += 2 * numRows - 2) { //deal with the first row
				result.append(s.charAt(j));
			}
			
			for(int row = 2;row < numRows;row ++) { //deal with the middle rows
				int i = 0; //// divide to two equal-difference array
				for(int j = row - 1; j < s.length(); j += 2 * numRows - 2) {
					result.append(s.charAt(j));
					i = j + 2 * (numRows - row);
					if(i < s.length())
						result.append(s.charAt(i));
				}
			}
			
			for (int j = numRows - 1; j < s.length(); j += 2 * numRows - 2) { //deal with the last line
				result.append(s.charAt(j));
			}
			
			return result.toString();
		}
		
	}
这里采用的方法是先处理首层，再处理中间层，最后处理尾层的方式。本质上来说，就是中间层有两个步长，每次走两步，但是各层的步长和是一致的，步长和公式为
	
	strideSum = 2n - 2
各层的步长有对应关系。

此外，我们还以申请层数个StringBuilder，按照之字形分别往不同层的builder中输出字符，最后再全部连接到一起即可。

* 下标取字符串时要判断字符串是否越界
* 双步长，各层起始点递增

