# Effective C++

## 条款1：视C++为一个语言联邦

可将C++视为4个主要的次语言：  
- C。
- Object-oriented C++。
- Template C++。
- STL。
  

在某个次语言中，各个守则与通例都相对简单易懂，容易记住。



## 条款2：尽量以const，enum，inline替换#define
（更愿意用编译器，而不是预处理器)  
~~~
#define ASPECT_RATIO 1.653
~~~
上述宏的缺点:  
- ASPECT_RATIO很有可能未被编译器看见，也未进入记号表，发生错误视很难追踪到ASPECT_RATIO（错误信息可能只会提到1.653）

建议采用常量或enum替换上述宏
~~~c++
const double AspectRatio =1.653；
enum {Max=1000}；
~~~

- #define不仅不能用来定义class专属常量，也不能提供任何封装性；
~~~c++
class GamePlayer{
private：
    static const int NumTurns=5；
    int score[]；
}
~~~
注：该NumTurns是一个声明式而非定义式。若一个C++的class专属常量既是static且为整数类，只要不取它的地址，就可以声明并使用它，而无需定义。否则要在实现文件中加上下面一条语句：
~~~c++
const int GamePlayer::NumTurns;
~~~
- 可复用的代码模板（采用内联模板函数template inline function而不是宏macros）

  - macros虽然可以避免函数调用带来的开销，但其仅仅是代码替换，相对于普通函数会有不可预料的行为。

  - template inline function像一般函数有可预料的行为，且拥有类型安全性。
  
    ~~~
    template <typename T>
    inline void callWithMax（const T& a,const T& b）
    {
    	f(a>b?a:b);
    }
    ~~~
    
    注：inline保证其也不会有函数调用带来的开销，拥有与宏同样的效率。
    

## 条款3：尽可能使用const

- STL的迭代器是对普通指针的抽象，作用应类似与T*指针，声明迭代器为const常量类似于表示指针本身为常量，表明迭代器不能指向其它东西。

  所以，STL专门定义了const_iterator类，来模拟const T*指针，即不希望通过该迭代器修改其指向的东西。

- const成员函数的好处：

  1. 容易理解类的接口，即公有成员函数是否能够修改对象本身。

  2. 可以用来操作const对象。该类有足够的const成员函数时，类对象可以放心pass  by reference to const来提高效率，不用担心传递之后，无法对对象引用进行操作。

- 两个成员函数只是常量性不同，则可以被重载。

- 如何解决const和non-const版本函数的代码重复？

  答：non-const版本的函数调用const版本的函数，中间需要两次常量性的转型，代码如下：

  ~~~c++
  class TextBlock
  {
  	public:
	  ...
  	  const char& operator[](std::size_t position) const
  	  {
  	  	...
  	  	...
  	  	...
  	  	return text[position];
	  }
	  char& operator[](std::size_t position)
	  {
	  	return 
	  	  const_cast<char&>(
	  	  	static_cast<const TextBlock&>(*this)[position]
	  	  );
	  }
  
	}
  ~~~
  
- mutable(可变的)修饰符可用于修饰成员变量，这样即使在const成员函数中也可以修改这些变量，从而实现
  
  logical constness，即一个const对象的部分成员变量是可以修改的，对象只需符合逻辑上的常量性，不一定非要bitwise constness。
  
##  条款4：确定对象使用前已被初始化

- 如何解决“定义与不同的编译单元内的non-local static对象”的初始化相对次序？

  答：将每个non-local static 对象搬到自己的专属函数内（该对象在此函数内被声明为 static)，这些函数返回一个 reference 指向它所含的对象。
  

根本：C++保证，函数内的 local static 对象会在该函数被调用期间初始化，另外，若从未调用此函数，就不会有该对象的构造和析构成本。

建议：手工初始化内置型non-member对象；使用成员初始列初始化对象的所有成分。

## 条款5：了解C++默认生成并调用了哪些函数

- 在下面三种情况下，编译器不会自动生成拷贝赋值函数：

1.内含引用类型成员的类：首先，C++不允许让reference指向其它对象；第二，直接改变引用的值不一定符合赋值的思想。

2.内含const成员的类：更改const成员不合法。

3.基类的拷贝赋值函数被声明为private：子类的拷贝赋值函数无法调用基类的拷贝赋值函数，也就无法处理对象中基类的成分。

## 条款6：若不想使用编译器自动生成的函数，就该明确拒绝

- 若不想使用编译器自动生成的函数（如拷贝构造函数和拷贝赋值函数），可以将其声明为private，这样类外和非友元函数都不可以调用该函数；此外，不去定义这些函数，就会在构建时报出连接错误。

  **注：C++11以后，可以使用=delete告知编译器不去自动生成这些函数。**

## 条款7：为多态基类声明virtual析构函数

- 若基类的析构函数没有声明为virtual，而一个派生类的对象经由一个基类类的指针去delete，该对象中属于子类的部分通常不会被正确销毁。

  心得：只有当class内含至少一个virtual函数，才为它声明virtual函数。

- STL中的string和所有容器（vector，list，set等等）都有着非虚的析构函数，我们不应该去继承它们。

  **注：C++11以后引入final关键字，可以用来阻止类进行派生**，但上述的STL类却没有添加上，依旧可以进行派生。

- 实践：设计一个抽象基类时，可以将其的析构函数设计为纯虚函数。

  注：该析构函数仍需要定义（空），因为编译器会在派生类的析构函数中调用该析构函数，不定义会发生链接错误。

## 条款8：别让异常逃离析构函数

- 不应该让析构函数抛出异常。

- 可以通过try，catch机制捕获析构函数内异常，然后结束程序或不做任何事。

  ~~~C++
  DBConn::~DBConn()
  {
  	try{db.close();}
  	catch(...)
  	{
  		制作运转记录，记下对close的调用失败。
  		std::abort();//去掉此行则是不做任何事的做法。
  	}
  }
  ~~~

## 条款9：绝不在构造和析构过程中调用virtual函数

- 问题：在基类的构造期间，虚函数不再是虚函数，派生类的构造函数并不会正确调用派生类版本的虚函数。

- 原因：构造对象时，基类的构造函数会先调用，此时派生类中成员变量并没有初始化，动态调用派生类的函数会造成不明确的行为（这些函数通常会取用成员变量）。

  根本原因：在派生类对象的基类部分构造期间（调用基类构造函数），对象的类型是基类。

- 一种解决方案：将构造函数中的虚函数改为非虚函数，将特定的信息通过参数传递给该函数,在派生类中则将该参数传递给基类的构造函数。

## 条款10：令operator=返回一个reference to *this

~~~c++
class Widget
{
public:
	...
	Widget& operator=(const Widget& rhs)
	{
		...
		return *this;
	}
}
~~~

## 条款11：在operator=中处理“自我赋值”

方案1：证同测试

~~~C++
Widget& Widget::operator=(const Widget& rhs)
{
	if(this==&rhs) return *this;//直接返回，不创建新对象。
    
    delete pb;
    pb=new Bitmap(*rhs.pb);
    return *this;
}
~~~

- 这种方法解决了自我赋值的安全性，但仍有着异常相关问题，new申请内存可能会申请失败。

方案2：copy and swap

~~~c++
class Widget
{
	...
	void swap(Widget& rhs);
	...
}
Widget& Widget::operator=(const Widget& rhs)
{
    Widget temp(rhs);
    swap(temp);
    return *this;
}
~~~

## 条款12：复制对象是勿忘其每一个成分

1. 拷贝函数应该确保复制对象内的所有成员，包括所有的基类成分（调用基类的拷贝函数）。

2. 不要尝试在某个拷贝函数中调用另一个拷贝函数，应创建一个新的private函数（通常命名为init），将相同功能的代码放置在这里，以供拷贝函数调用。

## 条款13：以对象管理资源（RAII）

- 关键想法
  1. 获取资源后立刻放进管理对象（例如智能指针）。
  2. 管理对象运用析构函数确保资源被释放。

## 条款14：在资源管理类中小心copying行为

- 措施
  1. 将拷贝函数delete掉，阻止拷贝。
  2. 采用引用计数法处理拷贝（share_ptr)。

## 条款15：在资源管理类中提供对原始数据的访问

1. 显式访问：返回底层原始指针。（例如share_ptr.get() ）

2. 隐式访问：提供隐式转换函数，返回底层原始指针。

   ~~~c++
   class Font
   {
   public:
       ...
       operator Fontandle() const //隐式转换函数
   	{return f;}
       ...
   };
   ~~~

## 条款16：成对使用new和delete是要采用相同形式

- 用new在堆中分配对象数组，一定要使用delete [] 来释放指针指向的资源。

## 条款17：以独立语句将newed对象置入智能指针

## 条款18：让接口容易被正确使用，不易被误用

- 预防误用接口的方法包括：建立新的类型，限制类型上的操作，束缚对象值，以及消除客户的资源管理责任。

- shared_ptr可以定制删除器。

## 条款19：设计class犹如设计type

## 条款20：宁以pass-by-reference-to-const替换pass-by-value

- 若将一个子类的对象以**值传递**的方式传递给一个以父类对象为参数的函数，子类对象的特化性质会被切割掉，最后成为一个父类对象。（所以需要用引用或指针调用完成虚函数的调用）

- 对于内置类型以及STL的迭代器和函数对象，pass-by-value比较合适。

## 条款21：必须返回对象是，别妄想返回器reference

- 绝对不要返回pointer或reference指向一个局部栈对象，或返回reference指向一个堆中的对象，或返回pointer或reference指向一个local static对象。

## 条款22：将成员变量声明为private

- 通过成员函数访问成员变量的好处：
  1. 客户访问数据的一致性（通过函数）。
  2. 可通过函数细微划分访问权限（可读，可写，只读等）
  3. 封装。外部接口不变，内部实现可以自由变化。

## 条款23：宁以non-member，non-friend函数替换member函数

- 非成员，非友元的函数相比与具有相同功能的成员函数（前者通常需要传入对象的引用或指针，借由对象的public接口完成功能），对类来说具有更高的封装性。因为前者不能访问对象的private成员变量。

- 具体实践：将类的定义和相关的non-member函数放在同一个命名空间中，但是最好将一类的non-member函数放在另一个头文件中，这样可以避免函数与函数，函数与类之间的编译相依关系。

  注：namespace与classes不同，前者能跨越多个源码文件而后者不能。

  

## 条款24：若所有参数皆需类型转换，请为此采用non-member函数

注：该类型转换指的是构造函数的隐式类型转换。

## 条款25:考虑写一个不抛异常的swap函数

- 若swap缺省实现版对class或class template的效率不足（你很可能使用了pimpl技术），以下是建议。

  1. 提供一个 public swap 成员函数，让它高效地置换你的类型的两个对象值。

  2. 在你的 class template 所在的命名空间内提供一个 non-member swap, 并令它调用

     上述 swap 成员函数。

  3. 如果你正编写一个 class （而非 class template) ，为你的 class 特化 七d:: swap 。并

     令它调用你的 swap 成员函数。

  最后当你调用 swap时, 确保包含一个 using 声明式，以便让 std: :swap 在你的函数内曝光可见，然后不加任何 namespace 修饰符，赤裸裸地调用 swap。

  注：成员版 swap 绝不可抛出异常。原因：swap 的一个最好的应用是帮助 classes (和 class templates) 提供强烈的异常安全性保障。

  

## 条款26：尽量延后变量定义式的出现时间

- 延后变量的定义直到使用该变量的前一刻。好处：可以避免构造和析构非必要的对象。

进阶：延后变量的定义直至能够获得初值实参为止。好处：可以避免无意义的default构造行为。

## 条款27：尽量少做转型动作

- const_cast:通常用来将对象的常量性移除。
- dynamic_cast：执行“安全向下转型”，即用来决定某对象是否归属于继承体系中的某个类型。
- reinterpret_cast：执行低级转型（C语言风格转型），重新解释该内存。
- static_cast：用来强迫隐式转换。

## 条款28：避免返回handles指向对象内部成分

- handle（包括reference，指针，迭代器）

## 条款29：为“异常安全”而努力是值得的

- 异常安全函数的特点：

  1. 不泄露任何资源。
  2. 不允许数据被破坏。

- 异常安全函数有三种类型的保证：
  1. **基本承诺**：如果异常被抛出，程序内的任何事物仍然保持在有效状态下。
  2. **强烈保证**：如果异常被抛出，程序状态不变。（例如copy and swap）
  3. **nothrow保证**：承诺绝对不抛出异常。（作用于内置类型的操作）

## 条款30：透彻理解inline的里里外外

- inline最好用来修饰那些小型，被频繁调用的函数。

## 条款31：将文件的编译依存关系降至最低

- 如果可以用对象的引用或指针来完成成员变量的定义，就不要直接在类中定义对象甭说。

- 尽量以class声明式替换class定义式。(方式：Handle classes和Interface classes)

- 为声明式和定义式提供不同的头文件。

## 条款32：确定你的public继承塑造出is-a关系

## 条款33：避免覆盖继承而来的名称

- derived classes 内的名称会遮掩 base classes 内的名称。例如：子类的函数名会覆盖掉父类所有相同的函数名（包括重载函数）

- 解决方案：using声明式和转交函数。

## 条款34：区分接口继承和实现继承

- 声明一个 纯虚函数的目的是为了让 derived classes 只继承**函数接口**。
- 声明一个普通虚函数的目的是让 derived classes 继承该**函数的接口**和**缺省实现**。
- 声明非虚函数的目的是为了令 derived classes 继承**函数的接口**和**强制性实现**。

## 条款35：考虑virtual函数以外的其他选择

- NVI手法允许 derived classes 重新定义**private/protected** virtual 函数，从而赋予它们“如何实现机能”的控制能力，但 base class 保留诉说“函数何时被调用”的权利。


总结：

1. virtual 函数的替代方案包括 NVI 手法及 Strategy 设计模式的多种形式。 
2. 将功能从成员函数移到 class 外部函数的一个缺点是非成员函数无法访class non-public 成员。

## 条款36：绝不重新定义继承而来的non-virtual函数

## 条款37：绝不重新定义继承而来的缺省参数值

- virtual函数是动态绑定的，而缺省参数值却是静态绑定的。

## 条款38：通过复合塑造出has-a或“根据某物实现出”

## 条款39：明智而谨慎地使用private继承

- 如果classes之间的继承关系是private，编译器不会自动将一个derived class对象转化为一个base class对象。
- Private 继承意味 implemented-in-terms-of（根据某物实现出），主要用于当一个可能成为 derived class 的类想访问另一个类的 protected 成分，或为了重写一个或多个 virtual 函数。
- private 继承可以造成 empty base 最优化。（即继承一个空类不会占用多余的空间，而复合一个空类则会多占用至少1个字节的空间，例如对象内存空间对齐）

## 条款40：明智而谨慎地使用多重继承

- virtual继承的好处：解决多继承时命名冲突和冗余数据的问题（菱形继承），虚继承可以使得在派生类中只保留一份间接基类的成员。
- virtual继承的缺点：virtual继承类的体积大，访问virtual base classes的成员变量时速度较慢。最麻烦的是”virtual base 的初始化责任是由继承体系中的最低层class 负责“（而不是由直接继承的class负责），所以当virtual base class不带任何数据成员（类似于java和.Net中的Interface），实用价值最高。

- 多重继承的一个合理的应用：将 “public 继承自某接口”和 “private 继承自某实现“结合在一起。

## 条款41：了解隐式接口和编译期多态

- “以不同的template参数具体化function templates”会导致调用不同的函数，这就是所谓的编译期多态。
- 隐式接口（函数模板）相比与普通函数，并不基于函数签名式，而是有效表达式。

- classes和templates 都支持接口 (interfaces) 和多态 (polymorphism)。
  1. 对 classes 而言接口是显式的 (explicit) ，以函数签名为中心。多态则是通过 virtual函数发生于运行期。
  2. 对 template 参数而言，接口是隐式的 (implicit) ，奠基于有效表达式。多态则是通过 template 具体化和函数重载解析 (function overloading resolution) 发生于编译期。

## 条款42：了解typename的双重意义

- 默认情况下，嵌套从属名称不是类型，需要在类型前添加typename关键字。
- typename不可以出现在类的基类列中，也不可以在成员初始列中修饰base class。

## 条款43：学习处理模板化基类内的名称

- 问题：编译器往往拒绝在templatized base classes 内寻找继承而来的名称。

- 解决方案：在derived class templates 内通过 “this->" 指涉 base class templates 内的成员

  名称，假设该成员被继承；通过using使用base class的命名空间或直接使用命名空间指定成员。

## 条款44：将参数无关的代码抽离template

- 因非类型模板参数 (non-type template parameters) 而造成的代码膨胀往往可消除，做法是以函数参数或 class 成员变量替换 template 参数。
- 因类型参数 (type parameters) 而造成的代码膨胀往往可降低，做法是让带有完全相同二进制表述 (binary representations) 的具现类型 (instantiation types) 共享实现码。

## 条款45：运用成员函数模板接受所有兼容类型

- 若声明 member templates 用于“泛化 copy 构造”或“泛化copy assignment “，还需要声明正常的 copy 构造函数和 copy assignment 操作符。因为即使前者已经声明template版本，编译器依旧会生成默认的函数。

## 条款46：需要类型转换时请为模板定义非成员函数

- 当我们编写一个 class template, 而它与此 template 相关的函数支持“所有参数的隐式类型转换”时，请将那些函数定义为 “class template 内部friend 函数”。

## 条款47：请使用traits classes表现类型信息

- STL的5种迭代器类型

  1. Input迭代器：只能向前移动，一次一步，客户只可读取（不能修改）它们所指的东西，而且只能读

     取一次，它们模仿指向输入文件的阅读指针 (read pointer) ，例如：istream_iterators。

  2. Output迭代器：它们只向前移动，一次一步，客户只可修改它们所指的东西，而且只能修改一次。

     它们模仿指向输出文件的涂写指针 (write pointer) ，例如：ostrean_iterators

  3. Forward迭代器：只能向前移动，一次一步，客户可以多次读或写它们所指的东西， 例如：单向链表的迭代器。

  4. Bidirectional迭代器：可以向前或向后移动，一次一步，客户可以多次读或写它们所指的东西，例如：双向链表的迭代器（STL中的list）。

  5. Random access迭代器：可以在常量时间内向前或向后跳跃任意距离，参考内置指针，例如：vector，deque和string的迭代器。

- Traits classes 使得“类型相关信息”在编译期可用。它们以 templates 和templates特化完成实现。
- 运用重载技术 (overloading) 后， traits classes 有可能在编译期对类型执行if... else 测试。

## 条款48：认识template元编程

- Template metaprogramming (TMP, 模板元编程）可将工作由运行期移往编译期，因而得以实现早期错误侦测和更高的执行效率。

- 用途：
  1. 确保量度单位正确。
  2. 优化矩阵运算。
  3. 可以生成客户定制的设计模式的实现品。

## 条款49：了解new-handler的行为

- 当operator new 抛出异常以反映一个未获满足的内存需求之前，它会先调用一个**客户指定**的错误处理函数，即new-handler。客户可以通过调用set_new_handler去指定该函数。

- new-handler函数的设计要求：
  1. 让更多内存可被使用
  2. 设置为另一个new-handler
  3. 卸载new-handler函数
  4. 抛出bad_alloc异常
  5. 不返回

## 条款50：了解new和delete的合理替换时机

- 常见的理由：
  1. 用来检测运用上的错误
  2. 为了强化效能
  3. 为了收集使用上的统计数据

## 条款51：编写new和delete时需固守常规

- operator new 应该内含一个无穷循环，并在其中尝试分配内存，如果它无法满足内存需求，就该调用 new-handler 。
- operator delete 应该在收到 null 指针时不做任何事。

## 条款52：写了tplacement new 也要写placement delete

- 当你写一个 platcement operator new, 请确定也写出了对应的 placement operator delete 。
- 当你声明 placement new和placement delete, 请确定不要无意识地遮掩了它们的正常版本。

## 条款53：不要轻易忽略编译器的警告

## 条款54：让自己熟悉包括TR1在内的标准程序库

## 条款55：让自己熟悉Boost
