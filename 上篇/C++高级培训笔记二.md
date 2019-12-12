## 1.拷贝赋值函数 ##
![](https://i.imgur.com/B6W7jnL.png)

如上图所示，在重载“=”赋值运算符时需要检查自我赋值，原因如下：

![](https://i.imgur.com/ymJ6YD2.png)

如果没有自我赋值检测，那么自身对象的m_data将被释放，m_data指向的内容将不存在，所以该拷贝会出问题。

## 2.探索new操作 ##
![](https://i.imgur.com/CoqboeP.png)

## 3.探索delete操作 ##
![](https://i.imgur.com/LF05Wt3.png)

## 4.探索创建对象的内存分配情况 ##
这里以两个类做出说明：

	class complex
	{
		public:
			complex (double r = 0, double i = 0)
			: re (r), im (i)
			{ }
			complex& operator += (const complex&);
			double real () const { return re; }
			double imag () const { return im; }
		private:
			double re, im;
			friend complex& __doapl (complex*,
			const complex&);
	};


	class String
	{
		public:
			String(const char* cstr = 0);
			String(const String& str);
			String& operator=(const String& str);
			~String();
			char* get_c_str() const { return m_data; }
		private:
			char* m_data;
	};

创建这两个对象后，编译器(VC)给两个对象分配内存如下：
![](https://i.imgur.com/1Ud7vVP.png)

左边两个是类complex在调试模式和release模式下的编译器内存分配。在debug模式下，编译器给complex对象内存插入了头和尾（红色部分），4*8 + 4大小的信息部分（灰色部分），绿色部分是complex对象实际占用的空间，计算后只有52字节，但VC以16字节对齐，所以52最近的16倍数是64，还应该填补12字节的空缺（青色pad部分）。对于release部分的complex对象，只添加了信息头和伟部分。string类的分析基本一样。

接下来看对于数组对象，VC编译器是如何分配的：
![](https://i.imgur.com/G4eXrRE.png)

类似的，编译器给对象增加了一些冗余信息部分，对于complex类对象，由于数组有三个对象，则存在8个double，然后编译器在3个complex对象前插入“3”用于标记对象个数。String类的分析方法也类似。

下面这张图说明了为何删除数组对象需要使用delete[]方法：
![](https://i.imgur.com/vNvDftQ.png)


## 5.String类

String.h

	#ifndef __MYSTRING__
	#define __MYSTRING__
	
	class String
	{
	public:                                 
	   String(const char* cstr=0);                     
	   String(const String& str);                    
	   String& operator=(const String& str);         
	   ~String();                                    
	   char* get_c_str() const { return m_data; }
	private:
	   char* m_data;
	};
	
	#include <cstring>
	
	inline
	String::String(const char* cstr)
	{
	   if (cstr) {
	      m_data = new char[strlen(cstr)+1];
	      strcpy(m_data, cstr);
	   }
	   else {   
	      m_data = new char[1];
	      *m_data = '\0';
	   }
	}
	
	inline
	String::~String()
	{
	   delete[] m_data;
	}
	
	inline
	String& String::operator=(const String& str)
	{
	   if (this == &str)
	      return *this;
	
	   delete[] m_data;
	   m_data = new char[ strlen(str.m_data) + 1 ];
	   strcpy(m_data, str.m_data);
	   return *this;
	}
	
	inline
	String::String(const String& str)
	{
	   m_data = new char[ strlen(str.m_data) + 1 ];
	   strcpy(m_data, str.m_data);
	}
	
	#include <iostream>
	using namespace std;
	
	ostream& operator<<(ostream& os, const String& str)
	{
	   os << str.get_c_str();
	   return os;
	}
	
	#endif

string_test.cpp

	#include "string.h"
	#include <iostream>
	
	using namespace std;
	
	int main()
	{
	  String s1("hello"); 
	  String s2("world");
	    
	  String s3(s2);
	  cout << s3 << endl;
	  
	  s3 = s1;
	  cout << s3 << endl;     
	  cout << s2 << endl;  
	  cout << s1 << endl;      
	}
