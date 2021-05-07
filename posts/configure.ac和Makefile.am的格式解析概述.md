
<!-- TOC -->

- [1. Autotools](#1-autotools)
    - [1.1. Autotools相关工具链](#11-autotools相关工具链)
        - [1.1.1. Autotools](#111-autotools)
        - [1.1.2. 其他相关工具](#112-其他相关工具)
    - [1.2. 工具链的流程](#12-工具链的流程)
    - [1.3. autoconf](#13-autoconf)
        - [1.3.1. configure.ac文件](#131-configureac文件)
        - [1.3.2. configure.ac文件的标准布局](#132-configureac文件的标准布局)
        - [1.3.3. configure.ac常见宏说明](#133-configureac常见宏说明)
        - [1.3.5. 常用变量](#135-常用变量)
        - [1.3.4. 关于自定义宏](#134-关于自定义宏)
    - [1.4. automake](#14-automake)
        - [1.4.1. Makefile.am文件](#141-makefileam文件)
        - [1.4.2. 语法说明](#142-语法说明)
            - [1.4.2.1. 统一命名规范](#1421-统一命名规范)
            - [1.4.2.2. 元](#1422-元)
            - [1.4.2.3. 目录](#1423-目录)
            - [1.4.2.4. 前缀](#1424-前缀)
            - [1.4.2.5. 衍生变量](#1425-衍生变量)
            - [1.4.2.6. 用户保留变量](#1426-用户保留变量)
            - [1.4.2.7. 目标变量](#1427-目标变量)
            - [1.4.2.8. 其他](#1428-其他)
        - [1.4.3. Makefile.am常用变量说明](#143-makefileam常用变量说明)
    - [1.5. 一些常见问题](#15-一些常见问题)
        - [1.5.1. 标识变量的顺序问题](#151-标识变量的顺序问题)
        - [1.5.2. 如何引用预定义的变量](#152-如何引用预定义的变量)
    - [1.6. clist示例工程](#16-clist示例工程)
    - [1.7. 参考资料](#17-参考资料)

<!-- /TOC -->

# 1. Autotools

## 1.1. Autotools相关工具链
### 1.1.1. Autotools
Autotools是一系列工具
- **autoscan**  
可以通过调用autoscan命令，得到一个初始化的configure.scan文件。然后重命名为configure.ac后，在此基础上编辑configure.ac。  
autoscan会扫描源码，并生成一些通用的宏调用，输入的声明，以及输出的声明。尽管autoscan十分方便，但是没人能够在构建之前，就把源码完全写好。  
因此,autoscan通常用于初始化configure.ac，即生成configure.ac的雏形文件configure.scan
- **aclocal**  
configure.ac实际是依靠宏展开来得到configure。因此，能否成功生成，取决于宏定义是否能够找到。  
autoconf会从自身安装路径下寻找事先定义好的宏。然而对于像automake，libtool，gettex等第三方扩展宏，autoconf便无从知晓。  
因此，aclocal将在configure.ac同一个目录下生成aclocal.m4，在扫描configure.ac过程中，将第三方扩展和开发者自己编写的宏定义复制进去。  
如此一来，autoconf遇到不认识的宏时，就会从aclocal.m4中查找
- **autoconf**  
autoconf是 用来产生configure文件的 .configure是 一个脚本,它能设置
源程序来适应各种不同的操作系统平台,并且根据不同的 系统来产生合适的 Makefile,从而可以使
你的源代码能在不同的操作系统平台上被编译出来.
- **autoheader**  
autoheader命令扫描configure.ac文件，并确定如何生成config.h.in。每当configure.ac变化时，都可以通过执行autoheader更新config.h.in。  
在configure.ac通过AC_CONFIG_HEADERS([config.h])告诉autoheader应当生成config.h.in的路径，
config.h包含了大量的宏定义，其中包括软件包的名字等信息，程序可以直接使用这些宏。  
更重要的是，程序可以根据其中的对目标平台的可移植相关的宏，通过条件编译，动态的调整编译行为。
- **automake**  
手工编写Makefile是一件相当繁琐的事情，并且随着项目的复杂程序变大，编写难度越来越大。automake工具应运而生。  
可以编辑Makefile.am文件，并依靠automake来生成Makefile.in

### 1.1.2. 其他相关工具
- **m4**  
m4是一个经典的宏工具。autoconf正是构建在m4之上，可以理解为autoconf预先定义了大量的，用户检查系统可移植性的宏，这些宏在展开就是大量的shell脚本。  
所以编写configure.ac就需要对这些宏掌握熟练，并且合理调用。
- **libtool**  
libtool试图解决不同平台下，库文件的差异。libtool实际是一个shell脚本，实际工作中，调用了目标平台的cc编译器和链接器，以及给予合适的命令行参数。  
libtool可以单独使用，也可以跟autotools集成使用。
- **autoreconf**  
早期autoreconf并不存在，软件开发者就自己编写脚本，按照顺序调用autoconf，autoheader，automake等工具.  
autoreconf程序能够自动按照合理的顺序调用autoconf，automake，aclocal程序


## 1.2. 工具链的流程
1. autotools完整流程  
![autotools流程](https://img2020.cnblogs.com/blog/1703382/202010/1703382-20201012164146315-807223944.jpg)
2. autoreconf  
![autotools生成configure](https://img2020.cnblogs.com/blog/1703382/202010/1703382-20201012164154785-893224028.png)
2. behind autoreconf  
![autoreconf](https://img2020.cnblogs.com/blog/1703382/202010/1703382-20201012164916086-1197131877.png)
4. configure生成Makefile  
![configure生成Makefile](https://img2020.cnblogs.com/blog/1703382/202010/1703382-20201012164156943-1641450036.png)

```none
your source files --> [autoscan*] --> [configure.scan] --> configure.ac
```
```none
[acinclude.m4] --.
                 |
[local macros] --+--> aclocal* --> aclocal.m4
                 |
configure.ac ----'
```
```none
configure.ac --.
               |   .------> autoconf* -----> configure
[aclocal.m4] --+---+
               |   `-----> [autoheader*] --> [config.h.in]
[acsite.m4] ---'
```
```none
configure.ac --.
               +--> automake* --> Makefile.in
Makefile.am ---'
```
```none
                         .-------------> [config.cache]
configure* --------------+------------> config.log
                         |
[config.h.in] -.         v          .-> [config.h] -.
               +--> config.status* -+               +--> make*
Makefile.in ---'                    `-> Makefile ---'
```

所以需要项目维护者手动修改的文件除了可选的NEWS README AUTHORS ChangeLog文件之外，就是configure.ac文件和每个源码文件夹下的Makefile.am文件了。

## 1.3. autoconf

### 1.3.1. configure.ac文件
configure.ac用于生成configure脚本，以下以clist工程举例。

例如有以下文件结构  
```
.
├── clist                   // clist库
│   ├── clist.c
│   ├── clist.h
│   ├── CMakeLists.txt      // 该文件是用于cmake的，忽略
│   ├── config.h.in         // 该文件是用于cmake的，忽略
│   └── Makefile.am
├── CMakeLists.txt          // 该文件是用于cmake的，忽略
├── configure.ac 
├── .gitignore              // 该文件是用于git的，忽略
├── LICENSE
├── m4
│   └── .gitignore          // 该文件是用于git的，忽略
├── Makefile.am
├── README.md
└── samples                 // 测试程序，需要链接clist库
    ├── CMakeLists.txt      // 该文件是用于cmake的，忽略
    ├── Makefile.am
    ├── memcheck.stp        // 该文件是用于systemstap的，忽略
    └── test.c
```
其中需要在autotools构建中用到的文件除了源码文件之外，还有`./configure.ac`、`./Makefile.am`、`./clist/Makefile.am`、`./samples/Makefile.am`、`./m4/`。  
m4为空文件夹，是为了后续存放自动生成的m4宏文件避免autoconf告警，由于git无法提交空文件夹，故在其下add了一个.gitignore文件。

使用autoscan扫描目录后，修改自动生成的configure.scan为configure.ac，并修改如下
```makefile
#                                               -*- Autoconf -*-
# Process this file with autoconf to produce a configure script.

AC_PREREQ([2.69])

AC_INIT([clist], [1.0.1.1114], [tangm421@outlook.com])
AC_CONFIG_SRCDIR([clist/clist.c])
AC_CONFIG_HEADER(config.h)
AC_CONFIG_AUX_DIR([build-aux])
AC_CONFIG_MACRO_DIR([m4])

#m4_pattern_forbid([^PKG_])

AM_INIT_AUTOMAKE
AM_PROG_AR([AC_MSG_ERROR([cannot find any available compression tool for \$AR])])

AC_DISABLE_SHARED
LT_INIT()
# Checks for programs.
AC_PROG_CC
#AC_DISABLE_SHARED  // 这类宏放在LT_INIT之后是无效的

#AC_PROG_INSTALL    // 若使用这两个老式宏，AC_PROG_CC最好放在此宏之前，否则会有N多警告
#AC_PROG_LIBTOOL
dnl AC_PROG_RANLIB  // 若使用了libtool则需要将AC_PROG_RANLIB去掉，且#注释无效

AC_CANONICAL_HOST

#AC_PROG_MAKE_SET

# Checks for libraries.

# Checks for header files.
#AC_CHECK_HEADERS([malloc.h stdio.h stdlib.h string.h])

# Checks for typedefs, structures, and compiler characteristics.
#AC_CHECK_HEADER_STDBOOL

AC_CHECK_HEADERS([db_cxx.h],
 [echo "db_cxx.h found"],
 [AC_MSG_WARN([db_cxx.h not found])],
 [#ifdef HAVE_FOO_H
 # include <inttypes.h>
 #endif
 ])

AC_MSG_NOTICE([Using native OSTYPE value from \$host_os: $host_os, ARCH/LARCH value from \$host_cpu: $host_cpu])

#AC_DEFINE(PROJECT_IMPORTEXPORT, [], [Only usefull for windows])
#AH_TEMPLATE(PROJECT_IMPORTEXPORT, [Only usefull for windows])
AC_DEFINE_UNQUOTED(PROJECT_IMPORTEXPORT, ${PROJECT_IMPORTEXPORT:""}, [Only usefull for windows])

AC_ARG_ENABLE(clist-debug,
[AS_HELP_STRING([--enable-clist-debug], [Enable clist debug print])],
 AC_SUBST(LIBCLIST_CPPFLAGS, "-DCLIST_DEBUG"),
 AC_SUBST(LIBCLIST_CPPFLAGS, ""))

# Checks for library functions.
#AC_FUNC_MALLOC

AC_CONFIG_FILES([Makefile clist/Makefile samples/Makefile])
AC_OUTPUT

#AC_OUTPUT([Makefile clist/Makefile samples/Makefile])
```

### 1.3.2. configure.ac文件的标准布局
configure.ac调用Autoconf宏的顺序并不重要，但有一些例外。每个configure.ac必须在检查之前包含对AC_INIT的调用，并在末尾包含对AC_OUTPUT的调用（参阅[Output](https://tool.oschina.net/uploads/apidocs/autoconf/Output.html#Output)）。此外，某些宏依赖于首先被调用的其他宏，因为它们会检查某些变量的先前设置值来决定要做什么。这些宏在各个描述中都有说明（参阅[Existing Tests](https://tool.oschina.net/uploads/apidocs/autoconf/Existing-Tests.html#Existing-Tests)），并且如果调用它们的顺序不正确，它们还会在创建配置时向您发出警告。

为了鼓励一致性，以下是建议的调用Autoconf宏的顺序
```none
Autoconf requirements
AC_INIT(package, version, bug-report-address)
information on the package
checks for programs
checks for libraries
checks for header files
checks for types
checks for structures
checks for compiler characteristics
checks for library functions
checks for system services
AC_CONFIG_FILES([file...])
AC_OUTPUT
```


### 1.3.3. configure.ac常见宏说明
| 宏                  | 说明                                                         |
| ------------------- | ------------------------------------------------------------ |
| AC_PREREQ           | 声明autoconf要求的版本号。                                   |
| AC_INIT             | 定义软件包全名称，版本号，联系方式。                         |
| AC_CONFIG_SCRDIR    | 用来侦测所指定的源码文件是否存在，来确定源码有效性。         |
| AC_CONFIG_HEADER    | AC_CONFIG_HEADERS([config.h])告诉autoheader应当生成config.h.in的路径，由autoconf自动生成config.h文件。在实际的编译阶段，生成的编译命令会加上-DHAVE_CONFIG_H定义宏，于是在代码中，就可以安全的引用config.h。<br>config.h包含了大量的宏定义，其中包括软件包的名字等信息，程序可以直接使用这些宏;更重要的是，程序可以根据其中的对目标平台的可移植性相关的宏，通过条件编译，动态的调整编译行为。<br>每当configure.ac有所变化，都可以通过再次执行autoheader更新config.h.in。在configure.ac通过 |
| AC_CONFIG_AUX_DIR   | 当我们以--install参数运行时，libtoolize --copy被调用，这将使得ltmain.sh被copy进来;接下来分别执行autoconf和autoheader;automake的参数为--add-missing --copy --no-force，这将使得几个辅助脚本和文件被安装到目录下。<br>这些辅助文件默认安装在configure.ac同一个目录下，如果你希望用另一个目录来存放他们，可以配置AC_CONFIG_AUX_DIR，例如AC_CONFIG_AUX_DIR([build-aux])将使用build-aux目录来存放辅助文件。<br>如果不使用--install参数，辅助文件要么不copy，要么以软链的形式创建。推荐使用--install，因为这样，其他软件维护可以避免由于构建工具版本不一致造成问题。 |
| AC_CONFIG_MACRO_DIR | AC_CONFIG_MACRO_DIR([m4])指定使用m4目录存放第三方宏;然后在最外层的Makefile.am中加入ACLOCAL_AMFLAGS = -I m4。 |
| AM_INIT_AUTOMAKE    | automake的出现晚于autoconf，所以automake是作为autoconf的扩展来实现的。通过在configure.ac中声明AM_INIT_AUTOMAKE告诉autoconf需要配置和调用automake。<br>在AC_INIT 宏之后添加AM_INIT_AUTOMAKE([foreign -Wall -Werror])，括号里面的选项可以根据需要来修改，具体请看[automake手册][automake]关于这个宏的说明。<br>NEWS README AUTHORS ChangeLog：这些文件是GNU软件的标配，不过在项目中不一定需要加入。如果项目中没有这些文件，每次autoreconf会提示缺少文件，不过这并不影响。如果不想看到这些错误提示，可以用AM_INIT_AUTOMAKE([foreign])来配置automake，或者在顶层Makefile.am中使用AUTOMAKE_OPTIONS = foreign；foreign参数就是告诉automake不要这么较真。 |
| AM_PROG_AR          | 指定压缩工具，构建静态库时需要。                             |
| AC_MSG_ERROR        | AC_MSG_XXX这些宏都是echo shell命令的包装器。configure时它们将输出定向到适当的文件描述符。配置脚本很少需要直接运行echo为用户打印消息。使用这些宏可以很容易地更改打印每种消息的方式和时间。 |
| AC_DISABLE_SHARED   | 更改LT_INIT的默认行为以禁用共享库。用户仍然可以通过指定“ --enable-shared”来覆盖此默认设置。 LT_INIT的"disable-shared"选项是该功能的简写。 AM_DISABLE_SHARED是AC_DISABLE_SHARED的已弃用别名；此选项必须在LT_INIT之前才能生效。 |
| LT_INIT             | 如果要使用libtool编译，需要在configure.ac中添加LT_INIT宏，同时去掉AC_PROG_RANLIB。<br>启用libtool后，该宏会添加对--enable-shared，--disable-shared，--enable-static，--disable-static，--with-pic和--without-pic配置标志的支持，查阅[LT_INIT说明](https://www.gnu.org/software/libtool/manual/html_node/LT_005fINIT.html)。 |
| AC_PROG_CC          | 指定C编译器，默认GCC。                                       |
| AC_PROG_LIBTOOL     | AC_PROG_LIBTOOL和AM_PROG_LIBTOOL是不推荐使用的旧版本，建议使用LT_INIT替代之。 |
| AC_PROG_RANLIB      | 构建静态库时需要，具体请看[automake手册][automake]，建议使用LT_INIT替代之。 |
| AC_CANONICAL_HOST   | 该宏调用后，可以在通过host_cpu，host_vendor和host_os这三个变量获得系统相关信息。 |
| AC_CHECK_HEADERS    | 检查一批头文件。                                             |
| AC_DEFINE           | 使用本宏进行符号定义，但要为其定义模板。如果缺少模板，autoheader将报错。 |
| AH_TEMPLATE         | 配合AC_DEFINE使用。                                          |
| AC_DEFINE_UNQUOTED  | 类似于AC_DEFINE，但还要对variable和value进行三种shell替换（每种替换只进行一次）： 变量扩展（'$'），命令替换（'\`'），以及反斜线转义符（'\\'）。值中的单引号和双引号 没有特殊的意义。在variable或者value是一个shell变量的时候用本宏代替AC_DEFINE。 |
| AM_CONDITIONAL      | 用于定义条件，生成一个automake宏，可以在Makefile.am中使用这个条件宏进行判断控制。 |
| AC_ARG_ENABLE       | 本宏用来增加编译时选项，该选项使用户可以选择要构建和安装的可选功能，若选项为软件包，类似于nginx中的引入第三方功能包时，参考AC_ARG_WITH。最终可以在./configure --help的Optional Packages选项中看到该项。 |
| AS_HELP_STRING      | 格式化帮助字符串。                                           |
| AC_SUBST            | 创建或者重新赋值一个automake变量。                           |
| AC_CONFIG_FILES     | 生成相应的Makefile文件，不同目录下通过空格分隔。             |
| AC_OUTPUT           | 建议使用AC_CONFIG_FILES替代之。                              |

### 1.3.5. 常用变量
> [autoconf 4.8.1 Preset Output Variables](https://www.gnu.org/savannah-checkouts/gnu/autoconf/manual/autoconf-2.69/autoconf.html#Preset-Output-Variables)

Autoconf会预设一些输出变量，例如：
| 变量           | 说明                                                         |
| :------------- | :----------------------------------------------------------- |
| `CFLAGS`       | 调用AC_PROG_CC时设置默认值`-g -O2`                           |
| `top_srcdir`   | 程序包的顶级源代码目录的名称。在顶级目录中，与`srcdir`相同。 |
| `srcdir`       | 包含该makefile的源代码的目录的名称。                         |
| `abs_srcdir`   | `srcdir`的绝对路径名称                                       |
| `top_builddir` | 当前构建树顶层目录的相对名称。在顶级目录中，与`builddir`相同。 |
| `buildidr`     | 当前构建树的当前目录，即`./`                                 |
| `abs_builddir` | `builddir`的绝对路径名称                                     |

### 1.3.4. 关于自定义宏
*后续待补。。。*


## 1.4. automake
### 1.4.1. Makefile.am文件
Makefile.am是一种比Makefile更高层次的规则。只需指定要生成什么目标，它由什么源文件生成，要安装到什么目录等构成。

- 原则1：每个目录一个Makefile.am文件；同时在configure.ac的AC_CONFIG_FILES宏中指定输出所有的Makefile文件  
- 原则2：父目录需要包含子目录，在父目录下的Makefile.am中添加: SUBDIRS = 子目录
- 原则3：Makefile.am中指明当前目录如何编译

Makefile.am用以生成最终的Makefile编译脚本，以下还是以clist工程为例

**./Makefile.am**
```makefile
AUTOMAKE_OPTIONS = foreign
ACLOCAL_AMFLAGS = -I m4

SUBDIRS = clist
SUBDIRS += samples

cmakedatadir = $(datadir)/cmake
nobase_dist_cmakedata_DATA = ./CMakeLists.txt
nobase_dist_cmakedata_DATA += clist/CMakeLists.txt
nobase_dist_cmakedata_DATA += ./samples/CMakeLists.txt
```
这里我把foreign选项放在了最顶层的Makefile.am中，由于configure.ac文件中加入了AC_CONFIG_MACRO_DIR宏，故此处必须加上`ACLOCAL_AMFLAGS = -I m4`。  
SUBDIRS是一个特殊变量，列出了在处理当前目录之前应递归到的所有目录。所有在此声明的子目录也参与到构建工程，故也需要Makefile.am文件。  
关于此文件中的最后几行则是在执行`make install`时将几个cmake文件按照原始文件结构层次安装到`$(datadir)/cmake`目录下，并且允许在分发（执行`make dist`）时将这几个cmake文件按照层次结构也包含在内。

**./clist/Makefile.am**
```makefile
lib_LTLIBRARIES = libclist.la

libclist_la_CPPFLAGS = @LIBCLIST_CPPFLAGS@

libclist_la_SOURCES = config.h clist.c

include_HEADERS = clist.h
```
子目录clist下用来编译clist库，因为启用了libtool（libtool将共享库和静态库抽象为一个统一的概念，称为libtool库。 libtool库是使用.la后缀的文件，并且可以指定静态库，共享库或两者。在运行./configure之前，无法确定它们的确切性质：并非所有平台都支持所有类型的库，并且用户可以显式选择应构建的库。由于共享库和静态库的目标文件必须以不同的方式进行编译，因此在编译过程中也会使用libtool。 libtool构建的对象文件称为libtool对象：这些文件使用.lo后缀。 Libtool库是从这些libtool对象构建的）所以此处库名为libclist.la，使用libclist_la_为前缀添加预编译指令、源码文件及包含文件。LIBCLIST_CPPFLAGS则是先前的configure.ac文件中声明的一个automake变量在此处引用。

**./samples/Makefile.am**
```makefile
#AM_CFLAGS = -I$(top_srcdir)/clist

bin_PROGRAMS = sample

sample_CFLAGS = -I$(top_srcdir)/clist
sample_SOURCES = test.c
sample_LDADD = ../clist/libclist.la

sample_CPPFLAGS = -D_BINDIR='"$(bindir)"'

nodist_sample_SOURCES = mydefines.h
BUILT_SOURCES = mydefines.h
CLEANFILES = mydefines.h
mydefines.h: Makefile
	echo '#define INSTALL_BINDIR "$(bindir)"' > $@
```
子目录samples用来编译测试程序，程序名为sample，使用sample_为前缀添加C编译标志、源码以及链接库。  
关于此文件中的最后几行，是为了说明后面[如何引用预定义的变量](#152-如何引用预定义的变量)的问题，`BUILT_SOURCES`则是为了在构建sample之前生成mydefines.h，关于`BUILT_SOURCES`变量的说明参阅[automake 9.4 Built Sources](https://www.gnu.org/software/automake/manual/automake.html#Sources)

**在静态库和动态库同时存在时，libtool默认链接的动态库**

*有关使用指定库链接以及静态库和动态库混合链接的情形后续待补...*


### 1.4.2. 语法说明
#### 1.4.2.1. 统一命名规范
> [automake 3.3 The Uniform Naming Scheme](https://www.gnu.org/software/automake/manual/automake.html#Uniform)  

Automake变量通常遵循统一的命名规范，可以轻松决定程序（和其他派生对象）的构建方式以及安装方式。该规范还支持`configure`时确定应构建的内容。

在`make`时，某些变量用于确定要构建的对象。变量名由几部分组成，这些部分串联在一起，其中指示构建的部分通常称为`元`。

#### 1.4.2.2. 元
就拿`_PROGRAMS`为例，以_PROGRAMS结尾的变量，列出了生成的Makefile应该生成的程序。用Automake语言来说，此_PROGRAMS后缀称为`元`。 

bin_PROGRAMS的`bin`部分告诉automake应该将生成的程序安装在bindir中。回想一下，GNU Build System使用一组变量来表示目标目录，并允许用户自定义这些位置。任何这样的目录变量都可以放在`元`前面（**省略dir后缀**），以告诉automake在何处安装列出的文件。

Automake可以识别与不同类型的文件相对应的其他元变量，例如_PROGRAMS，_LIBRARIES，_LTLIBRARIES等。

| 元变量         | 说明                                                  |
| :------------- | :---------------------------------------------------- |
| `_PROGRAMS`    | 标识需要被编译和连接的程序的列表                      |
| `_LIBRARIES`   | 标识需要被编译和连接的库的列表                        |
| `_LTLIBRARIES` | 标识需要被编译和连接的库（使用libtool生成的库）的列表 |

当前`元`有`_PROGRAMS`, `_LIBRARIES`, `_LTLIBRARIES`, `_LISP`, `_PYTHON`, `_JAVA`, `_SCRIPTS`, `_DATA`, `_HEADERS`, `_MANS`, `_TEXINFOS`。

**将没有前缀的`元`定义为变量（例如 `PROGRAMS`）是错误的！**

#### 1.4.2.3. 目录
> [automake 3.3 The Uniform Naming Scheme](https://www.gnu.org/software/automake/manual/automake.html#Uniform)

某些变量用于确定应该把创建了的对象安装在哪里。这些变量中在`元`前面的部分指示了应将哪个标准目录作为安装目录。标准目录名在GNU标准中给出（参考[GUN编码标准中的目录变量](http://www.gnu.org/prep/standards/standards.html#Directory-Variables)）。automake通过前缀`pkg`扩展了这些标准目录变量，这些扩展的与标准的相同，但附加了`$(PACKAGE)`。例如：
| 变量名          | 说明                             |
| :-------------- | :------------------------------- |
| `pkgdatadir`    | 相当于`$(datadir)/$(PACKAGE)`    |
| `pkgincludedir` | 相当于`$(includedir)/$(PACKAGE)` |
| `pkglibdir`     | 相当于`$(libdir)/$(PACKAGE)`     |

GNU编码标准中的目录变量，如：`prefix`、`exec_prefix`、`bindir`、`libdir`、`includedir`、`datarootdir`、`datadir`等。

**在构造变量名称时，通用的`dir`后缀被保留了**。因此，使用`bin_PROGRAMS`而不是`bindir_PROGRAMS`。在使用automake扩展目录时，同样适用于此规则。例如，以下代码片段会将./CMakeLists.txt安装到`$(datadir)/cmake`文件夹中。
```
cmakedatadir = $(datadir)/cmake
nobase_dist_cmakedata_DATA = ./CMakeLists.txt
```

#### 1.4.2.4. 前缀
> [automake 3.3 The Uniform Naming Scheme](https://www.gnu.org/software/automake/manual/automake.html#Uniform)

某些`元`还可由一个或者多个前缀和变量串联起来形成。某些前缀如下：

- `EXTRA_`

    对于每个`元`，都可以插入一个`EXTRA_`前缀。该前缀用于储存根据configure的运行结果，可能创建、也可能不创建的对象列表。引入该变量是因为Automake必须静态地知道需要创建的对象的完整列表以创建在所有情况下都能够工作的`Makefile.in`。  
    例如：`EXTRA_LTLIBRARIES = lib1.la lib2.la`，这两个库不会被明确的构建，如果其在任何地方都不会作为Makefile依赖项出现，就不会被构建。一般该前缀用在条件编译的情形下比较常见。

- `check_`

    该前缀表示仅仅在运行`make check`命令的时候才创建这些对象。

- `inst_` / `noinst_`

    用于控制`make install`时的安装行为。  
    `inst_`表示应构建相关对象，且进行安装。  
    `noinst_`表示应构建相关对象，但不安装。这通常用于构建程序包其余部分所需的对象，例如静态库或帮助程序脚本。

- `dist_` / `nodist_`

    用于控制`make dist`时的发布行为。  
    任何`元`或`_SOURCES`变量都可以使用dist_作为前缀，以将列出的文件添加到发行版中。类似地，nodist_可用于从发行版中省略文件。
    | 示例                                | 说明                                                       |
    | :---------------------------------- | :--------------------------------------------------------- |
    | `dist_maude_SOURCE = test.c`        | 表示test.c作为构建目标`maude`时候的源，并进行分发。        |
    | `nodist_maude_SOURCE = mydefines.h` | 表示mydefines.h只作为构建目标`maude`时候的源，但不进行分发 |

-  `nobase_`

    默认情况下，在子目录中指定的可安装文件在安装前将删除其目录名称。例如，在此示例中，头文件将安装为$(cmakedatadir)/CMakeLists.txt。
    ```
    dist_cmakedata_DATA += clist/CMakeLists.txt
    ```
    但是，可以使用`nobase_`前缀来规避此路径剥离。在此示例中，头文件将安装为$(cmakedatadir)/clist/CMakeLists.txt。
    ```
    nobase_dist_cmakedata_DATA += clist/CMakeLists.txt
    ```
    **当与其他前缀混合出现时，应首先指定该前缀**

#### 1.4.2.5. 衍生变量
> [automake 3.5 How derived variables are named](https://www.gnu.org/software/automake/manual/automake.html#Canonicalization)

有时Makefile变量名是从用户提供的某些文本中派生而来的。例如，`_PROGRAMS`中列出的程序名称将被重写为`_SOURCES`变量的名称。Automake把这些文本规范化，以使它可以不必服从Makefile的变量名规则。进行变量引用时，名称中的所有字符（字母，数字，`@`和下划线除外）都将变为下划线。

例如，如果程序名为`sniff-glue`，则派生变量名称将为`sniff_glue_SOURCES`，而不是`sniff-glue_SOURCES`。同样的，名称为`libmumble++.a`的库的源文件应在`libmumble___a_SOURCES`变量中列出。

#### 1.4.2.6. 用户保留变量
> [automake 3.6 Variables reserved for the user](https://www.gnu.org/software/automake/manual/automake.html#User-Variables)

GNU编码标准保留了一些Makefile变量，以供“用户”（构建软件包的人）使用，这些变量有些可能会被Autoconf预设。例如：
| 用户变量   | 说明            |
| :--------- | :-------------- |
| `CC`       | c编译器         |
| `CXX`      | c++编译器       |
| `CPPFLAGS` | c/c++的预编译宏 |
| `CFLAGS`   | c编译标识       |
| `CXXFLAGS` | c++编译标识     |
| `LDFLAGS`  | 链接标识        |

Automake为每个用户标志变量引入了一个特定于automake的影子变量（没有为CC之类的变量引入影子变量，因为它们没有意义）。影子变量的名称是在用户变量名前加上`AM_`。例如，`CFLAGS`的影子变量为`AM_CFLAGS`。软件包维护者（即Makefile.am和configure.ac文件的作者）可以根据需要调整这些影子变量。

#### 1.4.2.7. 目标变量
> [automake 27.6 Program and Library Variables](https://www.gnu.org/software/automake/manual/automake.html#Program-and-Library-Variables)

与每个程序相关联的是变量集合，可用于修改该程序的构建方式。每个库都有类似的此类变量列表。程序或库的规范名称用作命名这些变量的基础。以程序或库名为`maude`举例，常见的变量如下：
| 变量             | 说明                 |
| :--------------- | :------------------- |
| `maude_SOURCES`  | 源文件               |
| `maude_LDADD`    | 添加其他对象到程序中 |
| `maude_LIBADD`   | 添加其他对象到库中   |
| `maude_CPPFLAGS` | c/c++的预编译宏      |
| `maude_CFLAGS`   | c编译标识            |
| `maude_CXXFLAGS` | c++编译标识          |
| `maude_LDFLAGS`  | 链接标识             |

#### 1.4.2.8. 其他
`SUBDIRS`  
`EXTRA_DIST`  
`BUILT_SOURCES`  


### 1.4.3. Makefile.am常用变量说明
<table>
    <tr>
        <td colspan="2">构建目标类型</td>
        <td>示例</td>
        <td>说明</td>
    </tr>
    <tr>
        <td rowspan="4" colspan="2">可执行程序</td>
        <td>bin_PROGRAMS = sample</td>
        <td>构建的可执行程序sample，并安装在bin目录</td>
    </tr>
    <tr>
        <td>sample_SOURCES = test.c</td>
        <td>参与构建sample的源文件</td>
    </tr>
    <tr>
        <td>nodist_sample_SOURCES = mydefines.h</td>
        <td>参与构建sample的源文件但不分发</td>
    </tr>
    <tr>
        <td>sample_LDADD = ../clist/clist.la</td>
        <td>需要链接到sample的库文件</td>
    </tr>
    <tr>
        <td rowspan="8">库</td>
        <td rowspan="4">静态库</td>
        <td>lib_LIBRARIES = libclist.a</td>
        <td>构建的静态库clist.a，并安装在lib目录</td>
    </tr>
    <tr>
        <td>libclist_a_SOURCES = clist.c</td>
        <td>参与构建clist静态库的源文件</td>
    </tr>
    <tr>
        <td><p>libclist_a_LIBADD = $(LIBOBJS) $(ALLOCA) \<br>sub1/libsub1.la \<br> sub2/libsub2.a</p></td>
        <td>需要链接到clist静态库的其他对象</td>
    </tr>
    <tr>
        <td>clistincludedir = $(includedir)/clist<br>nobase_clistinclude_HEADERS = clist.h</td>
        <td>安装到$(clistincludedir)目录下的头文件，且按照原目录结构组织</td>
    </tr>
    <tr>
    <td rowspan="4">libtool库</td>
        <td>lib_LTLIBRARIES = libclist.la</td>
        <td>构建的libtool库（静态库或动态库），并安装在lib目录</td>
    </tr>
    <tr>
        <td>libclist_la_SOURCES = clist.c</td>
        <td>参与构建libtool库的源文件</td>
    </tr>
    <tr>
        <td>nodist_libclist_la_SOURCES = config.h</td>
        <td>参与构建libtool库的源文件，但不分发</td>
    </tr>
    <tr>
        <td>clistincludedir = $(includedir)/clist<br>nobase_clistinclude_HEADERS = clist.h</td>
        <td>安装到$(clistincludedir)目录下的头文件，且按照原目录结构组织<</td>
    </tr>
    <tr>
        <td rowspan="1" colspan="2">头文件</td>
        <td>include_HEADERS = clist.h</td>
        <td>构建目标的头文件，并安装到include目录，参考<a href="https://www.gnu.org/software/automake/manual/automake.html#Headers">automake 9.2 Header files</a></td>
    </tr>
    <tr>
        <td rowspan="1" colspan="2">源文件</td>
        <td>sample_SOURCES = test.c</td>
        <td>构建目标的源文件，并分发，参考<a href="https://www.gnu.org/software/automake/manual/automake.html#Default-_005fSOURCES">automake 8.5 Default _SOURCES</a></td>
    </tr>
</table>

当一些头文件不需要在外部被引用时，也应该列在`_SOURCES`中，而不是`_HEADERS`。例如，库的头文件一般列出在`_HEADERS`，程序的头文件一般列出在`_SOURCES`中。另外有一些参与目标构建但既不属于目标对象，又不需要被外部引用的头文件，则可以使用`noinst_HEADERS`，比较典型的如configure生成的config.h文件。

`_LDADD`和`_LIBADD`不适合传递程序特定的链接器标志（-l，-L，-dlopen和-dlpreopen除外）。



## 1.5. 一些常见问题
### 1.5.1. 标识变量的顺序问题
> [automake 27.6 Flag Variables Ordering](https://www.gnu.org/software/automake/manual/automake.html#Flag-Variables-Ordering)

以CPPFLAGS为例，automake中有三个有关该标识变量，`CPPFLAGS`、`AM_CPPFLAGS`、`maude_CPPFLAGS`，用于传递给C/C++预处理器，CPPFLAGS是用户变量，AM_CPPFLAGS是Automake变量，而mumble_CPPFLAGS是特定于目标对象的变量，也即每个目标变量。  
**编译时，automake始终使用这些变量中的两个，如果`maude_CPPFLAGS`定义了，则使用该变量，否则，使用AM_CPPFLAGS，第二个变量始终是CPPFLAGS。该规则也同样适用于其他类似标记。**  例如：  
```makefile
bin_PROGRAMS = foo bar
foo_SOURCES = xyz.c
bar_SOURCES = main.c
foo_CPPFLAGS = -DFOO
AM_CPPFLAGS = -DBAZ
```
xyz.o将使用`$(foo_CPPFLAGS)$(CPPFLAGS)`进行编译（因为xyz.o是foo目标的一部分），而main.o将使用`$(AM_CPPFLAGS)$(CPPFLAGS)`进行编译（因为bar目标变量未定义）。

推荐使用额外自定义的变量，然后使用目标对象的标记变量追加。例如：
```makefile
AM_CFLAGS = $(WARNINGCFLAGS)
bin_PROGRAMS = prog1 prog2
prog1_SOURCES = ...
prog2_SOURCES = ...
prog2_CFLAGS = $(LIBFOOCFLAGS) $(AM_CFLAGS)
prog2_LDFLAGS = $(LIBFOOLDFLAGS)
```

### 1.5.2. 如何引用预定义的变量
> [autoconf 20.5 How Do I #define Installation Directories?](https://www.gnu.org/savannah-checkouts/gnu/autoconf/manual/autoconf-2.69/autoconf.html#Defining-Directories)

例如想将`bindir`定义到程序中使用，直接使用以下方式是行不通的：
```shell
AC_DEFINE_UNQUOTED([INSTALL_BINDIR], [$bindir],
            [Define to the read-only architecture-independent
             data directory.])
```
最终得到的结果是：
```c
 #define DINSTALL_BINDIR "${prefix}/bin"
```
因为该行为是GNU编码标准强制规定的，要达到目的，不能使用`AC_DEFINE`，而是使用Makefile通过编译标识传递，或者定义到专门的头文件中。  
- 方式一：使用Makefile通过编译标识传递
```
AM_CPPFLAGS = -DINSTALL_BINDIR='"$(bindir)"'
```
- 方式二：创建一个专用头文件
```
DISTCLEANFILES = mydefines.h
    mydefines.h: Makefile
            echo '#define INSTALL_BINDIR "$(bindir)"' >$@
```

## 1.6. clist示例工程
[https://gitee.com/tangm421/clist.git](https://gitee.com/tangm421/clist.git)


## 1.7. 参考资料
> [autoconf][autoconf]  
> [automake][automake]  
> [libtool][libtool]  
> [autoconf宏定义][autoconf宏定义]  
> [Autotools 工具][Autotools 工具]    
> [绝世秘籍之GNU构建系统与Autotool概念分析][绝世秘籍之GNU构建系统与Autotool概念分析]  
> [automake,autoconf使用详解][automake,autoconf使用详解]  
> [Autoconf中文手册][Autoconf中文手册]



[autoconf]:https://www.gnu.org/software/autoconf/manual/
[automake]:https://www.gnu.org/software/automake/manual/
[libtool]:https://www.gnu.org/software/libtool/manual/libtool.html
[autoconf宏定义]:https://www.csdn.net/gather_25/MtjaMgysNTYwMTItYmxvZwO0O0OO0O0O.html
[Autotools 工具]:https://www.jianshu.com/p/b3b0a090a01e
[绝世秘籍之GNU构建系统与Autotool概念分析]:https://blog.csdn.net/looper66/article/details/52825210
[automake,autoconf使用详解]:https://www.laruence.com/2009/11/18/1154.html
[Autoconf中文手册]:https://www.cnblogs.com/fengwei/p/4394165.html