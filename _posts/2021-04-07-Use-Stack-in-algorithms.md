---
layout: post
title:  "Use Stack in Algorithms"
categories: algorithm
tags: [cheatsheet, competitive-programming]
toc: true
--- 
At some point, something must have come from nothing.
{: .message }

在算法问题中我们经常使用单调栈解决一些问题，然而有时问题不同，单调栈维护方式也相应不同。

## 左右最近的小(大)值位置
一个数组arr, 考虑j在范围[0, i)中，用单调栈寻找距离i<strong>最近</strong>的j，且满足:
{% highlight cpp %} arr[j] < arr[i] {% endhighlight %}

我们通常使用将所有数组元素都入栈一次的维护型单调栈，
{% highlight cpp %}
// leftMost[i] 记录i左侧最近小值位置
for(int i = N - 1; i >= 0; --i) {
	while(!st.empty() && heights[st.top()] > heights[i]) {
		leftMost[st.top()] = i;
		// 此时若栈非空，可得到右侧最近 小于等于值 位置
		// 有时可以用来常数优化，减少再次遍历的开销
		st.pop();
	}
	st.emplace(i);
}
// pop all 
while(!st.empty()) {
	leftMost[st.top()] = -1; // no small value in the left
	st.pop();
} 
{% endhighlight %}

这是我们通常使用的单调栈，有时需要左右侧都处理记录一遍，有时可以技巧性地仅处理一遍。
右侧(正序处理)同理，大值同理。

例题，[84. 柱状图中最大的矩形](https://leetcode-cn.com/problems/largest-rectangle-in-histogram/) [42. 接雨水](https://leetcode-cn.com/problems/trapping-rain-water/)

## 左右最远的小(大)值位置

一个数组arr, 考虑j在范围[0, i)中，用单调栈寻找距离i<strong>最远</strong>的j，且满足:
{% highlight cpp %} arr[j] <= arr[i] {% endhighlight %}

这时使用的单调栈更像一种单调性记录，不是所有元素都要入栈一次，而是仅最小值入栈
{% highlight cpp %}
stack<int> st;
st.push(0);
for(int i = 1; i < N; ++i) {
	if(A[st.top()] > A[i])
		st.push(i);
}
// 栈的记录可以看作最小值的变化轨迹(分段变化)
int ret = 0;
for(int i = N - 1; i >= 0; --i) {
	while(!st.empty() && A[st.top()] <= A[i]) {
		// i - st.top() 为正，说明可以找到 满足条件的 arr[j] <= arr[i],
		// 为零，说明本身成为过最小值
		// 为负，应该被舍弃，去寻找下一个最小值，这里不会出现
		ret = max(ret, i - st.top());
		st.pop();
	}
	if(st.empty()) break;
}
{% endhighlight %}

右侧(正序处理)同理，大值同理。

例题，[962. 最大宽度坡](https://leetcode-cn.com/problems/maximum-width-ramp/) [1124. 表现良好的最长时间段](https://leetcode-cn.com/problems/longest-well-performing-interval/)
