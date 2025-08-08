目标：流水线产品不可用，我需要重新提取数据，每一次观测数据含有event文件，需要通过xselect软件从event中提取数据产品，数据产品包括能谱（包括源和背景能谱）和光变曲线（源和背景的光变曲线）。

思路：
- 1. 我知道目标源的坐标，可以生成源和背景的区域文件（src1.reg和bkg1.reg）；
- 2. 使用xselect读取event文件，导入reg提取能谱和光变曲线。

目的：`处理值班认证的2-3级数据`

产品：
- 1. 自动使用`lcurve`画出净光变曲线，未扣除背景的光变曲线和背景的光变曲线3*1面板。
- 2. 自动生成了.xcm文档，可以进入`Xspec`拟合

## 1. 定义生成region文件的函数

```python
import os
import re
def write_ds9_annulus_region(ra, dec, output_dir='.', filename='bkg.reg',
                              inner_radius='1080.000"', outer_radius='2160.000"',comment='bkg'):
    """
    写入 DS9 annulus region 文件

    参数:
    - ra, dec: 坐标（浮点型，单位为度）
    - output_dir: 输出目录（默认为当前目录）
    - filename: 文件名（默认为 bkg.reg）
    - inner_radius, outer_radius: 环形区域的内/外半径（字符串，带单位，如 "1080.000"）

    返回:
    - 文件的完整路径
    """
    os.makedirs(output_dir, exist_ok=True)
    full_path = os.path.join(output_dir, filename)

    with open(full_path, 'w') as f:
        f.write('# Region file format: DS9 version 4.1\n')
        f.write('global color=green dashlist=8 3 width=1 font="helvetica 10 normal roman" '
                'select=1 highlite=1 dash=0 fixed=0 edit=1 move=1 delete=1 include=1 source=1\n')
        f.write('icrs\n')
        comment_str = f' # text={{ {comment} }}' if comment else ''
        f.write(f'annulus({ra},{dec},{inner_radius},{outer_radius}){comment_str}\n')

    return full_path

import os

def write_ds9_circle_region(ra, dec, output_dir='.', filename='bkg.reg',
                              radius='543.243"',comment='src'):
    """
    写入 DS9 annulus region 文件

    参数:
    - ra, dec: 坐标（浮点型，单位为度）
    - output_dir: 输出目录（默认为当前目录）
    - filename: 文件名（默认为 bkg.reg）
    - radius: 源区域的半径（字符串，带单位，如 "540.000"）

    返回:
    - 文件的完整路径
    """
    os.makedirs(output_dir, exist_ok=True)
    full_path = os.path.join(output_dir, filename)

    with open(full_path, 'w') as f:
        f.write('# Region file format: DS9 version 4.1\n')
        f.write('global color=red dashlist=8 3 width=1 font="helvetica 10 normal roman" '
                'select=1 highlite=1 dash=0 fixed=0 edit=1 move=1 delete=1 include=1 source=1\n')
        f.write('icrs\n')
        comment_str = f' # text={{ {comment} }}' if comment else ''
        f.write(f'circle({ra},{dec},{radius}){comment_str}\n')

    return full_path

from typing import Tuple

def parse_filename(filename: str) -> Tuple[str, str, str]:
    """
    从标准文件名中解析 obsid, cmos_num, 和 src_num。
    """
    pattern = r'ep(\d+)wxtCMOS(\d+)s(\d+)v\d+'
    match = re.match(pattern, filename)
    if not match:
        raise ValueError(f"文件名 '{filename}' 格式不符合 'ep...wxtCMOS...s...v...' 的预期格式！")
    obsid, cmos_num, src_num = match.group(1), match.group(2), match.group(3)
    return obsid, cmos_num, src_num
```

## 2. 生成src.reg和bkg.reg

```python
import os
import subprocess
from datetime import datetime as dt
import math
from astropy.io import fits
import numpy as np
import matplotlib.pyplot as plt
plt.style.use('https://raw.githubusercontent.com/wangboting/python-style/main/pythonstyle.style')
plt.rcParams['font.sans-serif'] = ['SimSun']
plt.rcParams['axes.unicode_minus'] = False
plt.rcParams['font.size'] = 12
plt.rcParams['font.family'] = 'Times New Roman'


ep_wxt_filename = 'ep11900327297wxtCMOS11s1v1'

ra_src,dec_src = 313.264, -4.648
# ra_bkg,dec_bkg = 227.429, -39.591
date = '20250730'
os.chdir(f'/Users/btwang/workshop/EP/WXT_datasets/{date}/{ep_wxt_filename}')
print("当前目录:", os.getcwd())
output_dir = "."
print("output_dir:",output_dir)
obsid, cmos_num, src_num = parse_filename(ep_wxt_filename)
print(f"  解析结果 -> obsid={obsid}, cmos_num={cmos_num}, src_num={src_num}")







src_reg = f'src{src_num}.reg'
## 记得手动修改reg注释
### WXT选源原则：67像素的圆，8.108 arcsec /pixel ,相当于 9 arcmin
comment_src = f's{src_num}'
src_reg_path = write_ds9_circle_region(ra_src, dec_src, output_dir, src_reg,comment=comment_src)
print(f'Region file saved to: {src_reg_path}')

bkg_reg = f'bkg{src_num}.reg'

#bkg_reg_path = write_ds9_annulus_region(ra_bkg, dec_bkg, output_dir, bkg)
bkg_reg_path = write_ds9_annulus_region(ra_src, dec_src, output_dir, bkg_reg)
print(f'Region file saved to: {bkg_reg_path}')
```

## 3. xselect提取能谱和光变曲线数据产品

`heasoftpy` 不支持直接调用 `xselect`。不过我们可以通过写`.xco`实现自动提取数据产品。这里我通过`python`针对上一步得到的 `    src1.reg` 和`bkg1.reg`文件，写一个脚本。

```python
# 写入 XSELECT 脚本
print("当前目录:", os.getcwd())
evt_file = f'ep{obsid}wxt{cmos_num}po_cl.evt'
src_spec = f'src{src_num}_spec.pha'
bkg_spec = f'bkg{src_num}_spec.pha'
src_lc = f'src{src_num}_lc.lc'
bkg_lc = f'bkg{src_num}_lc.lc'
image_src = f'src{src_num}_img.img'
###  源的能谱和光变曲线
xselect_src = f"""src
yes
set mission ep
read event {evt_file}
.
filter pha_cutoff 50 400
extract image xybinsize=16
save image {image_src}
clear pha_cutoff
filter region {src_reg}        
extract spectrum
save spectrum {src_spec}
extract curve
save curve {src_lc}

exit
no
"""

###  背景的能谱和光变曲线
xselect_bkg = f"""bkg
yes
set mission ep
read event {evt_file}
.
filter region {bkg_reg}          
extract spectrum
save spectrum {bkg_spec}    
extract curve
save curve {bkg_lc}          
exit
no
"""

# 保存为 .xco 文件
with open("xselect_src.xco", "w") as f:
    f.write(xselect_src)

with open("xselect_bkg.xco", "w") as f:
    f.write(xselect_bkg)


#### 检查如果有输出同名文件，删除

if os.path.exists(src_spec):
    os.remove(src_spec)
    print(f"Deleted source pha file: {src_spec}")

if os.path.exists(bkg_spec):
    os.remove(bkg_spec)
    print(f"Deleted background pha file: {bkg_spec}")

if os.path.exists(src_lc):
    os.remove(src_lc)
    print(f"Deleted source lc file: {src_lc}")

if os.path.exists(bkg_lc):
    os.remove(bkg_lc)
    print(f"Deleted background lc file: {bkg_lc}")

if os.path.exists(image_src):
    os.remove(image_src)
    print(f"Deleted background lc file: {image_src}")

### 源的能谱和光变曲线
# 调用 XSELECT（需要 HEASoft 环境已激活）
subprocess.run("xselect  @xselect_src.xco", shell=True, check=True)


###  背景的能谱和光变曲线
subprocess.run("xselect  @xselect_bkg.xco", shell=True, check=True)
```

## 4. 画光变曲线

### <span style="color:red">4.1 使用 `lcmath` 提取净光变曲线</span>
#### 4.1.1 提取backscal参数

```python
with fits.open(f"{src_spec}") as hdul:
    # 打印包含的 HDU 列表
    scal_src = hdul[1].header.get("BACKSCAL")
    print("src_BACKSCAL =", scal_src)

with fits.open(f"{bkg_spec}") as hdul:
    # 打印包含的 HDU 列表
    scal_bkg = hdul[1].header.get("BACKSCAL")
    print("bkg_BACKSCAL =", scal_bkg)
```

#### 4.1.2 计算缩放背景的因子

```python
infile = src_lc 
bgfile = bkg_lc
outfile='net.lc'
multi = 1.0
multb = scal_src / scal_bkg
print("multb",multb)
# 确保输入文件存在
if not os.path.exists(infile):
    print(f"错误：输入文件 {infile} 不存在")
    exit(1)
if not os.path.exists(bgfile):
    print(f"错误：背景文件 {bgfile} 不存在")
    exit(1)
cmd = f"lcmath infile={infile} bgfile={bgfile} outfile={outfile} multi={multi} multb={multb:.5f} addsubr=no"
```

#### 4.1.3 提取净光变曲线

```python
try:
    result = subprocess.run(cmd, check=True, capture_output=True, text=True, shell=True)
    print("Command output:", result.stdout)
except subprocess.CalledProcessError as e:
    print("Error occurred:")
    print("Command:", e.cmd)
    print("Return code:", e.returncode)
    print("Output:", e.stdout)
    print("Error:", e.stderr)
```

### <span style="color:red">4.2 使用 `lcurve` 画出净光变曲线</span>

```python
three_lc = 'net_src_bkg.lc'
# plotfile = 'net_src_bkg.gif'
plotfile = 'net_src_bkg.eps'
time_bin = 12.0         # 时间分箱，单位：秒

# === Step 1：从 GTI 读取总有效曝光时间 ===
def get_gti_exposure(filename):
    with fits.open(filename) as hdul:
        gti_data = None
        for hdu in hdul:
            if hdu.name.upper() in ['GTI', 'STDGTI']:
                gti_data = hdu.data
                break
        if gti_data is None:
            raise ValueError("未找到 GTI extension")
        start_times = gti_data['START']
        stop_times = gti_data['STOP']
        total_exposure = sum(stop - start for start, stop in zip(start_times, stop_times))
        return total_exposure

try:
    exposure_time = get_gti_exposure(infile)  # 用源光变文件的 GTI
    print("GTI总曝光时间 =", exposure_time)

    bin_num = math.ceil(exposure_time / time_bin)
    print("bin_num（基于GTI）:", bin_num)
except Exception as e:
    print("出错：", e)
    exit(1)

# === Step 2：检查文件是否存在 ===
for f, name in zip([outfile, infile, bgfile], ['outfile', 'infile', 'bgfile']):
    if not os.path.exists(f):
        print(f"错误：输入文件 {name} ({f}) 不存在")
        exit(1)

# === Step 3：构造 lcurve 命令 ===
cmd = f'lcurve nser=3 cfile1="{outfile}" cfile2="{infile}" cfile3="{bgfile}" ' \
      f'window="-" dtnb={time_bin} nbint={bin_num} outfile="{three_lc}" ' \
      f'plot=yes plotdev="/xw" plotdnum=3 clobber=yes'

# === Step 4：发送给 PLT 的命令 ===
# plt_commands = f"exit\n"
plt_commands = f"res y 0,0.1\nhardcopy {plotfile}/cps\nexit\n"
# === Step 5：运行命令并自动传输 PLT 指令 ===
try:
    result = subprocess.run(
        cmd,
        check=True,
        capture_output=True,
        text=True,
        shell=True,
         input=plt_commands
    )
    print("lcurve 输出:", result.stdout)
#    print(f"图已保存为: {plotfile}")
except subprocess.CalledProcessError as e:
    print("执行 lcurve 时出错：")
    print("命令:", e.cmd)
    print("返回码:", e.returncode)
    print("输出:", e.stdout)
    print("错误信息:", e.stderr)
```
### 4.3 Python可视化光变曲线

```python
from astropy.io import fits
import matplotlib.pyplot as plt

# 打开 .lc 文件
filename = f"{three_lc }"
with fits.open(filename) as hdul:
    data = hdul[1].data  # 通常光变数据在扩展1（即 hdul[1]）

    # 提取数据
    time = data['TIME']
    rate_net = data['RATE1']
    err_net = data['ERROR1']
    rate_src = data['RATE2']
    err_src = data['ERROR2']
    rate_bkg = data['RATE3']
    err_bkg = data['ERROR3']

# 开始绘图
plt.figure(figsize=(7, 5))
scaling_factor = 1.0 / 12.0
# net curve
plt.errorbar(
    time, 
    rate_net, 
    yerr=err_net,
    fmt='o', 
    color='black', 
    elinewidth=1, 
    markersize=3, 
    label='net')

# src curve
# plt.errorbar(time, rate_src, yerr=err_src, fmt='^', color='grey', elinewidth=1, markersize=3, markerfacecolor='none',label='src')

# bkg curve
# plt.errorbar(time, rate_bkg*scaling_factor, yerr=err_bkg*scaling_factor, fmt='x', color='red', elinewidth=1, markersize=5, label='bkg')
# plt.errorbar(time, rate_bkg, yerr=err_bkg, fmt='x', color='red', elinewidth=1, markersize=5, label='bkg')
plt.errorbar(
    time,
    rate_bkg * scaling_factor,
    yerr=err_bkg * scaling_factor,
    fmt='x',
    color='red',
    elinewidth=1,
    markersize=3,
    label='bkg',
    alpha=0.5  # 透明度设置为50%
)
# 设置图形
plt.xlabel("Time (s)")
plt.ylabel('RATE (Count/sec)')
ymin = min(rate_net)
ymax = max(rate_net)
# plt.xlim(0,1000)
plt.ylim(-0.1, 2.0)
plt.title(f'ep{obsid}wxt{cmos_num}s{src_num}  Bin time = {time_bin}s')
plt.legend()
# plt.grid(True)
plt.tight_layout()
plt.savefig(f'{three_lc }.pdf',dpi=400,bbox_inches="tight")
plt.show()
```
## 5. 能谱分析
### <span style="color:red">5.1 使用grppha 合并能道,check `bkg.pha`,`rmf`,`arf` keywords to `src.pha`</span>

```python
grp_infile = src_spec
grp_outfile = f'src{src_num}_spec.grp'
grp_min_bins = 5 # 按需修改
bkg_file = bkg_spec
arf_file = f'ep{obsid}wxt{cmos_num}s{src_num}.arf' # 需修改为目标源的arf
rmf_file = f'ep{obsid}wxt{cmos_num}.rmf' # 需修改为目标源的rmf

# 手动bin
cmd_grp = f'grppha {grp_infile} {grp_outfile}  comm="group min {grp_min_bins} & chkey backfile {bkg_file} & chkey respfile {rmf_file} & chkey ancrfile {arf_file} & clobber = yes & show keywords & exit"'

# elisa自动bin
# cmd_grp = f'grppha {grp_infile} {grp_outfile}  comm="chkey backfile {bkg_file} & chkey respfile {rmf_file} & chkey ancrfile {arf_file} & clobber = yes & show keywords & exit"'


#### 注意⚠️：文件夹中已有输出文件，会报错，检查如果有输出同名文件，删除

if os.path.exists(grp_outfile):
    os.remove(grp_outfile)
    print(f"Deleted existing file: {grp_outfile}")


try:
    result = subprocess.run(cmd_grp, check=True, capture_output=True, text=True, shell=True)
    print("Command output:", result.stdout)
except subprocess.CalledProcessError as e:
    print("Error occurred:")
    print("Command:", e.cmd)
    print("Return code:", e.returncode)
    print("Output:", e.stdout)
    print("Error:", e.stderr)
```

### 5.2 写一个xcm

先进入Xspec，@xspec_src.xcm，然后直接输入拟合的模型即可

```python
###  xcm
xspec_src = f"""data {grp_outfile}
ignore **-0.5 4.0-**
ignore bad
cpd /xw
statistic cstat
setpl en
abun wilm
"""
with open("xspec_src.xcm", "w") as f:
    f.write(xspec_src)
```
