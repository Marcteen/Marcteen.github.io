---
title: 39 Combination Sum
date: 2017-03-04 19:41:18
tags: [combination, dp]
categories: [leetcode]
---
# 给定一些正整数，找出所有和等于目标值的组合，组合内可以使用重复值
<!--more-->
[description](https://leetcode.com/problems/combination-sum/?tab=Description)

	public List<List<Integer>> combinationSum(int[] candidates, int target) {
        List<List<List<Integer>>> dp = new ArrayList<>(); // form the solution from 1 to target, not so relevant to candidates
        if(null == candidates || 0 == candidates.length)
            return null;
        Arrays.sort(candidates);
        List<List<Integer>> newList = null;
        for(int subTarget = 1; subTarget <= target; subTarget ++) { // dp is one-way done, no re-run needed!!!!
            newList = new ArrayList<>();
            for(int j = 0; j < candidates.length && candidates[j] <= subTarget; j ++) {// we must use a existing candidate to form combination
                if(candidates[j] == subTarget) {// no need to find complement
                    newList.add(Arrays.asList(new Integer[]{subTarget}));// add a array to the list of array
                }
                else {
                    for(List<Integer> complement : dp.get(subTarget - candidates[j] - 1)) { // the index is smaller by one
                        //to avoid duplication, we need to keep it sorted, very concise!!!!!!
                        if(candidates[j] <= complement.get(0)) {// smaller candidates will cover bigger one, and keep combination sorted
                            List<Integer> combine = new ArrayList<>();
                            combine.add(candidates[j]);
                            combine.addAll(complement);
                            newList.add(combine);
                        }
                    }
                }
                
            }
            dp.add(newList);// add it any way to avoid null in impossible combination, make algorithm general
        }
        return dp.get(target - 1);
    }

使用动态规划，逐步构造子结果，为了保证最后不会产生重复的组合，可以令组合呈递增排序。同时将原数组排序，这样便于利用所有可以利用的元素进行组合。外循环构字结果从1（1至target），内循环逐步选用每一个小于当前子结果的和进行新一层解的构造，并且规定每一次只能从头部将元素加入扩充，并且保证扩充后的组合依然是有序的，这样就能够不重复地找到所有满足条件的组合。

例如，给定数字1，2，3，要求和为5，那么对于3，可以产生(2, 3)作为解，但是当构造5时，可以使用3加入到(2)中，这会与2加入到(3)中重复，通过限制头部插入，结合候选值的递增性，可实现重复避免。

* 排序候选数组
* 初始化一个空list作为0的解
* 利用有序性和头部插入（新加值小于待合并序列的第一个值），穷举并避免重复


