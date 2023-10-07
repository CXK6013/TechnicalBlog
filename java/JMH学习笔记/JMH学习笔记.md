# JMH探索学习笔记(一)

> 这篇想换成对话体，像柏拉图的《理想国》一样，尝试切换文章的风格。

## 缘起

本篇对话的双方是悟空与龟仙人，这两个也学了Java，龟仙人决定传授给悟空JMH。于是龟仙人找到悟空。

龟仙人: 悟空，你可曾听过语言之间的性能对比。

悟空： 我听过，一般C++最快，然后是Java之类的。

龟仙人: 所以为什么C++最快呢。

悟空:  这个我学过诶，武天老师，因为计算机最终执行的都是机器码，C++经过编译执行的时候，都是相关平台的机器码，比如我在Windows下面就是exe，但是C++可移植性不好。

龟仙人: 你说的移植性不好是什么意思，你指的是我写了一个简单的小程序，就是hello world，在Windows上写了一个HelloWorld像下面这样:

```c++
#include <iostream>

using namespace std;

int main()
{
    cout << "Hello world!" << endl;
    return 0;
}
```

这个到mac下面就编译不了？

悟空: 当然不是，武天老师，这个文本在.cpp文件照样可以编译，高级语言的源代码都具备可移植性，但是最终执行的是对应平台的可执行文件，在Windows后缀为.exe。但是Linux并不认识这个文件，所以C++在部署的时候要针对各个平台输出可执行文件。有时候你用了对应平台的库函数，比如我想做网络编程，我就需要调操作系统提供的能力，但是Windows上有一套，Linux上有一套，也就是说C++在使用这些能力的时候，要针对不同的平台做适配。想到这里，武天老师，我想说，那做C++服务端是不是有些头疼，每个平台都要写一套，这我还能下班呢。

龟仙人哈哈大笑道: 1974年，贝尔实验室正式对外发布Unix，贝尔实验室还对外提供了源代码，于是就出现了好些独立开发与Unix基本兼容但是又不完全兼容的OS，20世纪80年代初期，Unix越来越不兼容，因为很多Unix厂商通过加入新的特性来使他们的程序与众不同，是啊，与众不同啊，那这位编写软件带来了巨大的麻烦，每个都要写一套，真是让人头痛啊，为了提高兼容性和应用程序的可移植性，IEEE(电气和电子工程师协会)开始标准化Unix的开发，这也就是POSIX，这套标准涵盖了很多方面比如网络编程接口，线程、网络编程。我们熟悉的Linux就实现了POSIX兼容，苹果的操作系统就是基于Unix的。好，悟空，你接着说，C++为什么快，Java为什么慢。

悟空: Java最终输出的是中立平台的jar，jar执行的时候需要JDK环境，因为编译最终输出的是独立于平台的产物，所以有非常良好的可移植性。因为不能编译的不是对应平台的机器码，所以Java就慢一些。

龟仙人: 还有什么要补充的呢？

悟空想了想，接着说道: 我记得Java是翻译执行的，也就是说编译器是一边读字节码，一边将字节码翻译成对应的机器码。这样肯定比直接执行机器码要慢上许多。

龟仙人:  悟空，下面是一份用莱布尼兹公式计算π值的性能报告(例子来自于Glavo):

![](https://a.a2k6.com/gerald/i/2023/09/25/3vr4s.png)

测试平台为: Ubuntu 22.04, CPU: Ryzen 7 5800X , 环境版本为:

- GCC: 11.4.0
- Java: 20.0.2
- GraalVM: Oracle GraalVM for JDK 20.0.2
- Go: 1.21.0
- CPython: 3.10.12
- PyPy: 7.3.9
- NodeJS: 12.22.9

小悟空，对于这个测试你又作何解释。

悟空摸了摸头，道: 你这真的嘛，这是为什么呢？ 按道理Java应该比C++要慢的。

龟仙人笑了笑道:  小悟空，你可还记得《编译原理》,  让我们回忆一下《编译原理》，在第八章就开门见山的说: "具有挑战性的是，从数学上讲，为给定源程序生成一个最优的目标程序是不可判定的问题，在代码生成中的很多子问题（比如寄存器分配）都具有难以处理的复杂性"。 这里的不可判定指的是不存在一个程序或算法，可以对所有可能的输入进行处理，并且始终产生正确的结果。这么说可能有点抽象，让我来举一个例子：

```java
static void test(Object input) {  
 if (input == null) {  
 return;  
 }  
 // do something  
}
```

如果这个函数在实际调用的过程中input执行了一万次还是为空，那么这个判断还需要走嘛，或者说这个判空是开发者为了提升程序的健壮性，到后面程序的输入不可能为空了，那这个input还有存在的必要嘛，所以产生的机器码要依附于输入，但是输入又无法枚举，所以这是不可判定问题。那么回到你上面描述的，将字节码翻译成指定的机器码交付给计算机执行的这个工作叫解释器，所以从这个角度来说，Java通常是不如C++的，但是Java为了提升性能引入了JIT(Just In Time)编译器，也就是说当程序运行的时候，解释器首先发挥作用，代码可以直接执行，随着时间推移，JIT编译器发现某段代码是热点代码，这个热点代码姑且可以理解为执行频率比较高的的代码，将越来越多的代码优化成本地机器码，来获取更高的执行效率，然后缓存起来，下次就不用翻译了。还是那个上面的例子，如果一段时间里面上面的test函数的输入又不为空了，那么这个时候的优化失灵了，JIT编译器会执行逆优化，如果你用阿里出品的Arthas工具进行分析的话，就会发现CPU会触发Deoptimization::uncommon_trap，这个也就是JIT编译器触发了逆优化，这个阶段控制流重新归于解释器或其他备用代码路径，以便重新执行相应的代码。这种方式也被称为JIT执行。而C++一开始选择最终的输出产物是相应平台的机器码，这种方式的优点是启动速度快，资源占用略低。JIT相对于AOT的优势是，就是能够根据运行时的某些指导性信息进行更深层次的优化，这也一般被称之为PGO，也就是Profile-Guided Optimizations，PGO 往往需要根据统计信息来优化，在程序中收集统计信息无疑会付出额外的成本。但这也不意味着AOT不能使用PGO，我们也可以先AOT，然后在真实场景下面或者仿真情景下面运行之后，根据运行的实际情况来重新编译程序，但是AOT+PGO非常耗时，在现实中很少有人会选择AOT+PGO部署。 JIT并非一无是处，从一些文章可以看到，有人认为因为Java不能执行机器码所以慢，这是一种谬误，JIT也有一些AOT所比不上的优点, 比如基于运行时环境的优化，JIT进行编译的时间节点要更晚，这意味这JIT编译器比AOT编译器掌握跟多知识，像运行时设备的硬件信息等等，JIT可以完全利用这些信息优化生成的代码。再比如，更加灵活的动态功能，比如热刷新，也就是在运行时不停机更新函数的功能，都是JIT容易做而AOT很难做到的。 而AOT相对于JIT，启动时间很短，因为不需要解释这个过程，因为是机器码，所以难以反编译，因为要考虑做更复杂的优化，所以编译时间要更多，所以AOT的时间可以理解为花在编译时，而JIT的时间花在运行时。现在两种AOT和JIT也在尝试融合，比如Graal在AOT的时候使用预训练的机器学习模型自动推导PGO所需要的行为数据，JIT也在通过缓存行为数据和生成的机器码，将部分运行时计算提到运行来加快预热。Azul的Cloud NativeComplier将JIT服务挪移到单独的编译服务器，通过在多台运行服务器间共享Profile和生成的机器码以减少重复工作，同时也允许进行更占资源的复杂优化。说到这里，我想起一种说法，Java算编译型语言还是解释型语言，现在来看来这并不是一个很好的概念，编译和解释尝试在进行融合，Java在Graal上是编译型，在普通的JDK也不能完全算解释型。同样的一直以来大家都认为C++是编译型语言，但是对于大型项目，编译太耗时了，所以C++也在引入JIT，在C++中做JIT并不是一件新鲜的事情，下面是对一些对C++中引入JIT的讨论: 

1. https://llvm.org/devmtg/2019-04/talks.html#Talk_15
2. https://root.cern/cling/
3. https://github.com/RuntimeCompiledCPlusPlus/RuntimeCompiledCPlusPlus/wiki/Alternatives

C++的标准委员会收到了引入JIT的提案, 也就是: P1609R1  C++ Should Support Just-in-Time Compilation。在这个议案里面C++跑了性能测试，做了矩阵运算，在矩阵比较小的时候，JIT的执行速度要显著由于AOT生成的机器代码，随着矩阵规模变大，JIT的优势在逐步变小，JIT的优势变得不那么明显，这是因为大矩阵的操作涉及更多的内存访问和计算，更多的依赖于内存和处理器的并行能力。所以你看随着问题规模的变大，性能开始受到越来越多因素的影响。

悟空点了点头:  那现在编译器变得这么聪明了，C++还有什么呢？  我好像听到过一个定律，就是说给JVM的内存不能超过32G，超过了压缩指针会失效，我想这是Java的局限，在大内存下面表现不佳。

龟仙人哈哈大笑:  悟空看东西要看本质，首先我们考察这个问题，即这个结论是怎么得出来的，不要老是记结论，我们要更深度的考察这些结论是如何得出的，这会让我们看到问题的本质，你听过内存对齐(memory alignment)嘛？

小悟空: 我好像学C++的时候，听过这个词，后面主要用Java，这个词就忘记什么含义了。

龟仙人: JVM真的好啊，屏蔽的真好，悟空，听我细细道来，首先我们要考察内存对齐是什么，对于内存来说，基本上程序员将其视为一个字节数组 , 每个字节可以存储8位的信息，如下图所示:

![](https://a.a2k6.com/gerald/i/2023/09/26/qivw.png)

每个字节都有唯一的地址，用来和内存中的其他字节做区分，如果内存有n个字节，那么可以认为地址的范围为0-n-1。可执行程序由代码(原始C程序中与语句对应的机器指令)和数据(原始程序中的变量)两部分构成。说到这里，悟空你想到了什么?

小悟空: 我想起来了，程序中的每个变量占有一个或多个内存字节，把第一个字节的地址称为是变量的地址。比如，变量i占有的字节从地址2000到2001，所以变量i的地址是2000。这就是指针的出处，虽然用数表示地址，但其取值范围可能超过普通整型的范围，所以这里我们不能用普通整型变量来存储地址。这里我们就创造了一个变量类型，也就是指针变量来存储地址。

龟仙人:  不错，不错，记性还不错。在程序员的角度来看，内存是一个字节数组，但是对于CPU来说并不是以字节为单位读写内存，CPU以2、4、8、16或甚至32字节为单位访问内存，对齐问题是严重的，这会对性能造成重大影响，你知道Java领域大名鼎鼎的Disruptor和Netty就有效的利用了内存对齐技术来做加速，但是在Java中我们不能自由控制对齐的粒度，我说的意思是，自由的控制内存对齐粒度，某个对象我希望是8字节对齐，但是某个对象我希望16字节对齐，在Java里面是做不到这么强有力的控制的，Java统一执行内存对齐，Java默认按8字节对齐，也就是说一个对象的占用空间，必须是8字节的整数倍，不足的话就必须填充到8的整数倍，但在C++里面我就可以做到这么强有力的控制: 

```c++
#define SIZE 5000000000 
double *array1 = new double[SIZE]; 
// aligned_alloc 是请求分配内存，按几字节对齐。
// aligned_alloc()一般使用在intel的AVX指令集中，从内存中初始化向量(也就是矩阵)
double *array2 = (double*) aligned_alloc(64, SIZE * sizeof(double)); 
 
// Initialize arrays with random values 
for (size_t i = 0; i < SIZE; ++i) { 
	array1[i] = rand(); 
	array2[i] = rand(); 
} 
 
using clock = std::chrono::high_resolution_clock; 
 
double alpha = 5.4; 
auto begin1 = clock::now(); 
#pragma omp simd 
for (size_t i = 0; i < SIZE; ++i) { 
	array1[i] *= alpha; 
} 
auto end1 = clock::now(); 
 
auto begin2 = clock::now(); 
#pragma omp simd aligned(array2:64) 
for (size_t i = 0; i < SIZE; ++i) { 
	array2[i] *= alpha; 
} 
auto end2 = clock::now(); 
 
delete[](array1); 
free(array2); 
 
auto time1 = end1 - begin1; 
auto time2 = end2 - begin2
```

这个代码很简单，结果是time2小于time1, 这是为什么呢？  这得益于现代处理器的进化，比较新的CPU有对应的AVX指令扩展，可以加速处理矩阵运算，但遗憾的是，它只能用于地址被64整除的数据，在循环一里面，必须在运行时检查对齐情况，只有按64字节对齐的元素才使用到AVX，随着矩阵的变大，需要更多的逻辑检查，这相当浪费CPU的算力。这就是C++的自由，注意看free函数，计算完成之后，我们直接对这些内存进行释放，如果在Java里面这类大对象对于GC来说是个难题，回收起来会稍微有些困难，但是对于成熟的ZGC来说，这个时间已经被压缩到亚毫秒级别了，基本可以忽略不计。但在Java里面无法我们自由的控制对齐的粒度，我们只能通过-XX:ObjectAlignmentInBytes控制对齐粒度，默认是8，那这意味着对于向量运算来说，我想做单独的优化，那整个JVM内的对象占用字节都会变大，这太僵硬了，一点都不灵活，但JDK也在不断的改进，这也就是JDK新特性中的Vector API , 这也就是上面的性能测试中AVX2的意思，代表使用CPU加速处理向量运算，这个特性孵化了六次了，真是没完没了。我们接着说为什么我们需要内存对齐，让我们举一个例子，首先是从地址0读取4个字节进入到CPU的寄存器，然后从地址1也读四个字节也进入到一样的寄存器里面。让我们首先从内存访问粒度为1个字节的处理器开始，悟空，你觉得流程是怎样的。

悟空:  我觉得就是先从地址0开始读4个字节，然后再从地址1读四个字节啊，像下面这样:

![](https://a.a2k6.com/gerald/i/2023/09/26/6zbgc.png)

也就是说从地址0读四个字节到寄存器和从地址1读四个字节所需要的次数相等。

龟仙人:   天真的程序员就是这么理解CPU读写数据的流程的，让我们以非对齐访问为例，为了讨论问题方便，我们扩展一下对CPU对内存的访问能力，让CPU对内存的访问粒度上升到四个字节，再假定CPU支持非对齐内存访问方式，因为一些CPU是不支持非对齐的方式访问内存的，现在我们还是将内存看做一个数组:

![](https://a.a2k6.com/gerald/i/2023/09/29/3kvbn.jpg)

首先CPU接收到指令请求从地址0读取两个字节，然后CPU收到指令从地址2开始读取4个字节，但是CPU不具备任意访问内存位置的能力，由于地址2位于0-3这个内存边界内，也就是CPU只能从内存访问粒度的整数倍开始读取，所以CPU只能从地址0开始读一次，然后读到四个字节，除此之外还要扔掉两个不要的字节，然后再从地址2读一次，再读四个字节。这样就是两次内存读取，如果是内存对齐，就只需一次，地址2就变成了地址四。悟空，现在你懂，内存对齐的必要性了嘛？

悟空点了点头:   看起来内存对齐很有必要，也就是说对于一个32位的系统来说，选择4字节对齐，对于64位的系统来说选择8字节对齐是必要的，是这样嘛。但是我也有问题，比如吧，对于windows上来说，32位应用程序还是跑在64位应用程序，这不会给CPU和操心系统带来麻烦嘛？

龟仙人点了点头: 你提的问题很好，原本是intel是不打算兼容的，AMD兼容了，所以现在流行的指令集应当是AMD64。在我上面的描述中，CPU是直接访问内存的，但是实际上来说，CPU直接访问的是自己的缓存，也就是L1、L2、L3 cache，CPU发起访问内存指令的时候，事实上是以块为单位访问内存的，这也被称为缓存行，数据会首先到达CPU的三级缓存中，三级缓存大一些，因此可以访问更多的内存，在Intel家族中，这些块的大小为64byte，我们可以说对于几乎桌面级处理器都是这样，Apple M1会特殊一些，它的缓存行更大为128byte。

悟空: 那这跟压缩指针有什么关系?  我不理解

龟仙人: 不错，不错，跟你聊了这么多，还记得上面的内容，现在我们有了内存对齐必要性，现在让我们重新审视为什么我们需要压缩指针，现在64位计算机设备已经很普及了，内存也很便宜，但是在64位系统推向市场的时候，32位系统还在运行，运行在32位系统上面的程序也大概率是32位应用，在一个32位系统上按4字节对齐，而在64位系统上按8字节对齐，这么看迁移应用我这个堆的内存还要上升了？ 这可能是32位开发者不能接受的，那怎么办呢，观察规律呗，JVM执行了严格对齐，所以每个对象的首地址必然是8的倍数，那么在二进制里面8的倍数后3位必然是0，所以有2的32次方乘以8,也就是32GB。

悟空摇了摇头:  2的32次方乘以8b才是32GB，32位后三位为0，这里面8的倍数的数字不才是2的29次方，再乘以8，才2的32次方，你算的不对。

龟仙人笑了笑: 华点，你发现了盲僧，提出问题代表你不是完全在接受我的概念，而是自己也在思考，其实这里面混淆了一些概念，让我们从一个对象的基本组成开始, 让我们来审视这个对象, 一个对象在内存中由实例数据和对象头组成，而对象头又有两部分组成:

- Mark Word(32位JVM占用4个字节，64位上占用8个字节)
- KClass Word(32位JVM占用4个字节，64位上占用8个字节)

压缩指针压缩的就是KClass Word部分，这部分也被称为对象指针，JVM通过这个指针确定这个对象属于哪一个实例，我们可以认为这个对象指针存储了对象的起始地址，但是在64位系统上，JVM执行8字节对齐，所以对象的起始地址必然是8的倍数，8的倍数后三位为0，那么也就是说对于35位的0和1，一共有2的32次方个8的倍数个地址，2的32次方，也就是32位存储，描述4GB存储，于是我们就可以用32位来表示35位的地址，在每个32位地址后面补三个0即可，我们又进一步假每个对象刚好占8个字节，于是2的32次方个地址代表2的32次方个对象，于是也就是2的32次方乘以8b，于是得到32GB。超过32GB之后也就是说会有地址超出35位描述的地址范围，于是压缩指针失效，于是KClass Word接着占用8个字节，但其实占用8个字节，其实问题也不大，因为Java过去的垃圾回收器不擅长管理大内存，一般超过8G堆内存就会有显著的性能下降，说的就是你CMS垃圾回收器。接下来有请G1出场，G1采取了分区的思想，但是场景也是面对的是32GB以下的内存，对于更高的内存，如果不调整默认region的大小，那么region会非常多，每个region的分析数据导致管理的GC信息也会占用很大的内存。但是如果调高了region的大小，G1 GC为了达到只回收部分region，每个region都需要RememberSet来记录各个Region之间的引用，这个大小会随着分区大小与每个对象大小及对象间引用复杂度上升。这个内存开销还是很大的。ZGC使用了着色指针技术，不支持压缩指针，但是性能也很好，也可以将GC暂停时间压缩在亚毫秒级别。所以这个压缩指针的话，还是看你选择的垃圾回收器，oracle发了一篇解释压缩指针，有人在下面问能否提高压缩指针的范围限制，答案也是可以的，默认内存对齐是8字节，那么压缩指针是压了一半，那么我们提升对齐尺度，就能增加压缩指针的内存范围，现在我们将内存对齐提升到24，通过-XX:ObjectAlignmentInBytes这个参数，然后压缩一半就是12,也就是96GB，如果我们设置为对齐尺度为16，也就是64GB才会失效。但是不同的应用性能受制于因素并不完全在内存，所以垂直扩展有的时候并不如水平扩展。

悟空点了点头: 那前面提到了刺激JIT编译器，那么该如何刺激呢？ 

龟仙人笑道: 这也是我本篇要交给你的，在Java领域有一个对应的库叫Java Microbenchmark Harness，我们以用这个框架来对我们的Java代码进行性能测试, 但是这个库的文档比较少，但是不要害怕我会教你，这个库在高版本已经被集成进JDK里面，但在JDK 8里面，还是要引入一下这个库的依赖:

```xml
<dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-core</artifactId>
            <version>1.36</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-generator-annprocess</artifactId>
            <version>1.36</version>
</dependency>
```

这次我想测试的是+号拼接和StringBuilder的性能区别:

```java
public class JMHTest {
    // 这个注解表示请求这个方法对进行JMH测试
    // 一次测试跑一个，先跑+，将这个方法注释跑stringBuilderContact
    @Benchmark
    public String  stringContact() {
        String s = "hello world";
        for (int i = 0; i < 100000 ; i++) {
            s = s + i;
        }
        return s;
    }
	
    @Benchmark
    public String  stringBuilderContact() {
        String s = "hello world";
        StringBuilder stringBuilder = new StringBuilder(s);
        for (int i = 0; i < 100000 ; i++) {
            stringBuilder.append(s);
        }
        return stringBuilder.toString();
    }

    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(JMHTest.class.getSimpleName())
                .build();
        new Runner(opt).run();
    }
}
```

输出为: 

```java
# JMH version: 1.36 JMH版本
# VM version: JDK 1.8.0_202, Java HotSpot(TM) 64-Bit Server VM, 25.202-b08 Java版本
# VM invoker: C:\Program Files\Java\jdk1.8.0_202\jre\bin\java.exe
# VM options: -javaagent:D:\Program Files\JetBrains\IntelliJ IDEA 2020.3.4\lib\idea_rt.jar=56750:D:\Program Files\JetBrains\IntelliJ IDEA 2020.3.4\bin -Dfile.encoding=UTF-8
# Blackhole mode: full + dont-inline hint (auto-detected, use -Djmh.blackhole.autoDetect=false to disable)
# Warmup: 5 iterations, 10 s each 表示在真正测试方法之前,会对方法进行越热，跑五次，每次跑10s
# Measurement: 5 iterations, 10 s each 实际的基准测试  跑五次，然后每次跑10s
# Timeout: 10 min per iteration How 每次迭代执行时间等待多久
# Threads: 1 thread, will synchronize iterations 每次迭代将在一个线程内进行
# Benchmark mode: Throughput, ops/time 模型表示处于吞吐量，也就是每秒执行多少次。
# Benchmark: com.example.demo.JMHTest.stringContact
```

这个测试交给你看看，今天就教到这里。悟空你好好体会一下，我教你的。

悟空: 武天老师，我还想听。

龟仙人: 程序不是全部，我想起C++之父说过: 不要过于专一，要有灵活性，记住职业和工作是一件长期的事情，要试着表达自己的想法，如果你不表达，过于沉浸在自己的世界跟玩数独没什么区别。一定要交流，不要认为写出最好的代码就能改变世界。所以多去交流吧，悟空，世界是广阔的。

## 写在最后

这篇文章本身只是打算介绍一些JMH，然后介绍JMH之前又想到了JIT、AOT，引入JIT和AOT之后，又想起了压缩指针，想到压缩指针之后又遇到了内存对齐，这是始料未及的，在最初我写这篇文章的时候，我只是打算介绍一下JMH，但是引入JMH并没有像之前的文章的那样，遵循是什么，能做什么，该怎么用这条线。而是尝试更为发散的思路，像是一张网一样。这里也想融入自己的一些思考，前段时间看到C++之父的建议，感觉颇为有用，索性将这部分内容也糅合了进来，最开始以为很快就能弄完，后面发现就算是JMH的铺垫弄完，JMH的例子也要与编译器相关，编译器相关又要发散许多，这次我们就小小的引入，让JMH露个脸，后面会尝试更加轻松的文笔。

## 参考资料

[1] Linux网络编程基础API https://swordandtea.github.io/2020/11/06/hight_performance_linux_coding/Linux%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80API/

[2] posix是什么都不知道，还好意思说你懂Linux？https://zhuanlan.zhihu.com/p/392588996

[3] leibniz-benchmark  https://github.com/Glavo/leibniz-benchmark

[4] 《编译原理》 第二版 机械工业出版社

[5] 对比JIT和AOT，各自有什么优点与缺点?  https://www.zhihu.com/question/23874627/answer/3213118573

[6]  C++ 标准委员会 https://github.com/Cpp-Club/Cxx_HOPL4_zh/blob/main/03.md#3-c-%E6%A0%87%E5%87%86%E5%A7%94%E5%91%98%E4%BC%9A

[7] Purpose of memory alignment  https://stackoverflow.com/questions/381244/purpose-of-memory-alignment

[8] Data alignment: Straighten up and fly right https://web.archive.org/web/20080607055623/http://www.ibm.com/developerworks/library/pa-dalign/

[9] 《C语言程序设计方法》

[10] Netty和Disruptor的cache_line对齐实践 https://plantegg.github.io/2022/05/05/Netty%E5%92%8CDisruptor%E7%9A%84cache_line%E5%AF%B9%E9%BD%90%E5%AE%9E%E8%B7%B5/

[11] 浅谈CPU内存访问要求对齐的原因 https://yangwang.hk/?p=773

[12] CPU and Data alignment https://stackoverflow.com/questions/3025125/cpu-and-data-alignment

[13] Data structure size and cache-line accesses https://lemire.me/blog/2022/06/06/data-structure-size-and-cache-line-accesses/
