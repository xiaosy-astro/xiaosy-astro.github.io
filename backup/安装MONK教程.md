
### 1. 在 ./sim5 下构建 `sim5` 库

首先进入到`sim5`目录下，使用vim打开`Makefile`文件，做如下修改
```shell
CC=gcc改为CC=gcc-15
```
然后构建：
```shell
cd sim5
make
```

### 2 修改 electron_population.h 的第 17 行。

设置正确的值 hotdir 是包含“logthetat.dat , logx.dat”和“hotx.dat”的目录路径，这些文件用于保存热克莱因-仁科截面的数据。我把它们放在了“./data”目录下，但你必须使用绝对路径。

### 3 编辑 Makefile 为以下变量分配适当的值

* EXTLIB_SIM5: 包含 sim5lib.h 的目录路径
* EXTLIB_SIM5OBJ: sim5lib.o 的路径
* OBJDIR:目标文件目录
* BINDIR: 二进制文件目录
* TRASHDIR: 垃圾目录
* INSTALLDIR：放置可执行二进制文件的目录

在项目目录下，

```shell
Vim Makefile
```

```shell
CXX = g++改为CXX = g++-15

STD = -std=c++14改为STD = -std=c++17

## 添加以下内容
EXTLIB_SIM5 = /Users/btwang/workshop/applications/monk/lib/sim5
EXTLIB_SIM5OBJ = /Users/btwang/workshop/applications/monk/lib/sim5/sim5lib.o
BINDIR = /Users/btwang/workshop/applications/monk/bin/temp
OBJDIR = /Users/btwang/workshop/applications/monk/obj/temp
TRASHDIR = /Users/btwang/workshop/applications/monk/trash
INSTALLDIR = /Users/btwang/workshop/applications/monk/install
```

在终端下，输入以下内容：
```shell
export export OMPI_CXX=g++-15
export PATH=/opt/homebrew/bin:$PATH
```
### 4 make objs 构建对象

### 5 make 3dcorona 3dcorona_mpi calspec

### 附录

代码链接：[MONK](https://projects.asu.cas.cz/zhang/monk)
文章链接：[Constraining the Size of the Corona with Fully Relativistic Calculations of Spectra of Extended Coronae. I. The Monte Carlo Radiative Transfer Code](https://iopscience.iop.org/article/10.3847/1538-4357/ab1261)