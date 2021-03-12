---
layout: post
title:  "2021 Mar 12th Learning Diary"
categories: journal
tags: [reading]
toc: true
--- 
It was new. It was singular . It was simple. It must succeed! –H .Nelson
{: .message }

## LeetCode
[331. 验证二叉树的前序序列化](https://leetcode-cn.com/problems/verify-preorder-serialization-of-a-binary-tree/)
依次读取节点，通过栈维护前序遍历过程中节点应有空位数，最后空位数应恰好为0。
时间复杂度必然O(N)，空间复杂度是O(N)。其实也可以不用栈，单纯记录总空位数，这样就可以O(1)。

实现上，需要split原字符串输入，在遍历过程中需要注意多个合法前序序列合并成一个序列的情况。

### cpp code smell
> When indexing C++ containers, such as std::string, std::vector, etc, the appropriate type is the member typedef size_type provided by such containers.

遍历容器类时最好用其命名空间的size_type，e.g.
{% highlight cpp %}
for (std::string::size_type ind{}; ind != str.length(); ++ind) {
	// do something
}
{% endhighlight %}

string::npos 是一个指示size_type可以表示的最大值，通常被string的一些函数用作指示字符串结尾位置的参数，或者作为返回值指示错误与无匹配。
{% highlight cpp %}
// string search functions return npos if nothing is found
std::string s = "test";
if(s.find('a') == std::string::npos)
    std::cout << "no 'a' in 'test'\n";
         
// functions that take string subsets as arguments 
// use npos as the "all the way to the end" indicator
std::string s2(s, 2, std::string::npos);
std::cout << s2 << '\n'; // st

std::bitset<5> b("aaabb", std::string::npos, 'a', 'b');
std::cout << b << '\n'; // 00011
{% endhighlight %}

### review
根据遍历序列还原二叉树结构的经典问题，我们知道“前序 + 中序” 或 “后序 + 中序”是可以还原的。但是“前序 + 后序”却不能，因为我们不能确定空节点是左还是右。e.g.
> preorder:  1 2

> postorder: 2 1

而这道题目之所以可以根据preorder确定二叉树结构，是因为序列中包含了空节点的信息。类似的，我们也可以根据含空节点信息的postorder确定二叉树结构，倒着读取并先放右子树即可。但我们可以根据含空节点信息的infix order确定二叉树结构吗？

答案是不能。e.g. 
> infix order: ‘#’ ‘1’ ‘#’ ‘2’ ‘#’

无论是‘1’还是‘2’作为根节点，都可以得到这个中序序列，二叉树结构有ambiguity。
从某种程度上说，空节点信息的作用和中序序列的作用是一样的，所以前序与后序可以利用之还原二叉树结构，而中序却不可。


## A Tour of C++ 1st
阅读ch8-9，ch8讲I/O Stream，ch9讲容器。

### What's new

一个重载 << 与 >> 以进行流操作的例子
{% highlight cpp %}
struct Entry {
	string name;
	int number;
};

ostream& operator<<(ostream& os, const Entry& e){
	return os << "{\"" << e.name << "\", " << e.number << "}";
}

istream& operator>>(istream& is, Entry& e)
	// read { "name" , number } pair. Note: for matted with { " " , and }
{
	char c, c2;
	
	if (is>>c && c=='{' && is>>c2 && c2=='"') { // star t with a { "
		string name; // the default value of a string is the empty string: ""
		while (is.get(c) && c!='"') //anything before a " is part of the name
			name+=c;
		
		if (is>>c && c==',') {
			int number = 0;
			if (is>>number>>c && c=='}') { // read the number and a }
				e = {name ,number}; //assign to the entry
				return is;
			}
		}
	}
	is.state_base::failbit); // register the failure in the stream
	return is;
}
{% endhighlight %}

通过stringstream构建类型转换的桥梁，只要Source & Target Type	均有string representation，即重载 >> 与 <<，e.g.
{% highlight cpp %}
template<typename Target =string, typename Source =string>
Target to(Source arg)  // convert Source to Target
{
	stringstream interpreter;
	Target result;
	
	if (!(interpreter << arg)  // write arg into stream
	 || !(interpreter >> result) // read result from stream
	 || !(interpreter >> std::ws).eof()) //stuff left in stream?
		throw runtime_error{"to<>() failed"};
	
	return result;
}

int main() {
	auto x1 = to<string,double>(1.2);  	//very explicit (and verbose)
	auto x2 = to<string>(1.2); 		// Source is deduced to double
	auto x3 = to<>(1.2); 			// Target is defaulted to string; Source is deduced to double
	auto x4 = to(1.2);			// the <> is redundant;
						// Target is defaulted to string; Source is deduced to double
}
{% endhighlight %}

C++ I/O Stream 一直为人所诟病，有机会研究下原因。

容器类常常使用的一个内存分配技巧就是 placement new，即在已经分配的内存上构造对象(并非申请内存再构造)。
{% highlight cpp %}
// within any block scope...
{
    alignas(T) unsigned char buf[sizeof(T)];
    // Statically allocate the storage with automatic storage duration
    // which is large enough for any object of type `T`.
    T* tptr = new(buf) T; // Construct a `T` object, placing it directly into your 
                          // pre-allocated storage at memory address `buf`.
    tptr->~T();           // You must **manually** call the object's destructor
                          // if its side effects is depended by the program.
}                         // Leaving this block scope automatically deallocates `buf`.
{% endhighlight %}

[alignas specifier](https://en.cppreference.com/w/cpp/language/alignas)用于声明类或变量时，定义更严格(stricter means larger)的字节对齐，简单来说就是2的幂次但大于自然字节对齐。

> Some data types are dependent on the implementation. [wiki](https://en.wikipedia.org/wiki/Data_structure_alignment)

因为自然字节对齐implementation-define，如果程序运行在依赖字节对齐的平台上，自定义字节对齐将使代码more portable。


> Creaing a new hash function by combining existing hash functions using exclusive or (ˆ) is simple and often very effective.

自定义哈希函数时，利用好异或，e.g.
{% highlight cpp %}
struct Record {
	string name;
	int product_code;
};

struct Rhash { //a hash function for Record
	size_t operator()(const Record& r) const
	{
		return hash<string>()(r.name) ˆ hash<int>()(r.product_code);
	}
};

unordered_set<Record,Rhash> my_set; // set of Recoreds using Rhash for lookup
{% endhighlight %}

### So simple

> To preserve polymorphic behavior of elements, store pointers; §9.2.1.

实现多态，容器类里类型是基类指针or引用。

几个map容器小心使用operator[]，不管是否显式地赋值，只要出现[]都会插入value，首次即插入default value。因为这个潜在地插入，operator []只能作用于non-const对象。

### Conclusion
IO Stream实现自定类的string representation；容器类关于placement new应该是一个重要的实现技巧，其本质感觉类似缓存(局部性原理)。

## Reference

1. [c++ reference type size_t](https://en.cppreference.com/w/cpp/types/size_t)
2. [c++ reference string npos](https://en.cppreference.com/w/cpp/string/basic_string/npos)
3. [c++ reference new expression: placement new](https://en.cppreference.com/w/cpp/language/new#Placement_new)
4. [c++ reference object alignment](https://en.cppreference.com/w/cpp/language/object#Alignment)
5. [c++ reference alignas specifier](https://en.cppreference.com/w/cpp/language/alignas)

