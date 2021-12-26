# 优化编译器的能力与局限性
1. 编译器经常利用一些机会简化表达式，降低计算的执行次数
2. 大多数编译器允许用户决定优化级别
   1. GCC通过`-Og`调用GCC的基本优化
   2. `-o1`和`-o2`是更高级别的优化，但是会造成程序规模的增加或者调试难度的增加
   3. 有时高级别优化可能反而会导致程序性能损失
3. 内存别名使用导致的优化阻碍<br>![](./pic/chapter5/01.PNG)
   1. 在这两段代码中，函数twiddle2 的效率更高。
      1. twiddle2：`*xp += 2 * *yp;`这句执行了一次读取\*yp，一次读取\*xp，然后将计算的结果写入\*xp。一共执行3次读写操作
      2. twiddle1：`*xp += *yp;`每次执行都会执行一次读取\*yp，一次读取\*xp，然后将计算的结果写入\*xp。一共两句，所以共执行6次读写操作
   2. 照理说，如果我们把代码写成twiddle1形式，编译器为了效率应该帮我们优化成twiddle2这种形式。但是现实中编译器不会这样做
      1. 考虑xp=yp的情况：
      ```
      //twiddle1:
      *xp += *xp;/2 * *xp/
      *xp += *xp;/2 * (2 * *xp)/
      
      //twiddle2:
      *xp += 2 * *xp; /*xp + 2*xp/ 
      ```
      2. 此时我们发现twiddle2和twiddle产生的结果不一样，所以编译器不会使用优化
      3. 内存别名使用：这种两个指针可能指向同一个内存的情况叫做内存别名使用
      4. 对于执行安全优化的编译器来说，编译器在面对内存别名使用的情况时不会进行优化
   3. 内存指向未知带来的优化阻碍
      1. 有代码：<br>![](./pic/chapter5/02.PNG)
      2. 因为我们起初明没有给出指针p与指针q的初始化代码。如果p和q指向的地址不同，那么t1=3000。但是如果两个指针指向的地址相同，那么*p=x会让这个地址最后对应的值为1000，导致t1=1000
      3. 因此，如果编译器如果无法确定两个指针的地址是否相异，那么他们会假设所有情况都会发生，限制了可能的优化策略
4. 函数调用导致的优化阻碍：
   1. 代码：<br>![](./pic/chapter5/03.PNG)
   2. 这个代码的func1和func2看似相同，但是func2却不能作为func1的优化。因为如果有<br>![](./pic/chapter5/04.PNG)<br>此时func1返回6（0+1+2+3=6），func2返回0（4*0=0）
   3. 大多数编译器会假设最糟的情况，因此保持原有调用不变而不会对其优化
   4. 关于`x++`和`++x`：“x++”是先把值参与运算以后，自身再进行加一的，而“++x”是自身先加一再参与运算。即如果y=x++，那么y=x，然后x再对自己执行+1；y=++x，是先对x+1，然后再赋值给y

# 程序性能的表示
1. 度量标准：每元素的周期数（CPE）
   1. 对于一个程序，如果我们记录该程序的数据规模以及对应的运行所需的时钟周期，并通过最小二乘法来拟合这些点，我们将得到形如y = a + bx 的表达式，其中y 是时钟周期，x 是数据规模。
   2. 当数据规模较大时，运行时间就主要由线性因子b 来决定。这时候，我们将b 作为度量程序性能的标准，称为**每元素的周期数**
2. 处理器活动由时钟控制，时钟控制某个频率的规律信号，用千兆赫兹（GHz）表示
   1. 千兆赫兹即十亿周期/秒
   2. 4GHz处理器表示处理器时钟运行频率为每秒$4 \times 10^9$
   3. 我们以纳秒（$10^{-9}$秒）或皮秒（$10^{-12}$秒）表示处理器的时钟周期。比如4GHz的1次时钟周期为0.25纳秒或者250皮秒


## 如何优化
## 1. 程序示例
1. 定义一个向量<br>:![](./pic/chapter5/05.PNG)<br>![](./pic/chapter5/06.PNG)<br>data_t为基本元素的数据类型：<br>![](./pic/chapter5/07.PNG)
2. 对向量求和或者求积：<br>![](./pic/chapter5/08.PNG)
   1. OP：OP是一个运算符，具体是求积还是求和取决于我们下面的定义：
        ```
        #define IDENT 0
        #define OP +

        #define IDENT 1
        #define OP *
        ```
   2. get_vec_element：获取第i个元素的值，并把结果保存在val中
   3. 运行效率对比：<br>![](./pic/chapter5/08.PNG)<br>我们发现O1的优化方法可以显著提高程序性能

## 2. 消除循环的低效率：减少重复计算
1. 代码移动：
   1. 主要针对要在循环中执行多次但是计算结果不会改变的计算，因此我们把计算移动到循环之外进行计算
   2. 编译器会试着进行代码移动，但是出于安全性考虑，最好还是由程序员进行手动改进
2. 例子：<br>![](./pic/chapter5/10.PNG)<br>![](./pic/chapter5/11.PNG)
   1. 我们在lower2中就把在lower1中反复计算的strlen()移除了循环，大大提升了程序的运算速度
   2. str(len)是依靠循环测验字符串长度，所以lower1的复杂度是$n^2$。但是lower2因为提前计算好的，程序复杂度只有$n$

## 3. 减少过程调用
1. 构成调用会带来开销，并且妨碍程序优化
2. 这个例子中，每次循环都会调用函数get_vec_element<br>![](./pic/chapter5/12.PNG)<br>![](./pic/chapter5/13.PNG)
   1. get_vec_element每次都会获取下一个向量元素，同时检查i是否超出循环边界。尽管这是一个很好的安全意识，但是combine2明显没有越界引用，那么这种检查就是多余的
   2. 我们使用如下的优化，即减少循环中的函数调用：<br>![](./pic/chapter5/14.PNG)<br>我们使用get_vec_start直接获取数组指针，然后直接在循环中访问数组
   3. 实际上，combine3效率还弱于combine2，这是因为其他部分出现了问题，我们将在[消除不必要的内存引用](#4-消除不必要的内存引用)进行解释。但是这种减少调用的习惯是必要的

## 4. 消除不必要的内存引用