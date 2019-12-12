## 1.inline内联函数##
![](https://i.imgur.com/CjYftqD.png)

1.inline关键字用来定义一个类的内联函数，引入它的主要原因是用它替代C中表达式形式的宏定义,解决一些频繁调用的小函数大量消耗栈空间（栈内存）的问题。

2.inline的使用是有所限制的，inline只适合涵数体内代码简单的涵数使用，不能包含复杂的结构控制语句例如while、switch，并且不能内联函数本身不能是直接递归函数（即，自己内部还调用自己的函数）。

3.inline函数仅仅是一个对编译器的建议，所以最后能否真正内联，看编译器的意思，它如果认为函数不复杂，能在调用点展开，就会真正内联，并不是说声明了内联就会内联，声明内联只是一个建议而已。

4.定义在类中的成员函数缺省都是内联的，如果在类定义时就在类内给出函数定义，那当然最好。如果在类中未给出成员函数定义，而又想内联该函数的话，那在类外要加上inline，否则就认为不是内联的。

## 2.构造函数初始化 ##
![](https://i.imgur.com/70WWLCi.png)

成员变量再构造函数中初始化中使用":"函数方式性能比直接在构造函数中赋值要快，建议使用这种方式。

## 3.构造函数重载注意事项 ##

![](https://i.imgur.com/OZzGtLc.png)

黄色标注的构造函数定义将出现问题，如果该函数与上面构造函数同时出现，在无参初始化该类对象时将产生冲突，因为第一个构造函数已经有参数默认初始化列表了，定义该类对象时可以不加入参数，这就产生了冲突。


## 4.常量成员函数 ##

![](https://i.imgur.com/NWsQ5HK.png)

在一个类中，如果成员函数中没有改变成员函数操作（例如get操作），那么建议在该方法声明处加入const关键字，如果不加入const关键字，那么c++编译器将认为该函数可能会修改类的成员变量。这样做有个好处，可以看到上图的"？！"一图，使用者利用了const关键字定义并通过构造函数初始化了一个complex类，这个类将不能被修改，只能读取属性。当使用者调用complex的real()或者imag()方法时，如果这两个方法在定义处没有加入const关键字，那么将报错。

##  5.参数传递和返回值使用const引用 ##
![](https://i.imgur.com/fSYQbDM.png)

函数参数传递，如果不需要改变参数值，建议使用const reference减小开销。

![](https://i.imgur.com/XETl4GC.png)

返回值建议使用引用，但如果返回引用的是该函数的一个指向堆局部变量指针，例如*ptr，那么不能使用引用，因为该局部变量在函数调用后就以及销毁了，引用可以认为是给当前对象换了个别名，如果当前对象已经被销毁，那么该“别名”也就失去了意义，不存在了。

当然，如下图所示，对于临时变量也不能返回引用：
![](https://i.imgur.com/zCKy5kk.png)
## 6.友元 ##
![](https://i.imgur.com/72ZMZHz.png)

相同类的各个实例对象互为友元，可以通过彼此的内部方法调用传入参数的内部私有成员变量。

## 7.运算符重载this指针 ##
![](https://i.imgur.com/UHP1Am5.png)

如上图所所示，在运算符重载过程中隐藏了this指针，该指针编译器会处理，使用者不能显式声明。

## 8.规范化代码一 ##
### complex.h ###

	#ifndef __MYCOMPLEX__
	#define __MYCOMPLEX__
	
	class complex; 
	complex&
	  __doapl (complex* ths, const complex& r);
	complex&
	  __doami (complex* ths, const complex& r);
	complex&
	  __doaml (complex* ths, const complex& r);
	
	
	class complex
	{
	public:
	  complex (double r = 0, double i = 0): re (r), im (i) { }
	  complex& operator += (const complex&);
	  complex& operator -= (const complex&);
	  complex& operator *= (const complex&);
	  complex& operator /= (const complex&);
	  double real () const { return re; }
	  double imag () const { return im; }
	private:
	  double re, im;
	
	  friend complex& __doapl (complex *, const complex&);
	  friend complex& __doami (complex *, const complex&);
	  friend complex& __doaml (complex *, const complex&);
	};
	
	
	inline complex&
	__doapl (complex* ths, const complex& r)
	{
	  ths->re += r.re;
	  ths->im += r.im;
	  return *ths;
	}
	 
	inline complex&
	complex::operator += (const complex& r)
	{
	  return __doapl (this, r);
	}
	
	inline complex&
	__doami (complex* ths, const complex& r)
	{
	  ths->re -= r.re;
	  ths->im -= r.im;
	  return *ths;
	}
	 
	inline complex&
	complex::operator -= (const complex& r)
	{
	  return __doami (this, r);
	}
	 
	inline complex&
	__doaml (complex* ths, const complex& r)
	{
	  double f = ths->re * r.re - ths->im * r.im;
	  ths->im = ths->re * r.im + ths->im * r.re;
	  ths->re = f;
	  return *ths;
	}
	
	inline complex&
	complex::operator *= (const complex& r)
	{
	  return __doaml (this, r);
	}
	 
	inline double
	imag (const complex& x)
	{
	  return x.imag ();
	}
	
	inline double
	real (const complex& x)
	{
	  return x.real ();
	}
	
	inline complex
	operator + (const complex& x, const complex& y)
	{
	  return complex (real (x) + real (y), imag (x) + imag (y));
	}
	
	inline complex
	operator + (const complex& x, double y)
	{
	  return complex (real (x) + y, imag (x));
	}
	
	inline complex
	operator + (double x, const complex& y)
	{
	  return complex (x + real (y), imag (y));
	}
	
	inline complex
	operator - (const complex& x, const complex& y)
	{
	  return complex (real (x) - real (y), imag (x) - imag (y));
	}
	
	inline complex
	operator - (const complex& x, double y)
	{
	  return complex (real (x) - y, imag (x));
	}
	
	inline complex
	operator - (double x, const complex& y)
	{
	  return complex (x - real (y), - imag (y));
	}
	
	inline complex
	operator * (const complex& x, const complex& y)
	{
	  return complex (real (x) * real (y) - imag (x) * imag (y),
				   real (x) * imag (y) + imag (x) * real (y));
	}
	
	inline complex
	operator * (const complex& x, double y)
	{
	  return complex (real (x) * y, imag (x) * y);
	}
	
	inline complex
	operator * (double x, const complex& y)
	{
	  return complex (x * real (y), x * imag (y));
	}
	
	complex
	operator / (const complex& x, double y)
	{
	  return complex (real (x) / y, imag (x) / y);
	}
	
	inline complex
	operator + (const complex& x)
	{
	  return x;
	}
	
	inline complex
	operator - (const complex& x)
	{
	  return complex (-real (x), -imag (x));
	}
	
	inline bool
	operator == (const complex& x, const complex& y)
	{
	  return real (x) == real (y) && imag (x) == imag (y);
	}
	
	inline bool
	operator == (const complex& x, double y)
	{
	  return real (x) == y && imag (x) == 0;
	}
	
	inline bool
	operator == (double x, const complex& y)
	{
	  return x == real (y) && imag (y) == 0;
	}
	
	inline bool
	operator != (const complex& x, const complex& y)
	{
	  return real (x) != real (y) || imag (x) != imag (y);
	}
	
	inline bool
	operator != (const complex& x, double y)
	{
	  return real (x) != y || imag (x) != 0;
	}
	
	inline bool
	operator != (double x, const complex& y)
	{
	  return x != real (y) || imag (y) != 0;
	}
	
	#include <cmath>
	
	inline complex
	polar (double r, double t)
	{
	  return complex (r * cos (t), r * sin (t));
	}
	
	inline complex
	conj (const complex& x) 
	{
	  return complex (real (x), -imag (x));
	}
	
	inline double
	norm (const complex& x)
	{
	  return real (x) * real (x) + imag (x) * imag (x);
	}
	
	#endif   //__MYCOMPLEX__

### complex_test.cpp ###

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
