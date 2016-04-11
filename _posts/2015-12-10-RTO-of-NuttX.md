---
layout: post
title:  "初识NuttX操作系统之NSH"
categories: "drons_lifes"
author: 吴兴章
tags: 工作生活
donate: true
comments: true
update: 2016-04-12 01:59:43
---
这篇文章主要记录学习NuttX的过程以及对NuttX的理解，并结合apm里的px4-v2例程设置进行说明。

<br>
1.0 NuttX Operating System User's Manual
============

[NuttX Operating System User's Manual](http://nuttx.org/doku.php?id=documentation:userguide)从软件开发者的视角为NuttX提供一般的使用信息。

<br>
2.0 NSH 启动脚本------<i>翻译自[NuttX文档1.8节](http://nuttx.org/doku.php?id=documentation:nuttshell)，欢迎提出宝贵意见</i>
============

>Tip: 源码中apps/nshlib/README.txt即为说明书。

##NSH 启动脚本。
NSH支持选项来为NSH提供一个启动脚本。一般来说这种能力是用CONFIG_NSH_ROMFSETC启用，但有几个相关的描述NSH特定配置设置的配置选项。这种能力还取决于：

* CONFIG_DISABLE_MOUNTPOINT 未设置
* CONFIG_NFILE_DESCRIPTORS > 4
* CONFIG_FS_ROMFS 使能


例程演示：

```c
#define CONFIG_NSH_ROMFSETC 1
//CONFIG_DISABLE_MOUNTPOINT未定义
#define CONFIG_NFILE_DESCRIPTORS 50
#define CONFIG_FS_ROMFS 1
```

##默认启动行为。
所提供的实施旨在为启动文件的使用提供极大的灵活性。本段将讨论所有配置选项设置为默认值时的一般行为。

<!--more-->
在默认情况下，使能CONFIG_NSH_ROMFSETC将导致NSH在启动时表现如下：

* NSH将创建一个只读的内存盘（ROM盘），含有一个微小的romfs文件系统包含以下：

				`--init.d/
				     `-- rcS
rcS是NSH启动脚本。

* NSH将装入romfs文件系统到/etc，导致：

				|--dev/
				|   `-- ram0
				`--etc/
				    `--init.d/
				        `-- rcS

* 默认情况下，rcS脚本的内容：

				# Create a RAMDISK and mount it at XXXRDMOUNTPOINTXXX

				mkrd -m 1 -s 512 1024
				mkfatfs /dev/ram1
				mount -t vfat /dev/ram1 /tmp

* NSH将在/etc/init.d/rcS启动时执行脚本(在第一个NSH提示前)。脚本执行后，根文件将看起来像：

				|--dev/
				|   |-- ram0
				|   `-- ram1
				|--etc/
				|   `--init.d/
				|       `-- rcS
				`--tmp/

**修改ROMFS镜像。**/etc目录的内容保留在文件apps/nshlib/nsh_romfsimg.h里，或者如果定义了CONFIG_NSH_ARCHROMFS则会包含在arch/board/nsh_romfsimg.h里。

```c++
#ifdef CONFIG_NSH_ARCHROMFS
#  include <arch/board/nsh_romfsimg.h>
#else
#  include "nsh_romfsimg.h"
#endif
```
为了修改启动行为，有三件事情要学习：

1. **配置选项**。其他CONFIG_NSH_ROMFSETC配置选项将在后面讨论。

2. **tools/mkromfsing.sh脚本**。脚本tools/mkromfsing.sh创建nsh_romfsing.h。不自动执行。如果你想改变创建和安装/tmp目录相关的配置设置，则必须使用tools/mkromfsimg.sh脚本重新生成该头文件。    
这个脚本的行为依赖三点：
	+ 已经安装的NuttX配置是怎么设置的。
	+ genromfs工具，可从[http://romfs.sourceforge.net](http://romfs.sourceforge.net)下载。
	+ 文件apps/nshlib/rcS.template，或者如果定义了CONFIG_NSH_ARCHROMFS，则include/arch/board/rcs.template。

3. **rcS.template**。 apps/nshlib/rcS.template文件包含一般形式的rcS文件，配置的值插入到该模板文件来产生最终的rcS文件。

**注意**：apps/nshlib/rcs.template生成标准的默认nsh_romfsimg.h文件，如果CONFIG_NSH_ARCHROMFS在NuttX配置文件中定义，那么一个自定义、板级相关的驻留在configs/\<board>/include里的nsh_romfsimg.h文件将被使用。注意：当操作系统配置完成，include/arch/board将被链接到configs/\<board>/include。

所有的启动行为都包含在rcS.template。mkromfsimg.sh的作用是将特定的配置来设置rcS.template创建最终的rcS，以及生成包含ROMFS系统映像的头文件nsh_romfsimg.h。

<br>
3.0 定制NuttShell------<i>翻译自[NuttX文档4.0节](http://nuttx.org/doku.php?id=documentation:nuttshell)，欢迎提出宝贵意见</i>
============

`概要`。NuttShell (NSH)是一个简单的可用于NuttX的shell程序。它支持多种命令，是（非常）松散基于bash shell和用于unix shell编程的常用工具。本附录的段落将专注于定制NSH：添加新命令，改变初始化序列，等等。

##3.1 NSH 库和NSH 初始化

`概要`。NSH是一个库，可以在apps/nshlib发现实现。作为一个库，它可以定制成任何遵循以下描述的NSH初始化序列中的应用。作为一个例子，在 apps/examples/nsh/nsh_main.c在的代码说明如何启动NSH，逻辑也适用于您自己的自定义代码。虽然代码生成简单的作为一个例子，在大多数人只是用这个例子的代码作为应用main()功能。在下面的段落中讨论了这个例子的初始化。

**3.1.1 NSH初始化序列**

NSH启动顺序很简单。作为一个例子，apps/examples/nsh/nsh_main.c的代码说明如何启动NSH。简单工作如下：

1. 如果你有C++静态初始化器，它将调用up_cxxinitialize()的执行，这个函数将依次调用这些静态初始化器。在对STM3240G-EVAL板的情况下，对up_cxxinitialize()的实施可以在nuttx/configs/stm3240g-eval/src/up_cxxinitialize.c中发现。

	```c++
	  /* Call all C++ static constructors */

	#if defined(CONFIG_HAVE_CXX) && defined(CONFIG_HAVE_CXXINITIALIZE)
	  up_cxxinitialize();
	#endif
	```
2. 这个函数，然后调用nsh_initialize()，它将初始化NSH库，nsh_initialize()将具体描述如下节。
3. 如果Telnetconsole启用，它调用驻留在NSH库里的nsh_telnetstart()。nsh_telnetstart()将启动telnet守护进程来监听Telnet连接和启动远程NSH会话。

	```c++
	#ifdef CONFIG_NSH_TELNET
	  ret = nsh_telnetstart();
	  if (ret < 0)
	    {
	     /* The daemon is NOT running.  Report the the error then fail...
      	- either with the serial console up or just exiting.
	      */

	     fprintf(stderr, "ERROR: Failed to start TELNET daemon: %d\n", ret);
	     exitval = 1;
	   }
	#endif
	```
4. 如果一个本地控制台启用（可能在一个串行端口），然后nsh_consolemain()将被调用。nsh_consolemain()也位于NSH库。nsh_consolemain()不返回，完成整个NSH初始化序列。

	```c++
	  /* If the serial console front end is selected, then run it on this thread */

	#ifdef CONFIG_NSH_CONSOLE
	  ret = nsh_consolemain(0, NULL);

	  /* nsh_consolemain() should not return.  So if we get here, something
   	- is wrong.
	   */

	  fprintf(stderr, "ERROR: nsh_consolemain() returned: %d\n", ret);
	  exitval = 1;
	#endif
	```

**3.1.2 nsh_initialize()**

NSH初始化函数，nsh_initialize()，在apps/nshlib/nsh_init.c发现。

```c++
void nsh_initialize(void)
{
  /* Mount the /etc filesystem */

  (void)nsh_romfsetc();

  /* Perform architecture-specific initialization (if available) */

  (void)nsh_archinitialize();

  /* Bring up the network */

  (void)nsh_netinit();
}
```
它也只有三件事：

1.*nsh_romfsetc()*:如果是这样的配置，它执行一个NSH启动脚本，这个脚本可以在目标文件系统/etc/init.d/rcS中发现。nsh_romfsetc()函数可以在apps/nshlib/nsh_romfsetc.c中发现。这个函数将一个ROMFS文件系统登记，然后安装ROMFS文件系统。/etc是只读、ROMFS文件系统通过nsh_romfsetc()安装的默认位置。   

ROMFS镜像本身，也简称固件。默认情况下，该RCS启动脚本中包含以下逻辑：

	# Create a RAMDISK and mount it at XXXRDMOUNTPOINTXXX

	mkrd -m XXXMKRDMINORXXX -s XXMKRDSECTORSIZEXXX XXMKRDBLOCKSXXX
	mkfatfs /dev/ramXXXMKRDMINORXXX
	mount -t vfat /dev/ramXXXMKRDMINORXXX XXXRDMOUNTPOINTXXX

ROMFS镜像被创建时模板中的XXXX*XXXX字符得到替换：

* XXXMKRDMINORXXX 将成为RAM的次设备号。默认：0
* XXMKRDSECTORSIZEXXX 将成为内存设备扇区大小
* XXMKRDBLOCKSXXX 将成为该设备的扇区数
* XXXRDMOUNTPOINTXXX 将成为配置的安装点。默认值：/etc

默认情况下，替换的值将产生一个rcS文件如：

	# Create a RAMDISK and mount it at /tmp

	mkrd -m 1 -s 512 1024
	mkfatfs /dev/ram1
	mount -t vfat /dev/ram1 /tmp

然后：

* 在/dev/ram1创建一个大小为512 * 1024字节的RAMDISK，
* 在磁盘/dev/ram1格式化FAT文件系统，然后
* 挂载FAT文件系统在配置挂载点，/tmp。

rcS模板文件能在apps/nshlib/rcS.template找到。由此产生的ROMFS文件系统可以在apps/nshlib/nsh_romfsimg.h发现。

2.*nsh_archinitialize()*：未来的任何特定于体系结构的NSH初始化将被执行（如果有）。如STM3240G-EVAL，这种特定结构的初始化可以在configs/stm3240g-eval/src/up_nsh.c配置。这也像：（1）初始化SPI设备，（2）初始化SDIO，和（3）安装，可以插入的任何SD卡。

3.*nsh_netinit()*：nsh_netinit()函数可以在apps/nshlib/nsh_netinit.c中发现。

##3.2 NSH命令

`概要`。NSH支持多种命令，是NSH程序的一部分。所有的NSH命令都列在上面的NSH文档里。然而，并不是所有这些命令都可以在任何时候使用。许多命令取决于某些NuttX配置选项。你可以输入命令help在NSH提示后看到实际可用的命令：

		nsh> help

例如，如果网络不支持，那么所有的网络相关的命令将从'nsh> help'展现的命令列表中消失。

**3.2.1 添加新的NSH命令**

新的命令可以非常容易地增加到NSH。你只需增加2件事：

1. 实现您的命令，和
2. NSH命令表中一个新的条目

`实现您的命令`。例如，如果你想添加一个新的叫mycmd命令到NSH，你会先在函数原型实现mycmd代码：

		int cmd_mycmd(FAR struct nsh_vtbl_s *vtbl, int argc, char **argv);

argc和argv是用来传递命令行参数到NSH命令。命令行参数是在一个非常标准的方式传递：argv [ 0 ]将该命令的名称，argv[1] 到 argv[argc-1]在NSH命令行提供额外的参数。

第一个参数，vtbl，是特殊的。这是一个指向特定会话状态信息的指针。你不需要知道状态信息的内容，但是当你与NSH的交互逻辑时你需要传递vtbl参数。你只需要利用vtbl参数来输出数据到控制台。你不用在NSH命令使用printf()，反而你会使用：

		void nsh_output(FAR struct nsh_vtbl_s *vtbl, const char *fmt, …); 

所以，如果你只想在控制台上输出“Hello, World!”，然后你的整个命令的执行可能是：

		int cmd_mycmd(FAR struct nsh_vtbl_s *vtbl, int argc, char **argv)
		{
		  nsh_output(vtbl, "e;Hello, World!"e;);
		  return 0;
		}

对新命令的原型应该放在apps/examples/nshlib/nsh.h。

`加入你命令到NSH命令表`。所有出现在单个表中支持NSH的命令都调用：

		const struct cmdmap_s g_cmdmap[]

该表可以在文件apps/examples/nshlib/nsh_parse.c中找到，结构cmdmap_s也是定义在apps/nshlib/nsh_parse.c中：

		struct cmdmap_s
		{
		  const char *cmd;        /* Name of the command */
		  cmd_t       handler;    /* Function that handles the command */
		  uint8_t     minargs;    /* Minimum number of arguments (including command) */
		  uint8_t     maxargs;    /* Maximum number of arguments (including command) */
		  const char *usage;      /* Usage instructions for 'help' command */
		};

这个结构提供了你需要的一切来描述你的命令：它的名字（CMD），处理命令的函数（cmd_mycmd()），命令需要的最大最小参数个数，和一个描述命令行参数字符串，最后一个字符串是输入"nsh> help"打印出来的东西。

所以，对于你commnd样本，你可以添加下面到g_cmdmap [ ]表：

		{ "mycmd", cmd_mycmd, 1, 1, NULL },

这项特别简单因为mycmd是如此简单。在g_cmdmap [ ]看看其他的命令更为复杂的例子。

##3.3 NSH“内置”的应用

`概要`。除了属于NSH一部分的命令之外，外部程序也可以作为NSH执行的命令。由于历史的原因这些外部程序被称之为“内置”应用。这个术语有点混乱，因为如上所述的实际的NSH命令是真正内置到NSH的，而这些应用是外接到NuttX的。

它们可以通过在NSH提示简单地键入应用程序的名称执行，在这个意义上这些应用可以内置到NSH。内置的应用程序支持是启用这些配置选项：

+ CONFIG_BUILTIN：使能NuttX支持内置应用。
+ CONFIG_NSH_BUILTIN_APPS：使能NSH支持内置应用。

当设置完这些配置选项，输入"nsh> help"可以看到这些内置应用。它们将出现在NSH命令列表的底部：

			Builtin Apps:

请注意：这些内置应用程序名字的外面没有提供详细的帮助信息。

**3.3.1 内置应用**

`概要`。基本逻辑就是支持NSH内置的应用程序称为“内置程序”。内置的应用程序方法可以在apps/builtin发现。这些方法简单实现如下：

1. 它支持注册机制，内置的应用程序可以动态注册自己的构建时间，和
2. 查找，列表，并执行内置的应用实用函数。

`内置应用实用函数`。内置应用程序方法导出的实用函数原型在nuttx/include/nuttx/binfmt/builtin.h和apps/include/builtin.h。这些实用函数包括：

- int builtin_isavail(FAR const char *appname); 检查应用程序构建时间注册为应用程式的可用性。
- const char *builtin_getname(int index);通过索引返回一个指向内置应用程序名称的指针。这是NSH使用的实用函数，为了输入“nsh> help"时列出可用的内置应用程序。
- int exec_builtin(FAR const char *appname, FAR const char **argv); 在编译时间执行内置式应用程序注册。这是NSH使用在执行内置应用时的实用函数。

`自动生成的头文件`。当NuttX第一次建立时带有需求的应用程序入口点都聚集在两个文件：

1. apps/builtin/builtin_proto.h: 应用程序任务入口点的原型。
2. apps/builtin/builtin_list.h:应用程序特定信息和启动要求。

`内置应用程序的注册`。由于不同的构造目标被执行，NuttX构建发生在几个不同的阶段。（1）建立配置，（2）产生目标依赖性，（3）默认的（所有）当正常编译链接操作完成。内置应用程序信息在构建阶段收集。

在apps/examples/hello目录可以一个内置应用程序的例子。让我们一起通过这个具体的原因来说明内置的应用程序创建和如何他们自己注册的一般方式，以至于他们可以从NSH使用。

*apps/examples/hello. *apps/examples/hello主要例程可以在apps/examples/hello/main.c发现。主要的例程是：

		int hello_main(int argc, char *argv[])
		{
		  printf("Hello, World!!\n");
		  return 0;
		}

这是一个内置的函数，它将在NuttX构造的上下文构建阶段被注册。注册是由apps/examples/hello/Makefile方法实现。但是，构建系统通过一个相当曲折的路径到达这个方法：

1. 顶层的make目标在nuttx/Makefile。所有的构建目标取决于context构建目标。对于apps/目录，这个构建目标将执行apps/Makefile里的context目标。
2. apps/Makefile将轮流执行所有配置的子目录context目标，在我们的情况下，将包括apps/examples里的Makefile。
3. 最后，apps/examples/Makefile将执行所有配置了的example子文件夹的context目标，最后到apps/examples/Makefile，如下。

**注意：**由于此上下文构建阶段只能执行一次，任何随后您将作出的配置更改，然后，不反映在构建序列。这是一个常见的混乱地区。在你可以实例化新的配置，你首先要摆脱旧的配置。最激烈的方式是：

		make distclean

但你将不得不从头开始重新配置NuttX。但是，如果你只想在apps/sub-directory重新构建配置，那么很少的工作量就可以实现。以下nuttx命令将只从apps/目录删除配置，将让你无需重新配置一切来继续：

		make apps_distclean

在apps/examples/hello/Makefile里的上下文目标方法注册builtin's builtin_proto.hand builtin_list.h文件里的hello_main()应用。这样的方法在apps/examples/hello/Makefiles里，摘录如下：

1、首先，Makefile 包括apps/Make.defs:

		include $(APPDIR)/Make.defs

这定义一个称为REGISTER的宏，能添加数据到内置的头文件：

		define REGISTER
		    @echo "Register: $1"
		    @echo "{ \"$1\", $2, $3, $4 }," >> "$(APPDIR)/builtin/builtin_list.h"
		    @echo "EXTERN int $4(int argc, char *argv[]);" >> "$(APPDIR)/builtin/builtin_proto.h"
		endef

当这个宏运行时，你会看到”Register:hello“的输出，这是注册成功的肯定迹象。

2、make文件然后定义应用程序名字（hello），任务的优先级（默认），和将被分配到的任务运行的栈的大小（2K）。

		APPNAME         = hello
		PRIORITY        = SCHED_PRIORITY_DEFAULT
		STACKSIZE       = 2048

3、最后，Makefile调用寄存器宏添加hello_main()内置应用。然后，当系统建立完成，hello命令可以从NSH命令行执行。当hello命令执行时，它将用默认的优先级和一个2K的堆栈大小启动任务的入口点hello_main()。

		context:
		  $(call REGISTER,$(APPNAME),$(PRIORITY),$(STACKSIZE),$(APPNAME)_main)

`内置应用程序的其他用途`。内置应用程序的主要目的是支持应用程序从NSH命令行执行。然而，有内置的应用程序应该提到的另一个用途。

1. *binfs*。binfs是一个位于apps/builtin/binfs.c的很小的文件系统。这提供另外一种可视化安装内置应用。没有binfs，你能用NSH帮助命令看到已经安装的内置应用。binfs将创建一个小的伪文件系统安装在/bin。使用binfs，你可以通过列出/bin目录的内容来看到可用的内置应用程序。这给一些肤浅的Unix兼容，但是没有添加任何新的功能。

**3.3.2 同步构建的应用程序**

默认情况下，从NSH命令行开始启动的内置命令将与NSH异步运行。如果你想强迫NSH执行命令，然后等待命令执行，您可以通过添加下面到NuttX配置文件启用功能：

		CONFIG_SCHED_WAITPID=y

此配置选项可以为标准waitpid() RTOS接口支持，当接口启用，NSH将用它来等待，睡眠到内置的应用程序执行完成。

当然，即使CONFIG_SCHED_WAITPID=y定义，具体的应用程序仍然可以在NSH命令后添加符号（&）强制异步运行。

##3.4 定制NSH初始化

`定制NSH初始化的方法`。这里有三种方法来定制NSH启动行为。依照难易程度呈现如下：

1. 你可以在configs/stm3240g-eval/src/up_nsh.c扩展初始化方法。这里的方法在每次NSH启动时调用，它特别适合任何设备相关初始化。
2. 用任何你想启动的方法替换apps/examples/nsh/nsh_main.c里的示例代码。NSH是一个位于apps/nshlib的库，apps/example/nsh只是一个微小的启动示例函数（CONFIG_USER_ENTRYPOINT()），你能快速运行，并说明如果你想要别的东西立即运行该如何启动NSH，然后你可以写你自己的自定义CONFIG_USER_ENTRYPOINT()函数，从你的自定义CONFIG_USER_ENTRYPOINT()开始其他的任务。
3. NSH还支持NSH第一次运行时执行的启动脚本。这种机制的优点是启动脚本可以包含任何NSH命令，可以用很少的代码做很多工作。缺点是，创建启动脚本是相当复杂的。这是足够复杂的，值得有自己的段落。

**3.4.1 NuttShell 启动脚本**

`NSH启动脚本`。NSH支持选项来为NSH提供一个启动脚本。启动脚本包含任何支持NSH的命令（即，当你进入“NSH >帮助”看到的）。一般来说这种能力是用CONFIG_NSH_ROMFSETC启用，但有几个相关的描述NSH特定配置设置的配置选项。这种能力还取决于：

- CONFIG_DISABLE_MOUNTPOINT=n。如果安装点支持被禁用，则不能安装任何文件系统。
- CONFIG_NFILE_DESCRIPTORS > 4。当然，您必须在文件系统中使用任何文件有文件说明。
- CONFIG_FS_ROMFS enabled。此选项使能ROMFS文件系统支持。

`默认启动行为`。所提供的实施旨在为启动文件的使用提供极大的灵活性。本段将讨论所有配置选项设置为默认值时的一般行为。

在默认情况下，使能CONFIG_NSH_ROMFSETC将使NSH在NSH启动时表现如下：

- NSH将创建一个只读的内存盘（ROM盘），含有一个微小的ROMFS文件系统包含以下：

						`--init.d/
						    `-- rcS

rcS是NSH启动脚本。
- NSH将装入ROMFS文件系统到/etc，导致：

						|--dev/
						|   `-- ram0
						`--etc/
						    `--init.d/
						        `-- rcS

- 默认rcS脚本的内容是：

				# Create a RAMDISK and mount it at /tmp

				mkrd -m 1 -s 512 1024
				mkfatfs /dev/ram1
				mount -t vfat /dev/ram1 /tmp

- NSH将在/etc/init.d/rcS启动时执行脚本(在第一个NSH提示前)。脚本执行后，根文件将看起来像：

						|--dev/
						|   |-- ram0
						|   `-- ram1
						|--etc/
						|   `--init.d/
						|       `-- rcS
						`--tmp/

`实例配置`。这里有一些配置，在这些配置文件里CONFIG_NSH_ROMFSETC=y。他们可以提供有用的例子：

				configs/hymini-stm32v/nsh2
				configs/ntosd-dm320/nsh
				configs/sim/nsh
				configs/sim/nsh2
				configs/sim/nx
				configs/sim/nx11
				configs/sim/touchscreen
				configs/vsn/nsh

在大多数情况下，配置设置默认的/etc/init.d/rcS脚本。默认的脚本是在这里：apps/nshlib/rcS.template（在模板里像XXXMKRDMINORXXX样的值在构建时通过sed替换）。默认的配置创建一个虚拟磁盘，安装在/tmp如上。

如果默认的行为是不是你想要的，那么你可以通过定义CONFIG_NSH_ARCHROMFS=y 在配置文件中提供您自己的自定义RCS脚本。在NuttX代码树中使用自定义/etc/init.d/rcS文件唯一的例子是这个：configs/vsn/nsh。configs/vsn/nsh/defconfig文件也有这样的定义：

		CONFIG_NSH_ARCHROMFS=y -- 支持一个特定结构的ROMFS文件。

`修改ROMFS镜像`/etc目录的内容包含在文件apps/nshlib/nsh_romfsimg.h里，或如果CONFIG_NSH_ARCHROMFS 被定义，包含/arch/board/nsh_romfsimg.h。

为了修改启动行为，有三件事情要学习：

1、配置选项。

2、tools/mkromfsimg.sh脚本。tools/mkromfsimg.sh脚本创建nsh_romfsimg.h。它不自动执行。如果你想改变创建和安装/tmp目录相关的配置设置，则必须使用tools/ mkromfsimg.sh脚本重新生成该头文件。 

这个脚本的行为取决于几件事：

1. 配置设置然后安装配置。
2. genromfs工具（可以从[http://romfs.sourceforge.net](http://romfs.sourceforge.net)下载），在NuttX工具库[这里](https://bitbucket.org/nuttx/tools/src/master/genromfs-0.5.2.tar.gz)还有一个可用快照。
3. 用来产生C 头文件的xxd工具（xxd是一个完整的Linux或Cygwin安装正常的一部分，通常是vi包的一部分）。
4. 文件 apps/nshlib/rcS.template（或如果CONFIG_NSH_ARCHROMFS 定义，包括/arch/board/rcs.template）。

3、rcS.template。文件apps/nshlib/rcS.template包含rcS文件的一般形式；配置的值插入到该模板文件来产生最终的RCS文件。

为了生成一个自定义的rcS文件，rcS.template的副本需要放到toos/，根据需要的启动行为改变。运行tools/mkromfsimg.sh创建nsh_romfsimg.h，这个头文件需要复制到apps/nhslib或者如果configs/\<board\>/include中CONFIG_NSH_ARCHROMFS定义。

`rcS.template`。默认的rcS.template, apps/nshlib/rcS.template生成标准的、默认的apps/nshlib/nsh_romfsimg.h文件。

如果CONFIG_NSH_ARCHROMFS在NuttX配置文件中定义，然后驻留在configs/\<board>/include中的自定义、板级相关的nsh_romfsimg.h将被使用。注意，当操作系统配置完成， include/arch/board将被链接到configs/\<board\>/include。

如上面提到的，在NuttX代码树中使用自定义/etc/init.d/rcS文件的唯一的例子是：configs/vsn/nsh。configs/vsn的自定义脚本是位于configs/vsn/include/rcS.template。

所有的自定义行为都包含在rcS.template。mkromfsimg.sh脚本的作用是（1）将特定的配置设置rcS.template来创建最终的RCS，和（2）生成包含romfs文件系统映像的头文件nsh_romfsimg.h。要做到这一点，mkromfsimg.sh使用必须安装在您的系统的两个工具：

1. genromfs工具，用来生成ROMFS文件系统映像。
2. xxd 用来创建C 头文件。

你能在configs/vsn/include/rcS.template找到为configs/vsn生成的ROMFS文件系统。