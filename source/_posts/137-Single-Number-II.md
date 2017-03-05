---
title: 137 Single Number II
date: 2017-03-05 10:42:54
tags: [bit_operation]
categories: [leetcode]
---
# 数组中除了一个数出现k = 3次，其他数字都出现p = 1次，找出这个特别的数
<!--more-->
[description](https://leetcode.com/problems/single-number-ii/?tab=Description)

	/*Statement of our problem: "Given an array of integers, every element
    appears k (k > 1) times except for one, which appears p times
    (p >= 1, p % k != 0). Find that single one."*/
    public int singleNumber(int[] nums) {
        /*because we have k = 3, so each bit need two bit to count, bit2 provide
        the higher bit in bit count*/ 
        int bit2 = 0, bit1 = 0, mask = 0;
        for(int ni : nums) {
            /* when bit1 and ni both provide 1, bit2 got a bypass, otherwise it won't
            change*/
            bit2 ^= bit1 & ni;
            // the exlusive or is doing adding without bypass
            bit1 ^= ni;
            /*when reach k, we should cut it to zero, here k == 3, so find the bit
            which provide in bit2 and bit1(0x3), then use xor to make them zero,
            for other k, we can use ~, & to construct the mask bits, such as 10,
            the mask is bit2 & ~bit1*/
            mask = bit1 & bit2;
            bit2 ^= mask;
            bit1 ^= mask;
        }
        // 0x1 indicate the number than appears only once
        return bit1;
    }
    
  * 使用m个整数为各个比特提供2^m个计数值
  * 将进位与加和分开，第n位只有前n - 1以及输入位全为1时才会变化（异或这些位的并），最低位直接异或求输入
  * 终极化简形式：

		public int singleNumber(int[] A) {
			int ones = 0, twos = 0;
			for(int i = 0; i < A.length; i++){
				ones = (ones ^ A[i]) & ~twos;
				twos = (twos ^ A[i]) & ~ones;
			}
			return ones;
		}

