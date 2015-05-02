##c++对象模型##
最近在看《深度探索c++对象模型》对于对象模型有了一点了解，做一个总结。以下的一些结论的实验见：https://github.com/xtestw/CPPObjModelTest

###单个对象模型###

对于一个单个对象而言，对象的内部结构是类似于一张表结构，依次存储着c++的对象内部数据，我们都知道，一个c++的类内部一般会包含下面的几个部分：

-  非静态成员变量
-  非静态成员函数
-  静态成员变量
-  静态成员函数
-  虚函数
-  友元函数
-  构造函数、析构函数、拷贝构造函数

对于一个简单的对象，将设我们定义类如下：

	class base{
		public:
			base();
			virtual ~base();
			int a,b;
			void f();
			static int c,d;
			static void f2();
			virtual void f3();
	};

不考虑继承的话，他们的存储结构会是这样的一个结构，如下图：
![](http://i.imgur.com/lVYnj2P.png)

其中，非静态数据成员，被放在对象的内部，而虚函数会放在对象的一个虚表中，对象在编译的时候会形成一个vptr的指针置于对象的内部，其指针指向这张虚表（考虑类的继承和多态，指向这张虚表的vptr的设置和重置会在构造函数、析构函数和拷贝函数中自动完成）。

对象在图对象表中的位置，是按照对象的申明顺序来排列的。比如a,b的申明，因为a先申明的，那么a就被先压栈，占据高地址。

而如图所见，不论虚函数是多少个，对象内部只有一个指针指向它，所以始终占一个指针大小的空间（32位机器下是4byte).

而对这个vptr在对象中的位置，不同的编译器的处理是不一样的，vc为代表的是将其放在了头部，而gcc等则是将其放在了对象的尾部，放在尾部是为了综合考虑与C的struct的兼容问题，而在头部，则是考虑继承之后的子类在调用vptr的方便性。(我个人更偏向于放在头部，因为这样在继承的时候更好理解也更方便自然),本文中的代码和模型是基于g++编译器做的实验，都是放在头部的。

还有一个问题，《深度探索c++对象模型》中，对这个1byte的类的type_info位置，他是说放在虚表的第一个位置，而其实g++中，下面这个实验，并不是放在虚表中的：
	
	#include <iostream>
	using namespace std;
	class base{
		public:
			virtual void f(){cout<<"f"<<endl;}
			virtual void f1(){cout<<"f1"<<endl;}
	};
	typedef void(*Fun)(void);
	int main(){
		base b;
		((Fun)*(int*)*(int*)(&b))();
		((Fun)*((int*)*(int*)(&b)+1))();
		return 0;
	}

上面的这段代码的输出是

	f
	f1

可见虚表的第一项并不是type_info。

###单继承对象模型###

######不考虑虚函数的单继承#####
在对象申明的过程中，如果从上级单继承了一个对象，那么对父类的成员变量的存储是怎样的一个结构呢，参看下面的代码

	class base{
		public:
			int a,b;
	};
	class derived:public base{
		public:
			int c,d;
	};

子类从父类继承过来的成员变量，无论是private,protected,public,也无论是通过private,protected还是public的方式继承过来的，其在子类中的对象中，都有内存空间来存储它，只是这些成员变量对子类的函数的可访问性的问题而已，不考虑非虚函数和静态变量这些（下面的没说明也是一样）没有存在类里的成员，类内部的结构应该是下面这种情况：

![](http://i.imgur.com/11kcdkI.png)

这个地方需要注意的是，如果基类的b不是int类型，而是char类型，而且c,d也都不是int类型，也都是char类型，我们sizeof(derived)的值是多少呢？！ 答案是12！<strong>因为在继承过来的时候，基类已经做了位对齐的处理，在b和c之间填充了3个空字节，</strong>所以，

	sizeof（derived)=sizeof(base)+1(c)+1(d)+2(padding);

######考虑虚函数的单继承#####

如果父类中，出现虚函数，子类中也出现虚函数，会是怎样的一个结构呢？！比如下面的代码：
	
	class base{
		public:
			int a,b;
			virtual void f();
	};
	class derived:public base{
		public:
			int c,d;
			virtual void f2();
	};

在实际的存储中的结构应该是这个样子的：
![](http://i.imgur.com/mnAVrDA.png)


###多继承对象模型###

上面讨论了单继承的对象模型，现在讨论一下多继承的对象模型，比如下面的代码：

	class base1{
		public:
			int a,b;
			virtual void f();
	};
	class base2{
		public:
			int c,d;
			virtual void f2();
	};
	class derived:public base1,public base2{
		public:
			int e,ff;
			virtual void f3();
	};

上面的代码，在类中的布局应该是下面的这样的：
![](http://i.imgur.com/ZXmsdm4.png)

在这种继承模式下，每个父类都会有一张自己的虚表，里面包含自己的虚函数，而派生类中自己的虚函数，则是放在第一个虚表中的，如果派生类重写了虚函数，那么会自动替换成派生类的虚函数。

###存在虚基类的对象模型###

对象继承中，涉及到虚基类的问题，对象继承链中，虚基类只会保存一个实例。如下面的代码：

	class base{
		public:
			int a,b;
			virtual void f();
	};
	class base1:public virtual base{
		public:
			int c,d;
			virtual void f1();
	};
	class base2：public virtual base{
		public:
			int e,ff;
			virtual void f2();
	};
	class derived:public base1,public base2{
		public:
			int g,h;
			virtual void f3();
	};

这份代码在实际的对象继承中应该是这个样子的布局：

![](http://i.imgur.com/P5WybyU.png)

可以发现虚继承过来的基类并不像之前那样放在最上面，而且其实是放在最下面的（事实上，在g++中，表中地址从上而下是变大的）,而每一个虚表都是指向的自己的虚函数，在继承类中，如果重写了这个虚函数函数，对应的虚表中的函数也是会被改成继承类中的虚函数的。



