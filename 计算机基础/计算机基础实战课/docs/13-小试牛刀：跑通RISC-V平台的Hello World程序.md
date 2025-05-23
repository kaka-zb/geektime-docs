你好，我是LMOS。

在上一课中，我们一起约定了主环境，安装了编译工具和依赖库，构建了交叉编译RISC-V工具链。

今天我们继续构建RISC-V版的模拟器QEMU（代码你可以从[这里](https://gitee.com/lmos/Geek-time-computer-foundation/tree/master/lesson12~13)下载），让它成为“定制款”，更匹配我们的学习需要。为此，我们需要设置好主环境的环境变量，安装好VSCode及其插件，这样才能实现编辑、编译、运行、调试RISC-V程序的一体化、自动化。

话不多说，我们开始吧。

## RISC-V运行平台

有了上节课成功构建好的交叉编译器，有很多同学可能按捺不住，急着想写一个简单的Hello World程序，来测试一下刚刚构建的交叉编译器。

恕我直言，这时你写出来的Hello World程序，虽然会无警告、无错误的编译成功，但是只要你一运行，铁定会出错。

这是为什么呢？因为你忘记了交叉编译器，生成的是RISC-V平台的可执行程序，这样的程序自然无法在你的宿主机x86平台上运行，它只能在RISC-V平台上运行。

摸着自己的荷包，你可能陷入了沉思：难道我还要买一台RISC-V平台的计算机？这样成本可太高了，不划算。

贫穷让人学会变通，为了节约成本，我们希望能用软件模拟RISC-V平台。嘿！这当然可以，而且前辈们，早已给我们写好了这样的软件，它就是QEMU。

## 揭秘QEMU

什么是QEMU？QEMU是一个仿真器或者说是模拟器软件，与市面上BOCHS类似，由软件来实现模拟。

QEMU就像计算机界的“孙悟空”，变化多端，能模拟出多种类型的CPU，比如IA32、AMD64、ARM、MIPS、PPC、SPARC、RISC-V等。QUEM通过动态二进制转换来模拟CPU。除了CPU，它还支持模拟各种IO设备，并提供一系列的硬件模型。这使得QEMU能模拟出完整的硬件平台，使得QEMU能运行各种操作系统，如Windows和Linux。

你可以把QEMU当做一个“双面间谍”，因为在它上面运行的操作系统，也许还认为自己在和硬件直接打交道，其实是同QEMU模拟出来的硬件打交道，QEMU再将这些指令翻译给真正硬件进行操作。通过这种模式，QEMU运行的操作系统就能和宿主机上的硬盘、网卡、CPU、CD-ROM、音频设备、USB设备等进行交互了。

由于QEMU的以上这些特点，导致QEMU在宿主平台上可以模拟出其它不同于宿主平台的硬件体系，这是QEMU的优点。

不过，由于是用了软件来实现的模拟，所以性能很差，这也是QEMU的缺点。正因为这个缺点，后来就出现了 **QEMU和KVM结合使用的解决方案**。

KVM基于硬件辅助的虚拟化技术，主要负责比较繁琐的CPU和内存虚拟化，而QEMU则负责 I/O设备的模拟，两者合作各自发挥自身的优势，成就了强强联合的典范。

回归主题，关于QEMU，现阶段你最需要记住的就是，它有**两种主要工作模式：系统模式和用户模式。**

在系统工作模式下，QEMU能模拟整个计算机系统，包括CPU及其他IO设备。它能运行和调试不同平台开发的操作系统，也能在宿主机上虚拟不同数量、不同平台的虚拟电脑。而在用户工作模式，QEMU能建立一个普通进程，运行那些由不同体系处理器编译的应用程序，比如后面我们要动手编写的RISC-V版的Hello World程序。

## 构建我们的“定制款”QEMU

说了这么多，其实是想让你更加了解QEMU。

下面我们来办正事儿——构建适合我们的QEMU，如果我们不是有特殊要求——模拟RISC-V平台且带调试功能的QEMU，用不着亲自动手去构建，只需要一条安装指令就完事了。

构建QEMU用四步就能搞定：首先下载QEMU源代码，接着配置QEMU功能选项，然后编译QEMU，最后安装QEMU。

我们需要从QEMU[官网](https://www.qemu.org/)上[下载稳定版本](https://download.qemu.org/qemu-6.2.0.tar.xz)的QEMU源代码。如果你和我一样，觉得在浏览器上点来点去非常麻烦，也可以在切换到RISCV\_TOOLS目录的终端下，输入如下指令：

```plain
wget https://download.qemu.org/qemu-6.2.0.tar.xz #下载源码包
tar xvJf qemu-6.2.0.tar.xz #解压源码包
```

这里跑完第一条指令以后，下载下来的是压缩的QEMU源码包。所以，在下载完成后，你要用第二条指令来解压。

由于[上节课](https://time.geekbang.org/column/article/554509)我们构建RISC-V工具链时，已经统一安装了构建QEMU所需要的相关依赖库，所以这里就不用安装相关依赖库了。

解压成功后，我们就要开始配置QEMU的功能了。同样，为了不污染源代码目录，我们可以先在qemu-6.2.0目录下，建立一个build目录，然后切换到build目录下进行配置，输入如下指令：

```plain
mkdir build #建立build目录
cd build #切换到build目录下
../configure --prefix=/opt/riscv/qemu --enable-sdl --enable-tools --enable-debug --target-list=riscv32-softmmu,riscv64-softmmu,riscv32-linux-user,riscv64-linux-user #配置QEMU
```

上述配置选项中，–prefix表示QEMU的安装目录，我们一起约定为“/opt/riscv/qemu”目录，–enable-sdl表示QEMU使用sdl图形库， --enable-tools表示生成QEMU工具集，–enable-debug表示打开QEMU调试功能。

最重要的是 **--target-list** 这个选项，它表示生成QEMU支持的模拟目标机器。不同选项所支持的平台不同，我们的选择如下表所示：

![图片](https://static001.geekbang.org/resource/image/18/7d/1893274bf776382cd139692e9aaf397d.jpg?wh=1920x804)

如果你什么都不选的话，它会默认生成QEMU支持的所有平台。按前面我们讲的操作配置，配置成功后，build目录下会生成后面截图里展示的文件和目录。

![图片](https://static001.geekbang.org/resource/image/61/22/612cb5524109cc97a8ce1e4b2810d922.jpg?wh=1920x1527)

配置好功能选项之后，下一步就是编译QEMU了。只要配置成功了，编译这事儿就非常简单了，我们只要输入如下指令，然后交给计算机编译就好了。别忘了等待期间泡杯茶，不知道你会不会像我一样哼起那首歌：“世上有没有人，安静的等待你，一直不愿回神……”

```plain
sudo make -j8
```

最后就是安装QEMU，经过漫长等待以后，我们终于迎来编译的成功。这时，你还需要输入如下指令进行安装。

```plain
sudo make install
```

这里说明一下，QEMU不像RISC-V工具链那样，会在编译结束后自动安装，它需要手动安装。

我们在终端中切换到“/opt/riscv/qemu/bin”目录下，执行如下指令：

```plain
qemu-riscv32 -version && qemu-riscv64 -version && qemu-system-riscv32 -version && qemu-system-riscv64 -version
```

上述指令会输出qemu-riscv32、qemu-riscv64、qemu-system-riscv32、qemu-system-riscv64的版本信息，以证明能运行RISC-V平台可执行程序的QEMU构建成功。你可以对照一下后面的截图。

![图片](https://static001.geekbang.org/resource/image/5f/a0/5f293367bb0821cyy49ee1d3a2c4bfa0.jpg?wh=1920x1162)

到这里，RISC-V平台的编译环境和执行环境已经构建完成，并且能生成和执行32位或者64位的RISC-V平台的可执行程序，无论是RISC-V平台的应用程序，还是RISC-V平台的操作系统。

### 处理环境变量

不知道你发现了没有？我们运行QEMU和RISC-V工具链相关的程序，都要切换到/opt/riscv/xxxx/bin目录中才可以运行，而不是像Linux中的其它程序，可以直接在终端中直接运行。

革命还未成功，我们还得努力。这是因为我们没有将QEMU和RISC-V工具链的安装目录，加入到Linux的环境变量中。

接下来我们就开始处理环境变量，修改环境的方法有好几种。这里我为你演示比较常用的一种，那就是在当前用户目录下的“.bashrc”文件中，加入相关的指令。

这里说的“当前用户的目录”就是在终端中执行"cd ~" 指令。怎么操作呢？我们切换到当前用户目录下，来执行这个指令。然后，在文件尾部加上两行信息就行了。具体指令如下所示：

```plain
cd ~ #切换到当前用户目录下
vim ./.bashrc #打开.bashrc文件进行编辑

#在.bashrc文件末尾加入如下信息
export PATH=/opt/riscv/gcc/bin:$PATH
export PATH=/opt/riscv/qemu/bin:$PATH
```

上述操作完成以后，你会看到下图所示的结果：  
![图片](https://static001.geekbang.org/resource/image/ef/3c/eff44c66b93b1aa531b46b47f2095d3c.jpg?wh=1920x1162)

随后，我们按下键盘上ESC键，接着输入":wq"以便保存并退出Vim。这样操作后，你会发现环境变量并没有生效。

这里还差最后一步，我们在终端中输入如下指令，让环境变量生效：

```plain
source ./.bashrc
```

现在，你在任何目录之下输入QEMU和RISC-V工具链相关的程序命令，它们就都可以正常运行了。

### 安装VSCode

有了QEMU和RISC-V工具链相关的程序命令，我们虽然可以编译调试和执行RISC-V平台的程序了，但是必须在终端中输入多条指令，才能完成相关的工作。

这对于很多同学来说，肯定觉得很陌生，特别是在图形化盛行的今天，我们更期待能有个轻量级的IDE。

这里我们约定使用VSCode，它安装起来也很简单。在 [VSCode官网](https://code.visualstudio.com/Download)上下载deb包，下载后双击deb安装，或者切换到刚才下载VSCode目录的终端中，输入如下指令就行了。

```plain
sudo apt-get install -f *.deb
```

安装好后，在你的桌面会出现VSCode图标，双击打开后的页面如下所示：

![图片](https://static001.geekbang.org/resource/image/e8/01/e80a0f46c534d870483fb74df77ecb01.jpg?wh=1920x1019)

不过，有了VSCode我们目前只能写代码，还不能编译和调试代码，所以需要给VSCode安装C/C++扩展。我们只需打开VSCode，按下ctrl+shift+x，就能打开VSCode的扩展页面，在搜索框中输入C/C++就可以安装了，如下所示：

![图片](https://static001.geekbang.org/resource/image/65/fc/65c609a360d33dfa7b1399c115yy49fc.jpg?wh=1920x1019)

至此，我们的VSCode及其需要的扩展组件就安装完成了。

下一步，我们还需要在你的代码目录下建立一个.vscode文件夹，并在文件里写上[两个配置文件](https://gitee.com/lmos/Geek-time-computer-foundation/tree/master/lesson12~13/.vscode)。这两个配置文件我已经帮你写好了，如下所示。

![图片](https://static001.geekbang.org/resource/image/7f/f1/7fd7yycd59e982ec7b104b47cf5009f1.jpg?wh=1920x1019)

在.vscode文件夹中有个tasks.json文件，它主要负责完成用RISC-V编译器编译代码的功能，还有用QEMU运行可执行文件的功能。

我们先说说这里的编译工作是怎么完成的。具体就是通过调用make，读取代码目录中的Makefile脚本，在这个脚本中会调用riscv64-unknown-elf-gcc完成编译。等编译成功后，才会调用QEMU来接手，由它运行编译好的可执行程序。代码注释已经写得很清楚了，你可以停下来仔细看看。

不过，tasks.json文件虽然解决了编译与运行的问题，但是它也是被其它文件调用的。被谁调用呢？那就是我们的调试配置文件launch.json文件，它用于启动调试器GDB，只不过这里启动的不是宿主平台上的GDB，而RISC-V工具链中的GDB。其内容如下所示：

![图片](https://static001.geekbang.org/resource/image/80/5b/80870a7d1a7f35b11228f6e7810c145b.jpg?wh=1920x1019)

当我们写好代码后，按下F5键后，VSCode就会执行launch.json文件的调试操作了。这里调试器和要调试的可执行程序已经制定好了。不过由于preLaunchTask的指定，开始执行调试命令之前，VSCode会首先执行tasks.json文件中的操作，即编译和运行。

### 运行Hello World

下面我们一起来写下那个著名的程序——Hello World！写好后，在main函数所在的行前打上一个断点，按下F5键，就会看到如下界面。

![图片](https://static001.geekbang.org/resource/image/9c/7b/9c14c4def94c7d719f732822b7fc807b.jpg?wh=1920x1019)

如果不出意外，哈哈，放心，按我提供给你的步骤，也出不了意外，你一定会看到以上界面。

我们重点来观察红色方框中的信息，可以查看代码变量值、CPU的寄存器值、函数的调用栈、断点信息、源代码以及程序执行后在VSCode内嵌终端中输出的信息。有了这些信息，我们就能清楚地看到一个程序运行过程的状态和结果。

走到这里，我们的定制款QEMU以及VSCode就搭好了，可以去图形化编辑、编译、运行和调试RISC-V平台的可执行程序了。

## 重点回顾

好了，我们的RISC-V平台的Hello World，也是我们在宿主机上开发的第一个非宿主机的程序，现在已经成功运行，这说明我们之前的工作完成得很完美，今天的课程不知不觉也接近了尾声。

下面来回顾一下，这节课我们都做了些什么。

首先，我们构建了能运行RISC-V可执行程序的QEMU模拟器，这使得我们不必购买RISC-V平台的机器，就能在宿主机上运行RISC-V可执行程序。这不但大大方便了我们的开发工作，而且节约了成本。

然后，我们处理了环境变量，方便我们在任何目录下，都可以随意使用RISC-V工具链中的命令和QEMU相关的命令。

最后，我们安装了VSCode，还在其中安装C/C++扩展并对其进行了相应的配置。以后我们在VSCode图形环境下编写代码、编译代码和调试代码，就能一气呵成了。

这节课的要点我整理了导图，供你参考。

![图片](https://static001.geekbang.org/resource/image/ba/be/ba36676884beyy0be6ec1f267b6215be.jpg?wh=1920x1430)

恭喜你坚持到这里，通过两节课的内容，我们拿下了开发环境这一关，这对我们后续课程中的实验相当重要。下一模块讲解和调试RISC-V汇编指令的时候，你会进一步体会到环境搭建好的便利，先好好休息一下，咱们下节课见。

## 思考题

处理环境变量后为什么要执行source ./.bashrc，才会生效？

欢迎你在留言区提问或者晒晒你的实验记录。如果觉得有收获，也推荐你把这节课分享给你的朋友。
<div><strong>精选留言（15）</strong></div><ul>
<li><span>廖雪峰</span> 👍（10） 💬（4）<p>如果有不想编译的同学，可以按照以下步骤运行：

1. 安装Ubuntu 22.04

2. 安装编译环境：
$ sudo apt-get install autoconf automake autotools-dev curl python3 libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev

3. 直接安装riscv-gcc发行包：
$ sudo apt install gcc-riscv64-linux-gnu gcc-riscv64-unknown-elf picolibc-riscv64-unknown-elf

4. 直接安装qemu发行包：
$ sudo apt install qemu-user qemu-system-misc

5. 编写hello.c
#include&lt;stdio.h&gt;
int main()
{
	printf(&quot;Hello, riscv!\n&quot;);
	return 0;
}

6. 编译：
$ riscv64-linux-gnu-gcc hello.c -o hello

7. 运行：
$ qemu-riscv64 hello
qemu-riscv64: Could not open &#39;&#47;lib&#47;ld-linux-riscv64-lp64d.so.1&#39;: No such file or directory

如果报错找不到&#47;lib&#47;ld-linux-riscv64-lp64d.so.1，是因为这个文件实际上在&#47;usr&#47;riscv64-linux-gnu&#47;lib下，加个参数运行：

$ qemu-riscv64 -L &#47;usr&#47;riscv64-linux-gnu hello
Hello, riscv!

最后，riscv64-unknown-elf-gcc编译还没搞定，正在找原因</p>2022-08-24</li><br/><li><span>光华路小霸王</span> 👍（1） 💬（1）<p>使用了 VS Code 的 Remote Development 远程调试，发现 F5 debug 回找不到编译器和 qemu，shell
登录可以，一通谷歌是  .bashrc 与 .bash_profile 的问题，把环境变量加到 bash_profile 就可以了，vscode 是运行在 login shell 的，加载的是 bash_profile ，不会加载  .bashrc 
refs: https:&#47;&#47;github.com&#47;microsoft&#47;vscode-remote-release&#47;issues&#47;854
</p>2022-09-02</li><br/><li><span>Liu Zheng</span> 👍（1） 💬（1）<p>纠正一下。在环境变量设置好之前，即使在`&#47;opt&#47;riscv&#47;qemu&#47;bin`目录下，也不能直接跑`
qemu-riscv32 -version &amp;&amp; qemu-riscv64 -version &amp;&amp; qemu-system-riscv32 -version &amp;&amp; qemu-system-riscv64 -version`. 而是需要`.&#47;qemu-riscv32 ...`.</p>2022-08-24</li><br/><li><span>TableBear</span> 👍（1） 💬（1）<p>source 的主要用途是执行文件并从文件加载变量及函数到执行环境。
~&#47;.bashrc文件中的环境变量已经在用户登录shell的时候加载进执行环境了，此时编辑不会触发加载。必须使用source或者重新登录才能触发重新加载</p>2022-08-24</li><br/><li><span>筱琲</span> 👍（0） 💬（1）<p>在build 目录下执行配置命令时，遇到几个包缺失，依次安装就好：
sudo apt install libglib2.0-dev libpixman-1-dev libsdl2-dev</p>2022-11-02</li><br/><li><span>overheat</span> 👍（0） 💬（1）<p>sudo make -j8, 这里应该不用sudo。</p>2022-09-22</li><br/><li><span>Geek_d47998</span> 👍（0） 💬（1）<p>tasks.json和launch.json在老师给的代码，gitee仓库下有，main.c文件里还需要放Makefile后才能按F5编译</p>2022-09-12</li><br/><li><span>jeigiye</span> 👍（0） 💬（1）<p>root@zgye-ubuntu:~&#47;test# cat hello.s 
	.file	&quot;hello.c&quot;
	.option nopic
	.attribute arch, &quot;rv64i2p0_m2p0_a2p0_f2p0_d2p0_c2p0&quot;
	.attribute unaligned_access, 0
	.attribute stack_align, 16
	.text
	.section	.rodata
	.align	3
.LC0:
	.string	&quot;Risc-V&quot;
	.align	3
.LC1:
	.string	&quot;Hello %s\n&quot;
	.text
	.align	1
	.globl	main
	.type	main, @function
main:
	addi	sp,sp,-16
	sd	ra,8(sp)
	sd	s0,0(sp)
	addi	s0,sp,16
	lui	a5,%hi(.LC0)
	addi	a1,a5,%lo(.LC0)
	lui	a5,%hi(.LC1)
	addi	a0,a5,%lo(.LC1)
	call	printf
	li	a5,0
	mv	a0,a5
	ld	ra,8(sp)
	ld	s0,0(sp)
	addi	sp,sp,16
	jr	ra
	.size	main, .-main
	.ident	&quot;GCC: () 12.1.0&quot;
root@zgye-ubuntu:~&#47;test# &#47;opt&#47;riscv&#47;qemu&#47;bin&#47;qemu-riscv64 hello
Hello Risc-V
root@zgye-ubuntu:~&#47;test#

ubuntu22.04上跑通。</p>2022-09-06</li><br/><li><span>Liu Zheng</span> 👍（0） 💬（3）<p>想问一下，https:&#47;&#47;gitee.com&#47;lmos&#47;Geek-time-computer-foundation&#47;blob&#47;master&#47;lesson12~13&#47;main.c#L7-8 这里面func和sumdata分别都是什么呢？没有看到哪个地方有定义这两个东西，跑make或者vscode里面按F5也是会报错。</p>2022-08-25</li><br/><li><span>LooMou</span> 👍（0） 💬（0）<p>window 的 wsl
Distributor ID: Ubuntu
Description:    Ubuntu 20.04.6 LTS
Release:        20.04
Codename:       focal

riscv-gnu-toolchain 我编译的是 2024.04.12-nightly，对应的 qemu 的版本是 8.2.1
wget https:&#47;&#47;download.qemu.org&#47;qemu-&lt;version&gt;.tar.xz
直接按步骤继续，编译成功，在 ~&#47;.bashrc 中加环境变量也要在 ~&#47;.profile 里加。

代码运行成功，调试控制台：
=thread-group-added,id=&quot;i1&quot;
Warning: Debuggee TargetArchitecture not detected, assuming x86_64.
=cmd-param-changed,param=&quot;pagination&quot;,value=&quot;off&quot;
0x000100c2 in _start ()

Breakpoint 1, main () at main.c:6
6	    int i = 250;

Breakpoint 2, main () at main.c:8
8	    return 0; 
终端输出：
 *  正在执行任务: make 

CC -[M] 正在构建... add.S
CC -[M] 正在构建... main.c
CC -[M] 正在构建... main.elf
 *  终端将被任务重用，按任意键关闭。 

 *  正在执行任务: echo Starting RISCV-QEMU&amp;qemu-riscv32 -g 1234 .&#47;*.elf 

Starting RISCV-QEMU
hello world i is 250 3 250!</p>2024-12-03</li><br/><li><span>Pony</span> 👍（0） 💬（0）<p>&#47;opt&#47;riscv&#47;gcc&#47;lib&#47;gcc&#47;riscv64-unknown-elf&#47;10.2.0&#47;..&#47;..&#47;..&#47;..&#47;riscv64-unknown-elf&#47;bin&#47;ld: error: &#47;opt&#47;riscv&#47;gcc&#47;lib&#47;gcc&#47;riscv64-unknown-elf&#47;10.2.0&#47;rv32i&#47;ilp32&#47;crtbegin.o: Mis-matched ISA version for &#39;i&#39; extension. 2.0 vs 2.1
&#47;opt&#47;riscv&#47;gcc&#47;lib&#47;gcc&#47;riscv64-unknown-elf&#47;10.2.0&#47;..&#47;..&#47;..&#47;..&#47;riscv64-unknown-elf&#47;bin&#47;ld: failed to merge target specific data of file &#47;opt&#47;riscv&#47;gcc&#47;lib&#47;gcc&#47;riscv64-unknown-elf&#47;10.2.0&#47;rv32i&#47;ilp32&#47;crtbegin.o
在make all的时候报这个错误是为什么</p>2024-11-15</li><br/><li><span>🔥Burn</span> 👍（0） 💬（0）<p>archlinux,在编译qemu时报了编译错误,麻烦老师看看什么问题:
lssh -lstdc++ -Wl,--end-group
&#47;usr&#47;bin&#47;ld: libcommon.fa.p&#47;ebpf_ebpf_rss.c.o: in function `ebpf_rss_load&#39;:
&#47;home&#47;burn&#47;riscv-gnu-toolchain&#47;qemu-6.2.0&#47;build&#47;..&#47;ebpf&#47;ebpf_rss.c:52: undefined reference to `bpf_program__set_socket_filter&#39;
collect2: 错误：ld 返回 1
[1926&#47;2548] Compiling C object libqemu-riscv64-linux-user.fa.p&#47;target_riscv_vector_helper.c.o
[1927&#47;2548] Compiling C object libqemu-riscv64-linux-user.fa.p&#47;target_riscv_translate.c.o
[1928&#47;2548] Compiling C object libqemu-riscv32-linux-user.fa.p&#47;target_riscv_translate.c.o
ninja: build stopped: subcommand failed.
make: *** [Makefile:162：run-ninja] 错误 1

</p>2023-05-18</li><br/><li><span>Geek_72a577</span> 👍（0） 💬（1）<p>qemu-6.2.0.tar.xz无法下载，有gitee替代地址的吗？</p>2023-03-14</li><br/><li><span>Dean🌞中偉</span> 👍（0） 💬（2）<p>gcc编译成功：
riscv64-unknown-elf-gcc -v
Using built-in specs.
COLLECT_GCC=riscv64-unknown-elf-gcc
COLLECT_LTO_WRAPPER=&#47;opt&#47;riscv&#47;gcc&#47;libexec&#47;gcc&#47;riscv64-unknown-elf&#47;10.2.0&#47;lto-wrapper
Target: riscv64-unknown-elf
Configured with: &#47;home&#47;adrain&#47;RISCV_TOOLS&#47;riscv-gnu-toolchain&#47;riscv-gcc&#47;configure --target=riscv64-unknown-elf --prefix=&#47;opt&#47;riscv&#47;gcc --disable-shared --disable-threads --disable-tls --enable-languages=c,c++ --with-system-zlib --with-newlib --with-sysroot=&#47;opt&#47;riscv&#47;gcc&#47;riscv64-unknown-elf --disable-libmudflap --disable-libssp --disable-libquadmath --disable-libgomp --disable-nls --disable-tm-clone-registry --src=&#47;home&#47;adrain&#47;RISCV_TOOLS&#47;riscv-gnu-toolchain&#47;riscv-gcc --enable-multilib --with-abi=lp64d --with-arch=rv64imafdc --with-tune=rocket --with-isa-spec=2.2 &#39;CFLAGS_FOR_TARGET=-Os   -mcmodel=medlow&#39; &#39;CXXFLAGS_FOR_TARGET=-Os   -mcmodel=medlow&#39;
Thread model: single
Supported LTO compression algorithms: zlib
gcc version 10.2.0 (GCC)
debug的时候：
$ make
CC -[M] 正在构建... add.S
CC -[M] 正在构建... main.c
&#47;opt&#47;riscv&#47;gcc&#47;lib&#47;gcc&#47;riscv64-unknown-elf&#47;10.2.0&#47;..&#47;..&#47;..&#47;..&#47;riscv64-unknown-elf&#47;bin&#47;ld: cannot find crtbegin.o: No such file or directory
&#47;opt&#47;riscv&#47;gcc&#47;lib&#47;gcc&#47;riscv64-unknown-elf&#47;10.2.0&#47;..&#47;..&#47;..&#47;..&#47;riscv64-unknown-elf&#47;bin&#47;ld: cannot find -lgcc
collect2: error: ld returned 1 exit status
make: *** [Makefile:37: main.elf] Error 1
是为啥呢</p>2023-01-23</li><br/><li><span>释迦</span> 👍（0） 💬（0）<p>为什么我按f5后弹出的对画框中cpu的寄存器是x86-64的寄存器，没有显示risc-v寄存器？步骤也是按照老师文中的步骤操作的。</p>2022-12-21</li><br/>
</ul>