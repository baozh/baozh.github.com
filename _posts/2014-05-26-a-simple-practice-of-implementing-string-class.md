---
layout: post
title: "A Simple Practice of Implementing String Class"
date: 2014-05-26 08:20
comments: true
categories: Software_development C/C++
tags: string sso eager cow 
---


Recently I read chenshuo's blog, [《A proper implementation of string class in C++ interview》][link1]，the article shows beautiful implementation of a minimalist string class, but not support some complex operator,such as [],==,<，no exception handing. So I try to implement a string class, supplement some remaining function, including exception handing and testing.

<!--more-->

###  Requirements

The specific requirements are these:

1. Support conversion from/to traditional c string.

	* string object -> c_str()
	* c string -> string object

2. Support [] operator, modify a character in a string object.
3. Overloaded operator =, ==, >, <, +, +=.
4. Some handy function, such size(),empty().
5. String object can use as value type of standard container,such as vector/list/deque.
6. String object can use as key type of map/unordered_map.
7. Support exception handing: it's best to specify a custom function to handing `new` failure, when allocating string space.
8. Support testing: Write test case(use google test, or Boost test framework) to test all kinds of unexpected circumstances.

###  Design

1. when the system unable to allocate the request memory, how to handling?  
   Generally, there are two ways to haneling this situation.
   - use "nothrow" form of new operator, write codes like this:

			Widget *pwd = new (std::nothrow) Widget;
			if (pwd == NULL)
			{
				...
			}

   - set a handler to handling allocating failure.  
   Befor operator new throws an exception in response to an unsatisfiable request for memory, it calls a client-specifiable error-handling function called a new-handler. You can call `set_new_handler` function to set it.


			void outOfMen()
			{
				std::cerr << "Unable to satisfy request for memory.";
				std::abort();
			}

			int main()
			{
				std::set_new_handler(outOfMen);
				int *pBigDataArray = new int[100000000L];
				...
			}

	In this place, I write a class `NewHandlerHolder`，inherited by string class to set a class-specfic new-handler. when dynamically allocating string object failed, it will call new-handler setted in string class.

2. string class support value type, key type of standard container.  
   Sequential container(such as vector, deque, list,etc.) can keep any type objects, but some container operation requires that value type class implement some operator, such as `<` operator for sort operation. For the associative containers(map, multimap, set, and multiset), the key type must define a way to compare the elements. By default, the library uses the < operator for the key type to compare the keys. In the set types, the key is the element type; in the map types, the key is the first type.   

3. testing  
   Use boost test framework to test every function in string class. Write test cases to test all kinds of success or failure.

###  Implementation  
Generally, there are three way to  implement string class:

1. Eager Copy  
   This has no special processing. The code skeleton like this:  

		class string
		{
		public:
			char *begin() {return start;}
			char *end()	{return start+size;}
			size_t size() const {return size;}	
			size_t capacity() const {return capacity;}
			...
		private:
			char *start;
			size_t size;
			size_t capacity;
		};

   Data structure schematic diagram like this:  
   ![aaa][pic1]

2. SSO(short-string-optimized)  
   It has local buffer to store short string, if the string too large to keep in local buffer, it will store it in a dynamic memory space addressed by `start` pointer.The code skeleton like this:

		class string
		{
		public:
			//some access function
			...
		private:
			char *start;
			size_t size;
			static const int kLocalSize = 15;
			union
			{
				char buffer[kLocalSize+1];
				size_t capacity;
			}data;
		};

   The memory layout like this:  
   ![bbb][pic2]

   If the string too large to store in the buffer, it will dynamically allocate a space in heap pointed by start to store string.  
   ![ccc][pic3]

3. COW(Copy-On-Write)  
   Copy a string with no modification, it will actually not copy all char string, just increase it's referance count. While copy a string with some modification, or directly modify one more char in string, it will reallocate a new space to store new string, and revise referance count.The code skeleton like this:

		class string
		{
		public:
			//some access function
			...
		private:
			struct Rep
			{
				size_t size;
				size_t capacity;
				size_t refcount;
				char *data[1];
			};
			char *start;
		};

   The memory layout like this:  
   ![ddd][pic4]

   Andrei Alexandrescu advise we should choose right version of string for different application Scenarios. For short strings, use SSO version; For strings of medium length, use Eager Copy version; For large strings, use COW version. You can profiling to decide s to use which version of string.

   I implement Eager Copy version, and SSO version string, the code is in [file1][link2], [file2][link3].


[pic1]: /assets/images/string1.png
[pic2]: /assets/images/string2.png
[pic3]: /assets/images/string3.png
[pic4]: /assets/images/string4.png

[link1]: http://coolshell.cn/articles/10478.html
[link2]: https://github.com/baozh/string/blob/master/include/string_eager.h
[link3]: https://github.com/baozh/string/blob/master/include/string_sso.h

