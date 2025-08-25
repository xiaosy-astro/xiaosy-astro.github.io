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
from extract_region import write_ds9_annulus_region, write_ds9_circle_region, parse_filename
import os
from datetime import datetime as dt
import math
from astropy.io import fits
import numpy as np
from astropy.time import Time
import matplotlib.pyplot as plt
import subprocess
plt.style.use('https://raw.githubusercontent.com/wangboting/python-style/main/pythonstyle.style')
plt.rcParams['font.sans-serif'] = ['SimSun']
plt.rcParams['axes.unicode_minus'] = False
plt.rcParams['font.size'] = 12
plt.rcParams['font.family'] = 'Times New Roman'


ep_wxt_filename = 'ep11916650903wxtCMOS25s1v1'

ra_src, dec_src = 0.803, 36.873
date = '20250819'

# 根目录
work_dir = f'/Users/btwang/workshop/EP/WXT_datasets/{date}/{ep_wxt_filename}'
print("工作目录:", work_dir)

# 产品目录
products_dir = os.path.join(work_dir, "products")
os.makedirs(products_dir, exist_ok=True)   # 如果没有就创建
print("产品目录:", products_dir)

# 解析文件名
obsid, cmos_num, src_num = parse_filename(ep_wxt_filename)
print(f"解析结果 -> obsid={obsid}, cmos_num={cmos_num}, src_num={src_num}")

# ---------------------------
# 生成源区 region 文件
src_reg = f'src{src_num}.reg'
comment_src = f's{src_num}'
src_reg_path = write_ds9_circle_region(
    ra_src, dec_src,
    products_dir,   # 输出目录换成 products
    src_reg,
    comment=comment_src
)
print(f'Source region file saved to: {src_reg_path}')

# ---------------------------
# 生成背景区 region 文件
bkg_reg = f'bkg{src_num}.reg'
bkg_reg_path = write_ds9_annulus_region(
    ra_src, dec_src,
    products_dir,   # 输出目录换成 products
    bkg_reg
)
print(f'Background region file saved to: {bkg_reg_path}')
```

## 3. xselect提取能谱和光变曲线数据产品

`heasoftpy` 不支持直接调用 `xselect`。不过我们可以通过写`.xco`实现自动提取数据产品。这里我通过`python`针对上一步得到的 `    src1.reg` 和`bkg1.reg`文件，写一个脚本。

```python
print("工作目录:", work_dir)
print("产品目录:", products_dir)

src_reg = os.path.join(products_dir, f'src{src_num}.reg')
bkg_reg = os.path.join(products_dir, f'bkg{src_num}.reg')
evt_file = f'ep{obsid}wxt{cmos_num}po_cl.evt'

# 输出文件都放 products 目录
src_spec = os.path.join(products_dir, f'src{src_num}_spec.pha')
bkg_spec = os.path.join(products_dir, f'bkg{src_num}_spec.pha')
src_lc   = os.path.join(products_dir, f'src{src_num}_lc.lc')
bkg_lc   = os.path.join(products_dir, f'bkg{src_num}_lc.lc')
image_src= os.path.join(products_dir, f'src{src_num}_img.img')

# xco 文件也放在 products
xco_src  = os.path.join(products_dir, "xselect_src.xco")
xco_bkg  = os.path.join(products_dir, "xselect_bkg.xco")

### 源的脚本
xselect_src = f"""src
yes
set mission ep
set datadir {work_dir}
read event {evt_file}
filter pha_cutoff 50 400
extract image xybinsize=16
save image {image_src}
clear pha_cutoff
filter region {src_reg}        
extract spectrum
save spectrum {src_spec}
set binsize 1
extract curve
save curve {src_lc}

exit
no
"""

### 背景的脚本
xselect_bkg = f"""bkg
yes
set mission ep
set datadir {work_dir}
read event {evt_file}
filter region {bkg_reg}          
extract spectrum
save spectrum {bkg_spec}    
set binsize 1
extract curve
save curve {bkg_lc}          
exit
no
"""

# 保存 .xco 文件
with open(xco_src, "w") as f:
    f.write(xselect_src)

with open(xco_bkg, "w") as f:
    f.write(xselect_bkg)

# 如果之前有旧输出，删除
for fpath in [src_spec, bkg_spec, src_lc, bkg_lc, image_src]:
    if os.path.exists(fpath):
        os.remove(fpath)
        print(f"Deleted old file: {fpath}")

# 运行 xselect 脚本
subprocess.run(f"xselect @{xco_src}", shell=True, check=True)
subprocess.run(f"xselect @{xco_bkg}", shell=True, check=True)
```

## 4. 画光变曲线

### <span style="color:red">4.1 使用 `lcmath` 提取净光变曲线</span>
#### 4.1.1 提取backscal参数

```python
with fits.open(src_spec) as hdul:
    # 打印包含的 HDU 列表
    scal_src = hdul[1].header.get("BACKSCAL")
    obs_id = hdul[1].header.get("OBS_ID")
    MJD = hdul[1].header.get("MJD-OBS")
    elapsed_time = hdul[1].header.get("TELAPSE")
#    MJD = Time(date_obs, format='isot', scale='utc')
    print("src_BACKSCAL =", scal_src)
    print("elapsed_time=",elapsed_time)

with fits.open(f"{bkg_spec}") as hdul:
    # 打印包含的 HDU 列表
    scal_bkg = hdul[1].header.get("BACKSCAL")
    print("bkg_BACKSCAL =", scal_bkg)
```

#### 4.1.2 计算缩放背景的因子

```python
infile = src_lc 
bgfile = bkg_lc
outfile=f'net_s{src_num}.lc'
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

### <span style="color:red">4.2 使用 `lcurve` 对净光变曲线完成时间bin</span>

```python
three_lc = os.path.join(products_dir, f'net_src{src_num}_bkg.lc')
plotfile = os.path.join(products_dir, f'net_src{src_num}_bkg.eps')

time_bin = 15.0         # 时间分箱，单位：秒

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


exposure_time = get_gti_exposure(infile)  # 用源光变文件的 GTI
print("GTI总曝光时间 =", exposure_time)
bin_num = math.ceil(elapsed_time / time_bin)
print("bin_num（基于GTI经历时间elapsed_time）:", bin_num)


# === Step 2：检查文件是否存在 ===
for f, name in zip([outfile, infile, bgfile], ['outfile', 'infile', 'bgfile']):
    if not os.path.exists(f):
        print(f"错误：输入文件 {name} ({f}) 不存在")
        exit(1)

# === Step 3：构造 lcurve 命令 ===
cmd =( f'lcurve nser=3 cfile1="{outfile}" cfile2="{infile}" cfile3="{bgfile}" '
      f'window="-" dtnb={time_bin} nbint={bin_num} outfile="{three_lc}" '
      f'plot=yes plotdev="/xw" plotdnum=3 clobber=yes'
)
# === Step 4：发送给 PLT 的命令 ===
# plt_commands = f"exit\n"
plt_commands = f"res y 0,0.1\nhardcopy {plotfile}/cps\nexit\n"

# === Step 5：运行命令并自动传输 PLT 指令 ===
subprocess.run(cmd, check=True, shell=True, input=plt_commands, text=True)
```
### 4.3 Python可视化光变曲线

```python
from astropy.io import fits
import matplotlib.pyplot as plt
import string
# 打开 .lc 文件
filename = three_lc
plot_pdf = os.path.join(products_dir, f'{obsid}_net_src{src_num}_bkg.pdf')
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
time_err = time_bin/2
plt.errorbar(
    time, 
    rate_net, 
    xerr= time_err,
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
    xerr= time_err,
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
plt.ylabel('Count Rate (0.5-4.0 keV) (count/sec)')
ymin = min(rate_net)
ymax = max(rate_net)
# plt.xlim(0,1000)
plt.ylim(-0.01, 0.25)
plt.xlim(-time_err,elapsed_time+time_err)
#plt.xlim(-time_err,300)
plt.title(f'ep{obsid}wxt{cmos_num}s{src_num}  Bin time = {time_bin}s')
plt.suptitle(f'ObsID: {obs_id}  OBS-MJD: {MJD:.5f}  R.A., DEC. {ra_src,dec_src} deg', fontsize=10, y=0.98)
plt.legend()
# plt.grid(True)
plt.tight_layout()
plt.savefig(plot_pdf,dpi=400,bbox_inches="tight")
plt.show()
print(f"Plot saved to {plot_pdf}")
```

### 4.3  保存净光变曲线
```python
# === Step 6：保存净光变曲线到新的 FITS ===
time_err = np.full_like(time, time_bin / 2.0)  # 时间误差
col_time = fits.Column(name='TIME', array=time, format='D', unit='s')
col_time_err = fits.Column(name='TIME_ERR', array=time_err, format='D', unit='s')
col_rate = fits.Column(name='RATE', array=rate_net, format='E', unit='count/s')
col_error = fits.Column(name='ERROR', array=err_net, format='E', unit='count/s')

cols = fits.ColDefs([col_time, col_time_err, col_rate, col_error])
hdu = fits.BinTableHDU.from_columns(cols)

# 保留原来的头信息
with fits.open(src_lc) as hdul:
    orig_header = hdul[1].header.copy()

def is_ascii_printable(s):
    try:
        return all(c in string.printable for c in str(s))
    except Exception:
        return False

for key, val in orig_header.items():
    if key not in hdu.header and is_ascii_printable(val):
        hdu.header[key] = val

# 创建 PrimaryHDU + TableHDU
primary_hdu = fits.PrimaryHDU(header=fits.Header())  # 主HDU可以留空或拷原来的
new_hdul = fits.HDUList([primary_hdu, hdu])

output_fits = os.path.join(products_dir, f'{obs_id}_net_lightcurve.fits')
new_hdul.writeto(output_fits, overwrite=True)

print(f"净光变曲线已保存到 {output_fits}")
```
## 5. 能谱分析
### <span style="color:red">5.1 使用grppha 合并能道,check `bkg.pha`,`rmf`,`arf` keywords to `src.pha`</span>

```python
grp_infile = src_spec
grp_outfile = os.path.join(products_dir, f'src{src_num}_spec.grp')
grp_min_bins = 3  # 按需修改
bkg_file = bkg_spec
arf_file = os.path.join(work_dir, f'ep{obsid}wxt{cmos_num}s{src_num}.arf')  
rmf_file = os.path.join(work_dir, f'ep{obsid}wxt{cmos_num}.rmf')            
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

### 5.2 使用`elisa`拟合能谱
```python
import os
from elisa import BayesFit, MaxLikeFit,Data
from elisa.models import TBAbs, PowerLaw,BlackbodyRad,EnFlux,CutoffPL
from elisa.models.xspec import XSagauss,XSdiskbb,XSapec,XScflux,XSbbodyrad
WXT = Data(name=f"{obsid}",
    erange=[0.5, 4.0],specfile=f'{grp_outfile}',group='opt')
#     erange=[0.5, 4.0],specfile=grp_outfile)


fig_spec = WXT.plot_spec(xlog=True, data_ylog=True)
fig_spec.set_size_inches(8, 6)
plt.show()
```

```python
data = [WXT]

def build_model(
    model_type='powerlaw',
    nH_val=0.022,
    flux_range=(0.5, 10.0),
    norm_K=1.0,
    fit_nH=True,
    fit_norm=True
):
    """
    通用模型构建器，支持单模型或组合模型，并控制 nH 和 norm 是否拟合。

    参数:
        model_type: str
            模型类型: 'powerlaw', 'bbodyrad', 'diskbb', 'powerlaw+bbodyrad', 'powerlaw+diskbb'
        nH_val: float
            氢柱密度值，可固定或拟合
        flux_range: tuple
            EnFlux 能量范围 (emin, emax)
        norm_K: float or list/tuple
            单个或多个归一化常数初始值（组合模型时用列表）
        fit_nH: bool
            是否拟合 nH（True 表示拟合，False 表示固定）
        fit_norm: bool
            是否拟合归一化参数 K/norm（True 表示拟合，False 表示固定）

    返回:
        model: 可用于 BayesFit 的完整模型对象
    """
    model_type = model_type.lower()

    # 工具函数：构造 norm 参数
    def build_norm(k, fit):
        return [k, 1e-5, 1e6] if fit else k

    # Step 1: 构建基础模型
    if model_type == 'powerlaw':
        K = build_norm(norm_K, fit_norm)
        base_model = PowerLaw(alpha=[2.0, -1, 5], K=K)

    elif model_type == 'bbodyrad':
        norm = build_norm(norm_K, fit_norm)
        base_model = XSbbodyrad(kT=[0.5, 0.01, 5], norm=norm)

    elif model_type == 'diskbb':
        norm = build_norm(norm_K, fit_norm)
        base_model = XSdiskbb(Tin=[1.0, 0.01, 5], norm=norm)

    elif model_type == 'bbody+radpowerlaw':
        # norm_K 为两个值的列表或元组
        if not isinstance(norm_K, (list, tuple)) or len(norm_K) != 2:
            raise ValueError("For combination models, norm_K must be a list or tuple of two values.")
        pl = PowerLaw(alpha=[2.0, -1, 5], K=build_norm(norm_K[0], fit_norm))
        bb = XSbbodyrad(kT=[0.5, 0.01, 5], norm=build_norm(norm_K[1], fit_norm))
        base_model = bb + pl 

    elif model_type == 'diskbb+powerlaw':
        if not isinstance(norm_K, (list, tuple)) or len(norm_K) != 2:
            raise ValueError("For combination models, norm_K must be a list or tuple of two values.")
        pl = PowerLaw(alpha=[2.0, -1, 5], K=build_norm(norm_K[0], fit_norm))
        dbb = XSdiskbb(Tin=[1.0, 0.01, 5], norm=build_norm(norm_K[1], fit_norm))
        base_model =dbb + pl 
        
    elif model_type == 'apec':
        norm = build_norm(norm_K, fit_norm)
        base_model = XSapec(
            
            Abundanc=1.0,             # 金属丰度，固定为太阳丰度
            Redshift=0.0,                 # 红移
            kT=[1.0, 0.5, 10],     # 温度参数，自行根据科学问题调整
            norm=norm
        )


    else:
        raise ValueError(f"Unsupported model_type: {model_type}")

    # Step 2: EnFlux 包装
    flux_model = EnFlux(emin=flux_range[0], emax=flux_range[1], ngrid=1000, elog=True)
    fluxed_model = flux_model(base_model)

    # Step 3: TBAbs 吸收
    absorption = TBAbs(nH=[nH_val, 1e-3, 10]) if fit_nH else TBAbs(nH=nH_val)

    return absorption * fluxed_model

# 拟合 nH 和 norm
#model = build_model('diskbb+powerlaw', norm_K=[1.0,1.0],fit_nH=True, fit_norm=True)

# 固定 nH，拟合 norm：
# model = build_model('diskbb', nH_val=0.176, fit_nH=False, fit_norm=True)

# 拟合 nH，固定norm
# model = build_model('diskbb+powerlaw', norm_K=[1.0,1.0],fit_nH=True, fit_norm=False)

# 固定nH，固定 norm
# model = build_model('diskbb+powerlaw', nH_val=0.21,norm_K=[1.0,1.0], fit_nH=False, fit_norm=False)
# 创建拟合器
model_name ='powerlaw' # apec拟合会失败
model = build_model(model_name, nH_val=0.09, norm_K=1.0,fit_nH=True, fit_norm=False)
fit = BayesFit(data=[WXT], model=model, stat='wstat')
fit
```

```python
posterior = fit.nuts(progress=False)
posterior
```


```python
fit_result =posterior.plot("data vFv rd")
plt.savefig(f'{obsid}_{model_name}_spectrum_data.pdf',dpi=400,bbox_inches="tight")
```

```python
fig = posterior.plot.plot_qq(detrend=False)
```

```python
fig_corner = posterior.plot.plot_corner()
corner_pdf = os.path.join(products_dir, f'{obsid}_{model_name}_ELSISA_fit_spectrum_result.pdf')
plt.savefig(corner_pdf,dpi=400,bbox_inches="tight")
```


### 5.2 写一个xcm

先进入Xspec，@xspec_src.xcm，然后直接输入拟合的模型即可

```python
###  xcm
xcm_src  = os.path.join(products_dir, "xspec_src.xcm")
xspec_src = f"""data {grp_outfile}
ignore **-0.5 4.0-**
ignore bad
cpd /xw
statistic cstat
setpl en
abun wilm
"""
with open(xcm_src, "w") as f:
    f.write(xspec_src)
```
