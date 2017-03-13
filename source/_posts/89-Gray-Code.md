---
title: 89 Gray Code
date: 2017-03-04 23:07:16
tags: [bit_operation]
categories: [leetcode]
---
# 生成格雷码
<!--more-->
[description](https://leetcode.com/problems/gray-code/?tab=Description)

	public List<Integer> grayCode(int n) {
        List<Integer> result = new ArrayList<>((int)Math.pow(2, n));
        result.add(0);
        if(0 == n)
            return result;
        result.add(1);
        int max = (int)Math.pow(2, n - 1), tail = 0;
        for(int prefix = 2; prefix <= max; prefix *= 2) {// add "1" bit to current digits inversely, just right.....
            tail = result.size() - 1;
            for(;tail >= 0; tail --) {
                result.add(prefix + result.get(tail));
            }
        }
        return result;
    }
    
    public List<Integer> grayCodeEval(int n) {
        List<Integer> result = new ArrayList<>((int)Math.pow(2, n));
        result.add(0);
        int all = (int)Math.pow(2, n) - 1;
        for(int i = 1; i <= all; i ++)
            result.add(i ^ (i >> 1));
        return result;
    }

 两种方式，一种是逐步添加最高位为1，低位部分恰好与之前的结果成镜像对称；第二种方式是格雷码(最常见最基础这种)与自然二进制的对应关系，解释太过晦涩，不必深究
 
 	grey_i = i ^ (i >> 1)


