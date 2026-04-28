## 一、准备
### 1. 下载KerrC
```shell
git clone https://gitlab.com/krawcz/kerrc-x-ray-fitting-code.git
```

### 2. 安装hdf5

```shell
brew install hdf5
```
### 3. 下载安装sherpa

```shell
conda create -n sherpa -c https://cxc.cfa.harvard.edu/conda/sherpa -c conda-forge sherpa
conda activate sherpa
conda install -c conda-forge matplotlib astropy
```
### 4. 下载和安装boost

下载
```shell
wget https://archives.boost.io/release/1.90.0/source/boost_1_89_0.tar.bz2
```
安装
```shell
cd boost_1_89_0
./bootstrap.sh --with-libraries=python
./b2 -j4
# 屏幕打印
# The Boost C++ Libraries were successfully built!
# The following directory should be added to compiler include paths:
# /Users/btwang/workshop/code/polarized_radiation_transfer_code/kerrc-x-ray-fitting-code/boost_1_89_0
# The following directory should be added to linker library paths:
# /Users/btwang/workshop/code/polarized_radiation_transfer_code/kerrc-x-ray-fitting-code/boost_1_89_0/stage/lib
```

## 二、编译
在进入kerrc-x-ray-fitting-code/raytracing/目录，临时把 Homebrew 放到前面（Apple Silicon 一般是 /opt/homebrew）：
```shell
export PATH="/opt/homebrew/bin:$PATH"
which h5c++
# 现在应该是 /opt/homebrew/bin/h5c++
```
```shell
make
# 最后打印的内容：5 warnings generated.
# h5c++ -lm aliev.o glampedakis.o kerr.o metric.o pani.o wind.o N.o corona.o dataWriter.o xTrack.o -o xTrack
```
## 三、运行
使用以下类型的语句运行代码： ./xTrack -d -D -K -n 2000 -r -10 -c 0000000 -f outdir 这些选项在代码 xTrack.cpp 中进行了解释。
```shell
 ./xTrack -d -D -K -n 2000 -r -10 -c 0000000 -f outdir
```
参数来自 TCLAP 的命令行定义，含义如下（对应 xTrack.cpp）：

-d: 选择热盘模拟（与 -g/-p 互斥）xTrack.cpp (line 166)，xTrack.cpp (line 171)
-D: 启用 Chandrasekhar 盘面反射 xTrack.cpp (line 189)
-K: 选择 Kerr 几何 xTrack.cpp (line 175)
-n 2000: 模拟光子数；热盘时是“每个径向 bin 的光子数”，灯柱时是总数 xTrack.cpp (line 205)
-r -10: 随机种子 xTrack.cpp (line 206)
-c 0000000: corona ID 字符串；提供后会初始化 corona 并用其配置 spin，同时也会被拼到输出文件名上 xTrack.cpp (line 186)，xTrack.cpp (line 262)，xTrack.cpp (line 305)
-f outdir: 输出文件名前缀；实际写入文件为 <outFile><coronaID>.dat（这里会变成 outdir0000000.dat）xTrack.cpp (line 204)，xTrack.cpp (line 305)
补充：-n、-r、-f 在代码里是必需参数。

## 四、修改配置
1. 将quickstart/目录copy到boost_1_89_0/libs/python/example/quickstart

```shell
cp -R quickstart boost_1_89_0/libs/python/example/quickstart
```
2. 切换到boost_1_89_0/libs/python/example/quickstart/quickstart/，将以下内容替换quickstart/comp
```shell
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

if [[ -z "${BOOST_ROOT:-}" ]]; then
  PROBE="${SCRIPT_DIR}"
  for _ in {1..8}; do
    if [[ -f "${PROBE}/boost/version.hpp" ]]; then
      BOOST_ROOT="${PROBE}"
      break
    elif [[ -f "${PROBE}/boost_1_89_0/boost/version.hpp" ]]; then
      BOOST_ROOT="${PROBE}/boost_1_89_0"
      break
    fi
    PROBE="$(dirname "${PROBE}")"
  done
fi

if [[ -z "${BOOST_ROOT:-}" ]]; then
  BOOST_ROOT="$(brew --prefix boost)"
fi

BOOST_LIB_DIR="${BOOST_ROOT}/stage/lib"
if [[ ! -d "${BOOST_LIB_DIR}" ]]; then
  BOOST_LIB_DIR="${BOOST_ROOT}/lib"
fi

H5CXX="$(command -v h5c++ || true)"
if [[ -n "${H5CXX}" ]]; then
  HDF5_PREFIX="$(cd "$(dirname "${H5CXX}")/.." && pwd)"
else
  HDF5_PREFIX="/opt/homebrew"
fi
HDF5_INC="${HDF5_PREFIX}/include"
HDF5_LIB="${HDF5_PREFIX}/lib"

PY_INC="$(python3-config --includes)"
PY_LDFLAGS="$(python3-config --ldflags)"
NUMPY_INC="$(python3 - <<'PY'
import numpy
print(numpy.get_include())
PY
)"

BOOST_PY_LIB="$(ls "${BOOST_LIB_DIR}"/libboost_python*.dylib "${BOOST_LIB_DIR}"/libboost_python*.a 2>/dev/null | head -n 1)"
BOOST_NUMPY_LIB="$(ls "${BOOST_LIB_DIR}"/libboost_numpy*.dylib "${BOOST_LIB_DIR}"/libboost_numpy*.a 2>/dev/null | head -n 1)"

if [[ -z "${BOOST_PY_LIB}" || -z "${BOOST_NUMPY_LIB}" ]]; then
  echo "Boost.Python/NumPy libs not found in ${BOOST_LIB_DIR}" >&2
  exit 1
fi

c++ -std=c++11 -fPIC -O0 -g -Wall \
  -I"${BOOST_ROOT}" -I"${NUMPY_INC}" ${PY_INC} \
  -I"${HDF5_INC}" \
  -c -o dataReader.o dataReader.cpp

# Build a Python extension module on macOS.
c++ -bundle -undefined dynamic_lookup -o dataReader.so dataReader.o \
  "${BOOST_PY_LIB}" "${BOOST_NUMPY_LIB}" \
  -L"${HDF5_LIB}" -lhdf5_cpp -lhdf5 \
  ${PY_LDFLAGS}
```
3. 编译
```shell
./comp
```
## 五、下载XILLVER的Cp_3.6表格
```
wget https://sites.srl.caltech.edu/~javier/xillver/tables/xillverCp_v3.6.fits
```

