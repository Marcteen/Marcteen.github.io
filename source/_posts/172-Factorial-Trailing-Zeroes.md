---
title: 172 Factorial Trailing Zeroes
date: 2017-03-05 15:23:39
tags: [sequence, number, factorial]
categories: [leetcode]
---
# 找出阶乘后零的个数
<!--more-->
[description](https://leetcode.com/problems/factorial-trailing-zeroes/?tab=Solutions)

	/* we need 2 and 5 to form the trailing zeroes, and every
	 * odd number has 2 as factor which is sure to be sufficient,
	 * so we just need to concern the number of fives*/
	public int trailingZeroes(int n) {
		int result = 0;
		/*basically, all the multiplier of 5^m (m = 1, 2, 3...) provide
		 * 5, but for 5^m, it only provide one more 5 because we have
		 * count it in 5^(m - 1), just like n / 5 + n / 25 + n / 125 + ...,
		 * but here we let n /= 5 (decrease the dividend) in each iteration
		 * to get the same resultas increasing the divisor*/
		while(n > 0) {
			result += n / 5;
			n /= 5;
		}
		return result;
	}	
* 找出所有包含5, 125, 625...为因数的数
* 包含5^m的数已经被5^k (k < m)，处理过数次，因此当除数增加时，这些数实际上只会再贡献一个5
* 求解时采用逐步降低被除数的办法，可达到完全一样的效果

