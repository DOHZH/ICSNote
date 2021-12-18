# 程序的机器级表示
1. 指令集架构（ISA，又称指令集体系结构）：定义机器级程序的格式和行为
2. 虚拟地址：机器级程序使用的内存地址都是虚拟地址，内存会被模拟成一个巨大的字节数组
3. linux上的指令：
   1. `gcc -Og -o prog main.c mstore.c`<br>
   编译选项og用来告诉编译器生成符合原始C代码的机器代码。如果想要更高性能，可以换用-O1或-O2，但是会导致机器代码变得极为抽象而难以理解<br>-o prog为生成可执行文件prog<br>
   1. `gcc -Og -S mstore.c`<br>生成汇编文件mstore.s<br>![](./pic/chapter3/1.PNG)<br>以.开头的行都是指导汇编器和链接器工作的伪指令，完全可以忽略。删除后，剩余汇编代码与源文件中代码是相关的。<br>![](./pic/chapter3/6.PNG)
      1. pushq：将%rbx值压入栈
      2. popq：将%rbx弹出栈，恢复rbx
      3. 因为是mulstore这个函数自身去做保存，所以我们可以推知这个处理器采用调用者寄存器这个策略（这个部分请看[寄存器中的2](#register))
      4. moveq：将rdx中的内容复制到rbx
      5. 寄存器定义函数mulstore中的x、y和dest分别指向rdi、rsi和rdx。具体见[寄存器中的3](#change)
      6. call 调用函数，结果保存在rax
      7. ret：即函数返回，表示函数结束
      8. 汇编指令最后的q和数据有关，详见[数据格式](#数据格式)。q只是表示大小指示符，大多数情况下可省略
   2. `gcc -Og -c mstore.c`<br>生成目标代码文件mstore.o，是二进制格式
   3. `objdump -d mstore.o`<br>对二进制文件进行反汇编，去将其转换成对应的汇编语言。objdump可以将机器代码等进行反汇编为汇编语言
   4. `objdump -d prog`<br>对prog这个可执行文件进行反汇编
# 数据格式
1. Intel用术语“字（word）”表示16位类型，32位为双字，64位为4字。这些不同的数据类型决定了汇编代码后缀的不同:<br>![](./pic/chapter3/7.PNG)
2.move指令的四个变种：moveb，传送字节；movw，传送字；movel，传送双字；moveq，传送四字
# 寄存器
   1. 寄存器：<br>![](./pic/chapter3/2.PNG)
   2. <a name="register">调用者保存寄存器和被调用者保存寄存器</a>
      1. 问题来源：<br>![](./pic/chapter3/3.PNG)<br>其中，A为调用者（“主调函数”），B为被调用者（“被调函数”）<br>在执行中，我们不难发现，函数B会对%rbx中的内容进行修改，但是实际上函数就是函数，他不应该在A里面对%rbx这个位置的内容做任何变动，所以我们需要对这个位置的数值进行保护
      2. 调用者寄存器：函数A在调用函数B之前，提前保存寄存器rbx的内容，执行完函数B之后，再恢复寄存器rbx 原来存储的内容，这种策略就称之为调用者保存：<br>
      ![](./pic/chapter3/4.PNG)
      3. 被调用者寄存器：函数B在使用寄存器rbx之前，先保存寄存器rbx的值，在函数B返回之前，先恢复寄存器rbx原来存储的内容，这种策略被称之为被调用者保存：<br>
      ![](./pic/chapter3/5.PNG)
      4. 不同的处理器才用的调用者策略采用不同
   3. <a name="change">寄存器的演变</a>：<br>![](./pic/chapter3/8.PNG)
      1. x86-64的CPU使用了16个存储64位值的通用目的寄存器，这些寄存器用来存放整数数据与指针
      2. 前8个处理器由于历史原因，命名方法与后8个不同。后8个采用的是新命名方法
      3. 最里层只能访问最低的1个字节。%ax这一层为16位寄存器，可以访问最低的两个字节。%eax这一层为32位寄存器，可以访问最低的4个字节。而最外层可以访问整个寄存器
      4. 寄存器rsp 用来保存程序栈的结束位置
# 操作指令
1. 指令包含两部分：操作码和操作数。movq一类，属于操作码，他决定了CPU执行操作的类型。操作码后面的部分是操作数。
2. 指令一般是1个操作码，多个操作数。但是ret没有操作数：<br>![](./pic/chapter3/9.PNG)
3. 操作数
   1. 立即数：用来表示常数值，书写方式为‘$’+标准C表示法表示的整数，比如$-577.不同的指令允许的立即数值范围不同。汇编器会自动选择最紧凑的方式进行数值编码
   2. 寄存器：表示某个寄存器的内容，可以引用8、16、32、64位的寄存器都可以作为操作数。表示形式见[寄存器](#寄存器)
   3. 内存引用：
      1. 我们一般将内存视为一个大数组，因此我们表示内存引用，需要目的数据的起始地址addr和数据长度d。记为符号$M_b[addr]$，b一般可以省略
      2. 寄存器带小括号，表示内存引用。比如`(%rax)`
      3. 内存引用的表示：$Imm(r_b,r_i,s)$
         1. Imm：立即数
         2. $r_b$：基址寄存器
         3. $r_i$：变址寄存器
         4. s：比例因子，取值为{1,2,4,8}。具体取什么数值和源代码中定义的数组类型有关，编译器根据数组的类型决定比例因子的数值。比如char的比例因子为1，int为4
         5. 有效地址：有效地址是通过立即数与基址寄存器的值相加，再加上变址寄存器与比例因子的乘积，即$Imm(r_b, r_i, s) → Imm + R[r_b] + R[r_i] · s$
   4. <a name = "form">操作数的格式(寻址操作)汇总</a>：<br>![](./pic/chapter3/10.PNG)<br>比如在%rbx这个操作数而言，就是表中$r_a$可以取的值。而$(r_a)$就是一种内存引用的表示<br>储存器也就是我们所说的内存引用<br>这里是一些操作数的例子：<br>![](./pic/chapter3/11.PNG) 
4. mov指令：
   1. 效果定义：`mov S D`，意思为$D \leftarrow S$
   2. 分为movb、movw、movl、movq四个子命令，传输1、2、4、8字节
   3. 传送指令有两个操作数，第一个是原操作数、第二个是目的操作数
      1. 源操作数：可以是立即数、寄存器或者内存地址
      2. 目的操作数：只能是寄存器，要么是一个内存地址，不可以是一个立即数
      3. x86-64处理器限制，mov指令的源操作数和目的操作数不能都是内存的地址。如果一个数需要从内存一个位置复制到另一个位置，需要用两条mov完成：
      ```
      mov memory, register
      mov register, memory
      ```
      1. mov的后缀一定要和寄存器的大小匹配，比如%eax是32位寄存器的表示，所以要使用movl。`movl $0x4050, %eax`，下图是源操作数与目的操作数的所有组合形式的例子：<br>![](./pic/chapter3/12.PNG)
   4. movabsq：
      1. 当movq的源操作数为立即数，他只能是32位的补码表示。我们需要对该32位数进行符号拓展，拓展为64位然后传输到目标位置。
      2. 如果我们想要操作64位数字，使用movabsq。movabsq的源操作数可以是任意64位数，**但是目的操作数只能是寄存器**
   5. mov这几种指令的具体传输过程
      1. 例子：<br>![](./pic/chapter3/13.PNG)
      2. movabsq：直接将64位数字写入
      3. movb：只改变rax寄存器中8bit部分，-1表示成1字节就是FF
      4. movw：和movb同理，只改变16bit部分
      5. movl：改变最低4字节长度的数值，并且将前面4字节全部变0。这种全变成0的操作只有目的操作数为寄存器时才会执行。**这是x86-64 处理器的一个规定，即任何位寄存器生成32位值的指令都会把该寄存器的高位部分置为0**
      6. movq：其实只改变了最后4位部分为F，剩下的F都是符号拓展，因为我们输入的是-1，他的符号拓展在二进制中全是1，所以16进制下都是F
   6. 源操作数小于目的操作数时的mov指令
      1. 零拓展指令（movz）：即目的操作数的空位由0补齐<br>![](./pic/chapter3/14.PNG)
      2. 符号拓展指令(movs)：目的操作数空位由符号扩展填充![](./pic/chapter3/15.PNG)<br>cltq没有操作数，他总是以寄存器%eax为源，%rax为目的数，效果等价于`movslq %eax, %rax`
      3. 为什么零拓展没有类似movslq这种4字节拓展到8字节的操作：因为使用movl即可实现。4字节值以寄存器为目的的指令，会将高4字节值全部设置为0(这是x86-64的规定，详见movl部分的解释)。因此没有必要为4字节转8字节单独设立指令
   7. 数据传输的示例：
      1. 代码:<br>
      ```
      int main(){
         long a = 4;
         long b = exchange (&a, 3);
         printf(”a = %1d , b = %1d\n”, a, b);
         return 0;
         }
      
      long exchange(long *xp , long y){
         long x = *xp;
         *xp = y;
         return x;
      }
      ```
      1. exchange的机器指令拆解：其中，xp作为第一个参数，在%rdi中，y作为第二个参数，在%rsi中
      ```
      exchange:
         movq (%rdi), %rax
         movq %rsi, (%rdi)
         ret
      ```
         1. 第一个movq：从内存读取数据到寄存器<br>为memory到register的命令，因为x为返回数，所以x的寄存器位置为%rax。第一步将xp的内存地址复制到x的寄存器中，即读取xp的数据
         2. 第二个movq：将寄存器数据写至内存<br>为register到memory
         3. C 语言中所谓的指针其实就是地址。
         4. 局部变量一般保存在寄存器中，而不是内存中。比如例子中的xp和x。C语言这样优化可以加快访问速度
         5. 这个例子中，因为根据寄存器规则，两个函数的参数会被写入寄存器以实现快速访问。但是xp是指针变量，它本身是地址，是一种间接寻址。因此如果想要获取数据，根据寄存器规则要回到内存找到数据。但是y是一种绝对寻址，因此不用加括号。具体可以见"操作指令3.4"的[表格](#form)
5. 栈操作下的数据传输指令
   1. 栈的增长方向是从高地址向低地址，栈顶的元素是所有栈中元素地址中最低的。根据惯例，栈是倒过来画的，栈顶在图的底部，栈底在顶部。![](./pic/chapter3/16.PNG)
   2. 栈的寄存器位置为rsp
   3. 压栈：push
      1. 如果压%rax中的数据，操作为:`pushq %rax`
      2. 这个指令可以拆解为两步：
      ```
      subq $8, %rsp
      movq %rax, (%rsp)
      ```
      因为%rax位置为8字节数，所以，第一步先将栈指针%rsp减8，让其指向更低的位置（也就是指针像栈顶部移动），然后将rax的值复制到rsp指向的新位置<br>
      ![](./pic/chapter3/17.PNG)
   4. 弹出栈：pop
      1. `popq %rax`
      2. 这个操作等价于：
      ```
      movq (%rsp), %rax
      addq $8, %rsp
      ```
      1. 实际上pop 指令是通过修改栈顶指针所指向的内存地址来实现数据删除的，此时，内存地址0x100 内所保存的数据0x123仍然存在，直到下次push 操作，此处保存的数值才会被覆盖<br>![](./pic/chapter3/18.PNG)

# 算数和逻辑操作
1. 加载有效地址：leaq
   1. 效果定义：`leaq S,D`，$D \leftarrow &S$
   2. leaq是movq的变形，主要作用是加载有效地址，效果是读内存数据到寄存器。因此，目标操作数必须是寄存器，源操作数必须是内存引用(memory)。
   3. 和mov的区别：
      1. lea 8(%ebx), eax就是将ebx+8这个值直接赋给eax，而不是把ebx+8处的内存地址里的数据赋给eax
      2. mov 8(%ebx), eax则是把内存地址为ebx+8处的数据赋给eax
   4. 在x86-64上，地址均为64位，因此没有leab、leaw这类命令
   5. 表示加法与有限的乘法运算：
      1. `leaq 7(%rdx,%rdx,4),%rax`
      假设寄存器rdx 内保存的数值为x，那么有效地址的值为7 + %rdx + %rdx * 4 = 7+ 5x。
      2. 有函数：<br>
      ```
      long scale(long x, long y, long z){
         long t = x + 4 * y + 12 * z;
         return t;
         }
      ```
      它的编译形式为：
      ```
      scale:
         leaq (%rdi, %rsi, 4), %rax
         leaq (%rdx, %rdx, 2), %rdx
         leaq (%rax, %rdx, 4), %rax
         ret
      ```
      为什么12\*z要拆成4*(3*z)?因为比例因子只能取1,2,4,8。这里如果写成12*z，那么就会变成(%rax, %rdx, 12)，比例因子是12，不符合要求
2. 一元操作
   1. 只有一个操作数，既是源，也是目的：<br>![](./pic/chapter3/19.PNG)
   2. 操作数可以是寄存器或者内存地址(memory)
3. 二元操作
   1. 第一个操作数为源操作数，第二个操作数既是源操作数，又是目的操作数
   2. 源操作数可以是立即数、寄存器或者内存地址；第二个操作数可以是寄存器或者内存地址：<br>![](./pic/chapter3/20.PNG)
   3. 如果第二个操作数为内存地址，处理器需要从内存读出值，再执行操作
   4. 一元操作和二元操作的例子：<br>该图为一开始的数据以及后续操作：<br>![](./pic/chapter3/21.PNG)
      1. addq：相加命令，改变的是%rax的内存位置，因此rax本身不变，但是0x100处的数据由0xFF变为0x100
      2. subq：相减命令，改变的是%rax地址+8位处的地址，也就是0x108处的数据，操作之后，0xAB变为0xA8
      3. incq：自身+1命令，改变的仍是内存位置的数据，0x13变成0x14
      4. subq：相减命令，但是改变的是rax在寄存器中的数据而不是地址位置，0x100变成0xFD<br>![](./pic/chapter3/22.PNG)
4. 移位
   1. 第一个操作数为移位量，第二项给出的是要移位的数：<br>![](./pic/chapter3/23.PNG)
      1. 左移指令两个效果相同，都是左移后在右侧补0
      2. 算数右移：左侧填符号位
      3. 逻辑右移：左侧补0
   2. 移位量k：可以是立即数或者存放在寄存器cl中的数
      1. cl长度为8bit，因此理论上位移量最大为$2^8-1=255$
      2. 但是实际上，我们的位移量只有m。m由这个式子求得$2^m=w$，w为当前操作数的位数。比如我们当前操作数是8位（salb指令），那么m=3，即我们的移位量由cl的低3位决定
   3. 移位的用途：
      1. 右移操作要求区分有符号和无符号数，所以补码运算成为实现有符号数运算的一种好选择
      2. 乘法指令在编译器往往会有更长的执行时间，因此我们会去找一个更高效的方法替代乘法指令。移位就是一个良好的指令
      3. 代码：<br>
      ```
      long arith(long x, long y, long z){
         long t1 = x ^ y;
         long t2 = z * 48;
         long t3 = t1 & 0xF0F0F0F;long t4 = t2 - t3;
         return t4;
         }
      ```
      对应的汇编代码为：<br>
      ```
      xorq %rsi, %rdi
      leaq (%rdx, %rdx, 2), %rax
      salq $4, %rax
      andl $252645135, %edi
      subq %rdi, %rax
      ret
      ```
      其中，我们发现乘法运算我们使用了leaq这个指令，在rax中构造了一个3*z<br>
      salq，将寄存器rax 进行左移4 位，左移4 位的操作是等效于乘以2 的四次方，也就是乘以16。

# 控制
程序语言中存在分支、循环这种非线性语句，我们只用前面的语句难以操作
## 1. 条件码
1. ALU 除了执行算术和逻辑运算指令外，还会根据该运算的结果去设置条件码寄存器
2. CPU负责维护条件码寄存器。条件码寄存器对于执行条件进行检测
3. 条件码寄存器在执行下一条语句时，上一个行语句的条件状态会被覆盖
4. 常见条件码：
   1. CF：进位标志，当CPU 最近执行的一条指令最高位产生了进位时，进位标志(CF）会被置为1，它可以用来检查无符号数操作的溢出。
   2. ZF：零标志，当最近操作的结果等于零时，零标志(ZF) 会被置1。
   3. SF: 符号标志，当最近的操作结果小于零时，符号标志(SF) 会被置1
   4. OF: 溢出标志，针对有符号数，最近的操作导致正溢出或者负溢出时溢出标志(OF)会被置1
5. 我们在前面提到的[算数与逻辑操作](#算数和逻辑操作)中除了leaq，其他指令都会设置条件码。比如XOR，执行时会将CF火刃OF均设置为0
6. CMP和test也会设置条件码寄存器：
   1. cmp 指令是根据两个操作数的差来设置条件码寄存器。cmp 指令和减法指令(sub）类似，也是根据两个操作是的差来设置条件码，二者不同的是cmp 指令只是设置条件码寄存器，并不会更新目的寄存器的值
   2. test 指令和and 指令类似，同样test 指令只是设置条件码寄存器，而不改变目的寄存器的值
      1. test可以用来检测一个数是否是负数：test两个操作数相同
      2. 用来测试一个操作数的某几个位置的bit：两个参数一个操作数，一个是掩码
   3. 指令细则：<br>![](./pic/chapter3/25.PNG)
## 2. 访问条件码
1. 条件码一般不会直接读取，而是通过三种方式实现
   1. 根据条件码的组合，将一个字节设置为0或1
   2. 用条件跳转到程序某个部分
   3. 有条件地传送数据
2. SET：根据条件码的组合，将一个字节设置为0或1
   1. set指令的目的操作数必须是低位单字节寄存器元素或者字节的内存位置，指令会将指定字节位置设置为0或1.
   2. 为了得到32位或者64位结果，我们必须针对高位清0
   3. 常用set指令：<br>![](./pic/chapter3/26.PNG)
   4. 示例：
      1. setb例子：
         ```
         int comp(long a, long b){
            return (a == b);
            }
         ```
         汇编指令为：
         ```
         comp:
            cmpq %rsi, %rdi
            sete %al
            movzl %al, %eax
            ret
         ```
         这里返回值int为4字节长度类型，所以寄存器开辟的是32位寄存器%eax<br>
         a in rdi, b in rsi。cmpq是对比a是否等于b。cmp命令中，被比较数a会在第二个位置<br>如果a和b相等，ZF设置为1<br>
         sete：如果ZF为1，eax最低位al设置为1，否则为0<br>
         movzl：对eax其他位进行清零<br>
      2. setl的例子：<br>
         ```
         int comp(char a, char b){
            return (a < b);
            }
         ```
         汇编指令为：
         ```
         comp:
            cmpb %sil, %dil
            setl %al
            movzbl %al, %eax
            ret
         ```
         这里a和b类型是long，为4字节长度类型，所以寄存器开辟的是32位寄存器%eax<br>
         a in dil, b in sil。cmpq是对比a是否等于b。cmp命令中，被比较数a会在第二个位置<br>如果a小于b，ZF设置1<br>
         setl：根据SF^OF的值设定al：<br>![](./pic/chapter3/27.PNG)<br>其中，case3和4均发生了溢出，所以OF=1<br>
         movzl：对eax其他位进行清零<br>
## 3. 跳转
1. 计算机中，跳转的目的地通常用label指明，比如下面的汇编代码：<br>![](./pic/chapter3/28.PNG)<br>指令会因为jmp而跳转到L1的位置
2. 产生代码文件时，汇编器会确定所有带标号指令的地址，并将跳转目标编码为跳转指令的一部分
3. 跳转指令的编码
   1. PC相对寻址：以程序计数器PC的当前值（R15中的值）为基地址，指令中的地址标号作为偏移量，将两者相加后得到操作数的有效地址。比如L2在第8行，那么这种寻址方法就是PC当前地址+0x8
   2. 绝对寻址：直接存储地址
4. 常见指令汇总：<br>![](./pic/chapter3/29.PNG)
5. jmp:无条件跳转指令
   1. 直接跳转：操作数为一个label
   2. 间接跳转：操作数为一个地址。间接地址写法为`*操作数`
      1. 跳到寄存器：jmp *%rax
      2. 跳到内存：jmp *(%rax)
## 4. 条件分支
1. C语言中有一个实现jmp的命令，goto。但是goto并不是一个好的编程风格，因此我们一般不去使用。但是我们可以借助goto理解汇编
2. 使用控制转移实现条件分支：
   1. if-else跳转的常见goto改写：<br>
   ```
   if (test-expr)
      then-statement
   else
      else-statement
   
   改写为goto：
      t = test-expr;
      if (!t)
         goto false;
      then-statement
      goto done;
   false:
      else-statemnt
   done
   ```
   2. 代码示例：
   ```
   long absdiff_se(long x, long y){
      long result;
      if(x < y){result = y - x;} else{result = x - y;}
      return result;
      }
   ```
   我们理解这个例子时候可以转成goto的形式
   ```
   long absdiff_se(long x, long y){
      long result;
      if(x >= y)
         goto x_ge_y;
      result = y - x;
      return result;
   x_ge_y:
      result = x- y;
      return result;
   }
   ```
   这种形式与汇编代码基本一致：
   x in %rdi, y in %rsi
   ```
   absdiff_se:
      cmpq %rsi, %rdi
      jge .L2
      movq %rsi, %rax
      subq %rdi, %rax
      ret
   .L2:
      movq %rdi, %rax
      subq %rsi, %rdi
      ret
   ```
3. 上述的例子的这种条件跳转在现代处理器中可能不够高效。**使用数据的条件转移会比控制转移更加高效**
   1. 为什么这种更高效：
      1. 处理器通过流水线获得高性能。
      2. 处理器常常会预先确定接下来要执行的语句，以便于并行处理
      3. 因此，当遇到条件跳转时，处理器会根据分支预测器来猜测每条跳转指令是否执行
      4. 当发生错误预测时，处理器会丢弃它为当前分支做的所有工作并回跳至起始位置。这会浪费大量的时间，导致程序性能严重下降
      5. 减少分支有利于提高程序效率
   2. 条件语句转化为赋值表达：<br>
   ```
   v = test-expr ? then-expr : else-expr;

   改写成数据传递表达：
   v = then-expr;
   ve = else-expr;
   t = test-expr;
   if (!t) v = ve;
   ```
   3. 原代码：
      ```
      long comvdiff_se(long x, long y){
         long result;
         if (x < y)
            result = y - x;
         else:
            result = x - y; 
         return result;
      }
      ```
   4. 用条件赋值进行改写：
      ```
      long comvdiff_se(long x, long y){
         long rval = y - x; 
         long eval = x - y;
         long ntest = x >= y;
         if(ntest){rval = eval;} 
         return rval;
      }
      ```
   5. 汇编代码：
      x in %rdi, y in %rsi
      ```
      comvdiff_se:
         movq %rsi, %rax
         subq %rdi, %rax
         movq %rdi, %rdx
         subq %rsi, %rdx
         cmpq %rsi, %rdi
         cmovge %rdx, %rax
         ret
      ```
   6. cmovge：是根据条件码的某种组合来进行有条件的传送数据，当满足规定的条件时，将寄存器rdx内的数据复制到寄存器rax内
   7. 常见的条件传送指令：![](./pic/chapter3/30.PNG)
4. 并非所有指令都可以修改成数据传送形式
   1. 反例：<br>
      ```
      long cread(long *xp){
         return (xp ? *xp : 0);
      }
      ```
      改写成条件传送
      ```
      v1 = *xp;
      v2 = 0;
      t = xp;
      if (!t){v1 = v2} return v1;
      ```
      如果是一个空指针xp，这明显会报错，v1根本没法赋值，也谈不上改写成汇编语言了
   2. 条件传送也可能会导致多余的计算。在面对分支条件是有较复杂计算的时候，条件传送并不划算，因为它要把分支全部计算

## 5. 循环
1.  do-while循环
    1. 改写方法：<br>
      ```
      do
         body-statement
         while (test-expr)
      ```
      改写为
      ```
      loop:
         body-statement
         t = test-expr;
         if (t)
            goto loop;
      ```
     2.  改写示例：
     ```
     long fact_do(long n)
     {
        long result = 1;
        do {
           result *= n;
           n = n -1;
        }while (n>1);
        return result;
     }
     ```
     汇编指令：
     n in %rdi
     ```
     mov1 $1, %eax
     .L2:
         imulq    %rdi, %rax
         subq     $1, %rdi
         compq    $1, %rdi
         jg       .L2
         rep
         ret
     ```
     rep：循环执行标识。主要是提醒程序要执行分支预测了。不写rep也是可以的，但是如果写了会极大优化编译
2.  while循环
    1. 改写方法：<br>
      ```
      while (test-expr)
         body-statement
      ```
      改写为
      ```
         goto test;
      loop:
         body-statement
      test:   
         t = test-expr;
         if (t)
            goto loop;
      ```
   2. guarded-do方法：先改成do-while再改成汇编
      ```
      t = test-expr
      if (!t)
         goto done
      do
         body-statement
         while (test-expr)
      done:
      ```
      汇编形式：
      ```
      t = test-expr
      if (!t)
         goto done
      loop:
         body-statement
         t = test-expr
         if (t)
            goto loop;
      done:
      ```
    3. 改写示例：
     ```
     long fact_do(long n)
     {
        long result = 1;
        while (n>1) {
           result *= n;
           n = n -1;
        }
        return result;
     }
     ```
     汇编指令：
     n in %rdi
     ```
     mov1 $1, %eax
     fact_while:
         movl     $1, %eax
         jmp      .L5
     .L6:
         imulq    %rdi, %rax
         subq     $1, %rdi 
     .L5:
         compq    $1, %rdi
         jg       .L6
         rep
         ret
     ```
     这里L6执行完，会顺序执行L5中的内容，以实现循环。<br>
     我们当然还可以用guarded-do方法
3.  for
    1.  for和while本质等价，产生的策略和while的两种翻译相同
    2.  原语句：<br>
    ```
    for (init-expr; test-expr; update-expr)
         body-statement
    ```
    3.  普通策略
    ```
      init-expr;
      goto test;
    loop:
      body-statement;
      update-expr;
    test:
      t = test-expr;
      if (t)
         goto loop
    ```
    4.  guarded-do策略
    ```
      init-expr;
      t = test-expr;
      if (!t)
         goto done;
    loop:
      body-statement;
      update-expr;
      t = test-expr;
      if (t)
         goto loop;
    done:
    ```
    test-expr是循环继续条件，因此在loop中要再次给t赋值，因为test-expr需要重新检查在test-expr之后的变量是否仍符合循环条件
## 6. switch
1. 提高了C代码的可读性，也通过使用跳转表这种数据结构使得程序更加高效
2. 跳转表：一个数组，表项i是一个代码段的地址。当开关索引值等于i，会跳转到在这个对应地址位置的代码块
3. 执行开关语句的时间与开关情况的数量无关。
4. GCC执行switch的时候，会根据开关情况的数量与细数成都来翻译开关语句。如果开关情况较多（比如4个以上），会使用跳转表
5. 汇编示例：<br>
![](./pic/chapter3/31.PNG)
<br>右侧为switch语句在C语言中的类汇编形式<br>汇编指令为<br>![](./pic/chapter3/32.PNG)
   1. cmpq：该段程序中，因为case一共有100，102-104，和106几种情况，因此编译器先将n-100，取值范围为0~6（在C语言中的类汇编形式中为index变量）。
   2. .L4：此时会开辟一个长度为7跳转表，表每个单元都是8字节地址，记录不同代码块位置。跳转表的地址记录形式为间接寻址，以.L4为地址起始，然后对于表内数值地址他们相对于L4的偏移量。<br>因为101和105没有情况，所以他们的地址都写为.L8，即默认地址。因为104和106的情况下，代码块相同，所以他们的地址也相同，都是.L7<br>![](./pic/chapter3/33.PNG)
   3. jmp \*.L4：\*表示这是一个间接跳转，操作数为一个内存位置，索引由%寄存器rsi给出，这里存放了index。8表示这是一个8字节地址
   4. case102，也就是loc_B位置，并没有写break，因此代码不会终止跳转至.L2，而是继续顺序执行，执行case103，然后在case103中跳出

# 过程


