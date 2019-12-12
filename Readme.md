#include <iostream>
#include "complex.h"

using namespace std;

ostream&
operator << (ostream& os, const complex& x)
{
  return os << '(' << real (x) << ',' << imag (x) << ')';
}

int main()
{
  complex c1(2, 1);
  complex c2(4, 0);

  cout << c1 << endl;
  cout << c2 << endl;
  
  cout << c1+c2 << endl;
  cout << c1-c2 << endl;
  cout << c1*c2 << endl;
  cout << c1 / 2 << endl;
  
  cout << conj(c1) << endl;
  cout << norm(c1) << endl;
  cout << polar(10,4) << endl;
  
  cout << (c1 += c2) << endl;
  
  cout << (c1 == c2) << endl;
  cout << (c1 != c2) << endl;
  cout << +c2 << endl;
  cout << -c2 << endl;
  
  cout << (c2 - 2) << endl;
  cout << (5 + c2) << endl;
  
  return 0;
}

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


## C++中的explicit ##

C++中， 一个参数的构造函数(或者除了第一个参数外其余参数都有默认值的多参构造函数)， 承担了两个角色。 1 是个构造器 ，2 是个默认且隐含的类型转换操作符。

所以， 有时候在我们写下如 AAA = XXX， 这样的代码， 且恰好XXX的类型正好是AAA单参数构造器的参数类型， 这时候编译器就自动调用这个构造器， 创建一个AAA的对象。

这样看起来好象很酷， 很方便。 但在某些情况下（见下面权威的例子）， 却违背了我们（程序员）的本意。 这时候就要在这个构造器前面加上explicit修饰， 指定这个构造器只能被明确的调用/使用， 不能作为类型转换操作符被隐含的使用。

explicit构造函数是用来防止隐式转换的。请看下面的代码：

	class Test1
	{
	public:
	    Test1(int n)
	    {
	        num=n;
	    }//普通构造函数
	private:
	    int num;
	};
	class Test2
	{
	public:
	    explicit Test2(int n)
	    {
	        num=n;
	    }//explicit(显式)构造函数
	private:
	    int num;
	};
	int main()
	{
	    Test1 t1=12;//隐式调用其构造函数,成功
	    Test2 t2=12;//编译错误,不能隐式调用其构造函数
	    Test2 t2(12);//显式调用成功
	    return 0;
	}

Test1的构造函数带一个int型的参数，代码23行会隐式转换成调用Test1的这个构造函数。而Test2的构造函数被声明为explicit（显式），这表示不能通过隐式转换来调用这个构造函数，因此代码24行会出现编译错误。

普通构造函数能够被隐式调用。而explicit构造函数只能被显式调用。

## 2.虚函数 ##

![](https://i.imgur.com/rLgBzuh.png)





## 1.转换函数 ##

![](https://i.imgur.com/D2CbMvC.png)
任何Fraction需要被转换为double类型的时候，自动调用double()函数进行转换。如上图所示，编译器在分析double d = 4 + f过程中判断4为整数，然后继续判断f，观察到f提供了double()函数，然后会对f进行double()操作，计算得到0.6，再与4相加，最后得到double类型的4.6。

## 2.explicit与隐式转换 ##

![](https://i.imgur.com/6hPhBVv.png)
上图中定义了一个类，叫Fraction，类里面重载了“+”运算符，在f+4操作过程中，“4”被编译器隐式转换（构造函数）为Fraction对象，然后通过Fraction重载的“+”运算符参与运算。

![](https://i.imgur.com/CJMA6oJ.jpg)
如上图所示，在Fraction中增加了double()函数，将Fraction的两个成员变量进行除法运算，然后强制转化为double类型并返回结果，在f+4重载过程中编译器将报错，可以做出如下分析：

1、首先4被隐式转化（构造函数）为Fraction对象，然后通过重载的“+”运算符与“f”进行运算返回一个Frction对象；

2、首先4被隐式转化（构造函数）为Fraction对象，然后通过重载的“+”运算符与“f”运算后对进行double运算，最返回一个Frction对象；

3、。。。

所以编译器有至少两条路可以走，于是产生了二义性，报错。
![](https://i.imgur.com/fZU3aYV.png)
如上图所示，在构造函数Franction前加入explict关键字，隐式转换将取消，所以在执行d2 = f + 4过程中，f将调用double函数转换为0.6，然后与4相加变成4.6，由于构造函数取消隐公式转换，4.6无法转换为Fraction，于是将报错。

下图为C++ stl中操作符重载和转换函数的一个应用：
![](https://i.imgur.com/01jFo9x.png)

## 3.智能指针 ##

下面这张图很好地说明了智能指针的内部结构和使用方法：
![](https://i.imgur.com/UenFAl5.jpg)
智能指针在语法上有三个很关键的地方，第一个是保存的外部指针，对应于上图的T* px，这个指针将代替传入指针进行相关传入指针的操作；第二个是重载“*”运算符，解引用，返回一个指针所指向的对象；第三个是重载“->”运算符，返回一个指针，对应于上图就是px。

迭代器也是一种智能指针，这里也存在上面提到的智能指针的三个要素，分别对应于下图的红色字体和黄色标注部分：
![](https://i.imgur.com/Gsty5Hz.jpg)

下面将仔细分析迭代器重载的“*”和“->”重载符：

![](https://i.imgur.com/8TiNyqF.jpg)

创建一个list迭代器对象，list<Foo>::iterator ite;这里的list用于保存Foo对象，也就是list模板定义里的class T，operator\*()返回的是一个(\*node).data对象，node是\_\_link\_type类型，然而\_\_link\_type又是\_\_list\_node<T\>\*类型，这里的T是Foo，所以node是\_\_list\_node<Foo\>*类型,所以\(\*node\).data得到的是Foo类型的一个对象，而&\(operator\*\(\)\)最终得到的是该Foo对象的一个地址，即返回Foo\* 类型的一个指针。

## 4.仿函数 ##

![](https://i.imgur.com/DhX59Kd.jpg)
重上图可以看到，每个仿函数都是某个类重载“()”运算符，然后变成了“仿函数”，实质还是一个类，但看起来具有函数的属性。每个仿函数其实在背后都集成了一个奇怪的类，如下图所示，这个类不用程序员手动显式声明。
![](https://i.imgur.com/EKpcd2X.jpg)
标准库中的仿函数也同样继承了一个奇怪的类：
![](https://i.imgur.com/GqWy1Yz.jpg)
这个类的内容如下图所示，只是声明了一些东西，里面没有实际的变量或者函数，具体的内容将在STL中讨论。
![](https://i.imgur.com/G5iodMK.png)

## 5.namespace ##

![](https://i.imgur.com/J2k7KGj.png)

## 6.类模板 ##

![](https://i.imgur.com/j1zJLWl.png)

## 7.函数模板 ##

![](https://i.imgur.com/J5MLXNd.png)
与类模板不同的是，函数模板在使用是不需要显式地声明传入参数类型，编译器将自动推导类型。

## 8.成员模板 ##

![](https://i.imgur.com/Y9FBR6k.png)
成员模板在泛型编程里用得较多，为了有更好的可扩展性，以上图为例，T1往往是U1的基类，T2往往是U2的基类，可以看下面这个例子：
![](https://i.imgur.com/FVN4ww1.jpg)
通过这种方法，只要传入的U1和U2的父类或者祖类是T1和T2，那么通过这样的方式可以实现继承和多态的巧妙利用，但反之就不行了。这样的方式在STL中用得很多：
![](https://i.imgur.com/7ig5lAg.png)

## 9.模板偏化 ##

正如其名，模板偏化指的是模板中指定特定的数据类型，这和泛化是不同的：
![](https://i.imgur.com/maGjkzC.png)
当然，模板偏化也有程度之分，可以部分类型指定，称之为偏特化：
![](https://i.imgur.com/1qfrg4O.png)
![](https://i.imgur.com/ixantUB.png)

## 10.模板模板参数 ##

![](https://i.imgur.com/TUIdZpP.png)
![](https://i.imgur.com/wulBSIu.png)
![](https://i.imgur.com/sUNnTCc.png)

## 11.可变数目模板参数 ##

![](https://i.imgur.com/YqS87h4.jpg)
过多内容将在C++11课程中讲解，这里暂时只做介绍。

## 12.auto关键字和增强型for循环

![](https://i.imgur.com/C3Gmlaa.png)
![](https://i.imgur.com/HwUMzfD.png)
过多内容将在C++11课程中讲解，这里暂时只做介绍

## 13.reference ##

reference可以看做是某个被引用变量的别名。
![](https://i.imgur.com/pYlpmg2.png)
![](https://i.imgur.com/3rfj4at.png)
![](https://i.imgur.com/iVYlytf.jpg)

## 14.虚指针和虚函数表 ##

![](https://i.imgur.com/nIkKZhP.jpg)
如上图所示，定义了三个类，A、B和C，B继承于A,C继承于B，A中有两个虚函数，B中有一个，C中也有一个。编译器将A的对象a在内存中分配如上图所示，只有两个成员变量m\_data1和m\_data2，与此同时，由于A类有虚函数，编译器将给a对象分配一个空间用于保存虚函数表，这张表维护着该类的虚函数地址（动态绑定），由于A类有两个虚函数，于是a的虚函数表中有两个空间（黄蓝空间）分别指向A::vfunc1()和A::vfunc2()；同样的，b是B类的一个对象，由于B类重写了A类的vfunc1()函数，所以B的虚函数表（青色部分）将指向B::vfunc1()，同时B继承了A类的vfunc2()，所以B的虚函数表（蓝色部分）将指向父类A的A::vfunc2()函数；同样的，c是C类的一个对象，由于C类重写了父类的vfunc1()函数，所以C的虚函数表（黄色部分）将指向C::vfunc1()，同时C继承了超类A的vfunc2()，所以B的虚函数表（蓝色部分）将指向A::vfunc2()函数。同时上图也用C语言代码说明了编译器底层是如何调用这些函数的，这便是面向对象继承多态的本质。

## 15.this指针 ##

![](https://i.imgur.com/PH9T0ss.jpg)
this指针其实可以认为是指向当前对象内存地址的一个指针，如上图所示，由于基类和子类中有虚函数，this->Serialize()将动态绑定，等价于(*(this->vptr)[n])(this)。可以结合上节虚指针和虚函数表来理解，至于最后为什么这样写是正确的，下面小结将会解释。

## 16.动态绑定 ##

![](https://i.imgur.com/C97NUFQ.jpg)
![](https://i.imgur.com/lDi3V3I.jpg)

## 17.重载delete和new操作符 ##

![](https://i.imgur.com/h5tWgeY.png)
![](https://i.imgur.com/EYF1OCI.png)
![](https://i.imgur.com/XZg8al9.png)
![](https://i.imgur.com/rqX4a3j.jpg)
![](https://i.imgur.com/WtkG1Kv.jpg)
![](https://i.imgur.com/EUhdjmM.jpg)

## 18.重载delete()和new()操作符 ##

![](https://i.imgur.com/c8jAMdu.png)
![](https://i.imgur.com/Sd2vMWM.jpg)
![](https://i.imgur.com/3Mgirvx.jpg)
