这份说明记录了在 Apple Silicon 设备上成功安装 AutoPhOT 的完整步骤，
适用于需要全功能（Astrometry.net + HOTPANTS + PyZOGY）的场景。

## 0. 前置条件

- 已安装 Conda（Miniconda/Anaconda）
- 已安装 Homebrew

建议在全新终端执行以下步骤。

## 1. 安装 Rosetta（首次需要）

```bash
softwareupdate --install-rosetta --agree-to-license
```

## 2. 创建并固定 osx-64 的 conda 环境

AutoPhOT 依赖 `astroscrappy==1.0.8`，该版本在 `osx-arm64` 没有包，
因此必须使用 `osx-64`（Rosetta）。

```bash
CONDA_SUBDIR=osx-64 conda create -n autophot_env python=3.7 -y
conda activate autophot_env
conda config --env --set subdir osx-64
```

## 3. 安装 AutoPhOT

```bash
conda install -c conda-forge -c astropy -c astro-sean autophot -y
```

### 3.1 修复 Python 搜索路径（必须）

该 conda 包会把 AutoPhOT 安装到 `$CONDA_PREFIX/site-packages`，
但默认 `sys.path` 只包含 `$CONDA_PREFIX/lib/python3.7/site-packages`。
需要添加一个 `.pth` 文件。

```bash
python - <<'PY'
import os, sys
pth = os.path.join(sys.prefix, "lib", "python3.7", "site-packages", "autophot_conda_site.pth")
target = os.path.join(sys.prefix, "site-packages")
with open(pth, "w") as f:
    f.write(target + "\n")
print(pth, "->", target)
PY
```

验证（建议在非仓库目录执行）：

```bash
cd /tmp
python -c "import autophot; print(autophot.__version__)"
```

> 如果在 AutoPhOT 源码目录下执行导入，可能会优先加载本地目录的同名包，
> 建议在 `/tmp` 等目录验证环境安装是否正确。

## 4. 安装可选功能 1：Astrometry.net

```bash
HOMEBREW_NO_AUTO_UPDATE=1 brew install astrometry-net
```

下载索引文件（约 40GB），建议放在：

```
/opt/homebrew/Cellar/astrometry-net/0.97/data
```
[下载网址](https://data.astrometry.net/4200/)
可选：设置环境变量（让 solve-field 能稳定找到索引）：

```bash
export ASTROMETRY_NET_DATA_DIR=/opt/homebrew/Cellar/astrometry-net/0.97/data
```

验证：

```bash
solve-field /path/to/your_image.fits --overwrite --no-plot
```

在 AutoPhOT 默认配置中设置：

```
solve_field_exe_loc: /opt/homebrew/bin/solve-field
```

## 5. 安装可选功能 2：HOTPANTS

### 5.1 安装 cfitsio

```bash
HOMEBREW_NO_AUTO_UPDATE=1 brew install cfitsio
```

### 5.2 修改 Makefile

在 `hotpants/Makefile` 中：

```
CFITSIOINCDIR = /opt/homebrew/opt/cfitsio/include
LIBDIR        = /opt/homebrew/opt/cfitsio/lib
```

在 `hotpants/Makefile.macosx` 中：

```
CFITSIOINCDIR = /opt/homebrew/opt/cfitsio/include
LIBDIR = /opt/homebrew/opt/cfitsio/lib
LIBS  =  -L$(LIBDIR) -lm -lcfitsio
```

### 5.3 修复 macOS 的 malloc.h 问题

对以下文件将 `#include <malloc.h>` 替换为：

```
#if defined(__APPLE__)
#include <stdlib.h>
#else
#include <malloc.h>
#endif
```

文件列表：

- `alard.c`
- `main.c`
- `maskim.c`
- `extractkern.c`
- `extractkernOnes.c`
- `functions.c`

### 5.4 编译

```bash
cd hotpants
make -f Makefile.macosx
```

生成的可执行文件：

```
/path/to/hotpants/hotpants
```

在 AutoPhOT 默认配置中设置：

```
hotpants_exe_loc: /path/to/hotpants/hotpants
```

## 6. 安装可选功能 3：PyZOGY

PyZOGY 依赖 `statsmodels`，Python 3.7 下建议使用 0.13.x：

```bash
conda activate autophot_env
conda install -c conda-forge "statsmodels<0.14" -y
cd /path/to/PyZOGY
pip install --no-deps .
```

验证：

```bash
python -c "import PyZOGY, statsmodels; print(PyZOGY.__file__); print(statsmodels.__version__)"
```

## 7. 常见问题

- conda 安装时提示包损坏：可尝试

```bash
conda clean --packages --tarballs -y
```

- `import autophot` 没有 `__version__`：通常是运行目录下有同名文件夹导致，
  在非源码目录执行导入即可。

