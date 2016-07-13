---
layout: post
title: 动态规划小结
subtitle: 用动态规划思想解决变态青蛙跳台阶问题
author: carm
header-img: img/home-bg.jpg
categories: Algorithms
tag:
  - algorithm
  - java
---
# 动态规划(Dynamic Programming)
马上要考算法了，复习了动态规划的内容，现作出笔记，加深印象。

### 概述
1. 动态规划（DP）是通过组合子问题的解而解决整个问题。动态规划是一种规则，一种思考问题的方式，而不是指程序编码
2. 动态规划适用于子问题不是独立的情况，即各子问题包含公共的子子问题
3. 使用分治法则会重复的计算子问题，而动态规划对每个子问题只求解一次，将结果保存在一张表中，避免多次遇到各个子问题时重新计算结果。
4. 动态规划通常应用于最优化问题。
5. 动态规划通常基于一个递推公式及一个或多个初始状态；当前子问题的解由上一个子问题的解推出。

### 算法设计步骤（参考算法导论）
1. 描述最优解的结构
2. 递归定义最优解的值
3. 按自底向上的方式计算最优解值
4. 由计算的结果构建最优解

### 如何找状态转移方程
1. 找出问题的“状态”
2. 达到这种“状态”有几种方式
3. 写出状态转移方程（欲求问题的解，先求子问题的解）

### 实际应用
<p style="color: green"><span style='color:green;font-size:18px'>问题</span>：一只青蛙一次可以跳上1级台阶，也可以跳上2级……它也可以跳上n级。求该青蛙跳上一个n级的台阶总共有多少种跳法。</p>
<p>典型的动态规划问题，求解如下：</p>
目标：青蛙跳到第n级的台阶，总的跳法。
<p>假设：f(n)表示青蛙跳到n台阶的总跳法<span style="color:green">（问题的状态)</span>，则青蛙可从（n-1）级台阶直接跳上，也可从（n-2）,(n-3)···3,2,1,0级台阶直接跳上<span style="color:green">（即达到这状态有几种方式）</span>，故<code>f(n) = f(n-1)+f(n-2)+f(n-3)···+f(2)+f(1)+f(0)</code><span style="color:green">(状态方程）</span>。</p>
当 <code>n=1</code>时，只有一种跳法，f(1) = 1;<br>
当 <code>n=2</code>时，会有两种跳法，一次1阶或2阶，f(2) =f(1)+f(0);<br> 
当 <code>n=3</code>时，会有三种跳法，f(3) =f(2)+f(1)+f(0);即，第一次由2级台阶跳上；第二次由一级台阶直接跳上；第三次跨越3个台阶直接跳上。<br>
由于 <code>f(n-1) = f(n-2)+f(n-3)+···f(2)+f(1)+f(0)</code>，则状态方程化简为：<code>f(n)=2*f(n-1)</code>。<br/>
故可得递归式：
<pre>
	f(n) = 0 			(n=0)
	f(n) = 1 			(n=1)
	f(n) = 2*f(n-1) 	(n>=2)
</pre>

### Java实现
	public class DP {
     	public int JumpFloorII(int target) {
         	if(target <= 0) return 0;       
            int[] jump = new int[target + 1];            
            jump[1] = 1;                 
            for(int i = 2 ; i <= target ; i++){               
                jump[i] = 2 * jump[i - 1];
            }
            return jump[target];	
     	}
	}

### 参考资料
1. 《算法导论》
2.  [动态规划：从新手到专家](http://www.360doc.cn/article/8076359_289597587.html)
