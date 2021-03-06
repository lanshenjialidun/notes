# 条款 06：当 auto 无法推导想要的类型时，则使用显式类型初始化的语法.md

条款05 解释了比起显式指定类型，使用 auto 来声明变量提供了大量技术上的优点，但是有时候 auto 的类型推导出 zigs（这个类型），但是你想要的是 zag（另外一个类型）。举个例子，假设我有一个函数以 Widget 为参数并且返回一个 std::vector\<bool\>，每个 bool 指示 Widget 是否提供了特定的特性：

```C++
std::vector<bool> features(const Widget& w);
```

进一步假设第 5 位指示了 Widget 有高优先级。因此我们写了如下的代码：

```C++
Widget w;

bool highPriority = features(w)[5];
processWidget(w, highPriority);               //根据w的优先级处理w
```

这段代码没有错。它工作得很好。但是如果我们做一些看起来无害的操作，也就是用 auto 来替换 highPriority 的显式类型声明，

```C++
auto highPriority = features(w)[5];
```

情况改变了。所有的代码将能继续通过编译，但是它的行为不再和预测的一样了：

```C++
processWidget(w, highPriority);               //未定义的行为！
```

就像注释指示的那样，调用 processWidget 现在是未定义的行为。但是为什么会这样？回答有可能很奇怪。在使用 auto 的代码中，highPriority 的类型不再是 bool，尽管 std::vector\<bool\> 概念上保存了bools，std::vector\<bool\> 的 operator[] 不返回容器元素的引用（除了 bool 类型，其他的 std::vector::operator[] 都这么返回）。作为替换，它返回一个类型是 std::vector\<bool\>::reference 的对象（一个类内部装了std::vector\<bool\>）。

std::vector\<bool\>::reference 之所以会存在是因为 std::vector\<bool\> 被指定来代表它包装的 bools，每个 bool 占一位。这将造成一个问题，对于 std::vector\<bool\> 的 operator[]，因为对于 std::vector\<T\>，operator[] 被假定要返回 T&，但是 C++ 禁止返回位的引用。因为没有可能返回一个 bool&，operator[] 为 std::vector\<bool\> 返回一个对象，这个对象表现得像 bool& 一样。为了这个行为能成功，在本质上，std::vector\<bool\>::reference 对象必须能被用在任何 bool& 能用的地方。在 C++ 的众多特性中，能让 std::vector\<bool\>::reference 这么工作的是一个到 bool 的隐式转换。（不是 bool& 到 bool，对于 std::vector\<bool\>::reference 如何效仿 bool&，如果要解释清楚所有的技术那就走远了，所以我只是简单地谈论了众多技术中的一个，隐式转换。）

记住了这些信息后，再看看源代码的这一部分：

```C++
bool highPriority = features(w)[5];           //显式声明highPriority的类型
```

这里，features 返回一个 std::vector\<bool\> 对象，它的 operator[] 被调用。operator[] 返回一个 std::vector\<bool\>::reference 对象，根据 highPriority 初始化的需要，这个对象将会隐式地转换为 bool。因此，highPriority 最后代表 features 返回的 std::vector\<bool\> 中的第 5 位的值（就像它支持的那样）。

对比对 highPriority 使用 auto 初始化声明时发生的情况：

```C++
auto highPriority = features(w)[5];           //推导highPriority的类型
```

再一次，features 返回一个 std::vector\<bool\> 对象，并且，再一次，operator[] 被调用。operator[] 继续返回一个 std::vector\<bool\>::reference 对象，但是现在这里有些变化，因为 auto 推导 highPriority 的类型为 std::vector\<bool\>::reference。highPriority 不再拥有 features 返回的 std::vector\<bool\> 中的第 5 位的值。

它拥有的值依赖于 std::vector\<bool\>::reference 是怎么实现的。一种实现是，这些对象包含一个指针指向一个机器字节（word）的引用，以及一个代表特定位的偏移。考虑一下，这样一个 std::vector\<bool\>::reference 的实现，对于 highPriority 的初始化意味着什么。

对于 features 的调用返回一个 std::vector\<bool\> 临时对象。这个对象没有名字，但是为了讨论的目的，我讲把它叫做 temp。operator[] 是对 temp 的调用，并且 std::vector\<bool\>::reference 返回一个对象，这个对象指向一个字节（word），这个被 temp 管理的数据结构中的字节持有所需的位，加上对象存储的偏移就能找到第 5 位。highPriority 是这个 std::vector\<bool\>::reference 对象的一份拷贝。所以 highPriority 也持有指向 temp 中某个字节（word）的指针，以及第 5 位的偏移。在语句的最后，temp 销毁了，因为它是一个临时对象。因此，highPriority 持有的是一个悬挂的指针，并且这就是在 processWidget 调用中造成未定义行为的原因：

```C++
processWidget(w, highPriority);               //未定义行为！highPriority持有悬挂的指针
```

std::vector\<bool\>::reference 是一个代理类的例子：一个类的存在是为了效仿和增加其他类型的行为。代理类被用作各种各样的目的。std::vector\<bool\>::reference 为了提供一个错觉：std::vector\<bool\> 的 operator[] 返回一个 bool 引用。再举个例子，标准库的智能指针是一个对原始指针进行资源管理的代理类。代理类的用法已经慢慢固定下来了。事实上，设计模式“Proxy”就是软件设计模式万神殿（Pantheon）中长期存在的一员。

对于客户来说，一些代理类被设计成可见的。举个例子，这就是 std::shared\_ptr 和 std::unique\_ptr 的情况。另外一种代理类被设计成或多或少不可见的。std::vector\<bool\>::reference 就是这种“不可见”代理类的例子，对于它的同胞 std::bitset 也有这样的代理类：std::bitset::reference。

同样在这个阵营中，一些 C++ 库中的类使用一种叫 expression template 的熟知科技。这样的库最初开发出来是为了提升算术运算代码的效率。给出一个类 Matrix 和 Matrix 对象 m1，m2，m3 和 m4，举个例子，表达式

```C++
Matrix sum = m1 + m2 + m3　+ m4;
```

能被计算得更加有效率一些，只需要让 Matrix 对象的 operator+ 返回一个结果的代理来代替结果本身。也就是，两个 Matrix 对象的 operator+ 将返回一个对象，这个对象用像 Sum\<Matrix, Matrix\> 一样的形式来代替一个 Matrix 对象。和 std::vector\<bool\>::reference 与 bool 同样的情况，这里有一个从代理类到 Matrix 的隐式转换，它允许用 “=” 右侧的表达式产生的代理类初始化 sum（这个对象的类型传统地编码整个初始化表达式，也就是，一个和 Sum<Sum<Sum<Matrix, Matrix>, Matrix> 类似的东西。对于这个类型而言，客户肯定不需要知道。）

一般情况下，“看不见的”代理类不能和 auto 很好地结合在一起。这样的对象的生命周期常常被设计成不能超过一条简单语句，所以创造这些类型的变量将违反基础库设计时假设的情况。这就是 std::vector\<bool\>::reference 的情况，我们看到了，违反假设将导致未定义行为。

你因此想要避免这样形式的代码：

```C++
auto someVar = expression of "invisible" proxy class type   //“不可见的”代理类的表达式
```

但是你要怎么意识到你在使用代理对象呢？制造代码的人不太可能主动解释它（代理类）的存在。至少在概念上，它们被假设为“不可见的”。并且一旦你找到了它们，你就真的能抛弃 auto 以及在 条款05 中说明的 auto 的众多优点吗？

让我先解决“如何去找到它们”的问题。尽管“不可见的”代理类被设计成，在日常使用中，程序员不会发现它们，但库生产者在使用它们时常常会用文档标示出他这么做了。你越多地让你自己熟悉你所使用的库的基础设计决策，你就越少地被库中的代理类给偷袭。

当文档出现遗漏的时候，头文件会填补这个缺陷。想要完全隐藏掉代理类对源代码来说是几乎不可能的。典型地，它们就是用户想要调用的函数的返回，所以函数签名常常反映了它们的存在。这里有 std::vector\<bool\>::reference 的原型，看这例子：

```C++
namespace std{

template <class Allocator>
class vector<bool, Allocator>{
public:
    class reference {...};

    reference operator[](size_type n);
};

}
```

假设你知道一个 std::vector\<T\> 的 operator[] 常常返回一个 T&，在这种情况下 operator[] 不寻常的返回就是一个使用代理类的提示。把注意力放在你使用的函数的接口上常常帮你揭露出代理类的存在。

作为实践，很多开发者只有在当他们尝试追踪神秘的编译错误，或者调试出不正确的单元测试结果时，才能发现代理类的存在。不管你怎么发现他们，一旦 auto 推导出一个代理类类型替换了被代理的类型，解决方法不是禁止 auto 的使用。auto 它本身没有问题。问题是 auto 不能推导出你想要它推导的类型。解决方法是强制一个不同的类型推导。你可以使用被我称为“显式类型初始化语法”的方法。

显示类型初始化语法涉及用 auto 声明一个变量，但是需要把初始化表达式的类型转换到你想要 auto 推导的类型。举个例子，这里给出怎么用这个方法来强制 highPriority 转换到 bool：

```C++
auto highPriority = static_cast<bool>(features(w)[5]);
```

这里，features(w)[5] 继续返回一个 std::vector\<bool\>::reference 对象，就像它以前一样，但是 cast 把表达式的类型转换成了 bool，然后 auto 推导出来的类型也是 bool。在运行期，std::vector\<bool\>::reference 对象从 std::vector\<bool\>::operator[] 返回，然后执行它支持的到 bool 的转换，并且作为转换的一部分，现在从 features 返回并且仍然有效的指向 std::vector\<bool\> 的指针被间接引用了。这样就避免了我们之前做过的未定义行为。然后下标 5 被应用在指针指向的字节上，找到对应的位，然后找出的 bool 值被用来初始化 highPriority。

对于 Matrix 的例子，显式类型初始化语法看起来将会是这样：

```C++
auto sum = static_cast<Matrix>(m1 + m2 + m3 + m4);
```

这个语法的应用不止限制在初始化时产生的代理类类型上。它在当你故意创建一个变量，让它的类型不同于初始化表达式产生的类型时同样可以起到一个强调的作用。举个例子，假设你有一个函数计算一些公差值：

```C++
double calcEpsilon();                   //返回公差值
```

calcEpsilon 明确返回一个 double，但是假设你知道，对于你的应用，float 的精度就足够了，并且你更关心 float 和 double 的大小。你可以声明一个 float 类型的变量来存放 calcEpsilon 的返回值，

```C++
float ep = calcEpsilon();               //double -> float的隐式转换
```

但是这几乎没有宣布“我故意降低了函数返回值的精度”。这时就可以用显式类型转换语法来声明它，行动起来：

```C++
auto ep = static_cast<float>(calcEpsilon());
```

同样的理由可以应用在你想故意用整形来存放浮点型的表达式。假设你需要用随机存取迭代器（比如，std::vector,std::deque 或 std::array 的迭代器）计算容器元素的下标时，并且你给出了一个从 0.0 到 1.0 的 double 值来标示从容器的开始到你需要的元素的位置有多远。（0.5将标示着容器的中间位置）进一步假设你确信下标计算的结果适合作为一个 int。如果容器是 c，并且 double 值是 d，你可以这样计算下标：

```C++
int index = d * c.size();
if (idx == c.size()) --idx;            //当d为1.0时，idx保持有效
```

但是这模糊了一个事实，就是你故意地把右边的 double 转换成了一个 int。显式类型初始化语法让事情变得易懂了：

```C++
auto index = static_cast<int>(d * c.size());
if (idx == c.size()) --idx;            //当d为1.0时，idx保持有效
```

## 请记住

* 对于一个初始化表达式，“看不见的”代理类型能造成 auto 推导出“错误的”类型
* 显式类型初始化语法强制 auto 推导出你想要它推导的类型。
