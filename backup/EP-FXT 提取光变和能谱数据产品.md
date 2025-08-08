> [!IMPORTANT]
> 本教程是在给定坐标下， 从EP/FXT的用户数据中，提取其能谱和光变曲线产品，并自动生成两个能谱拟合xcm文件，一个是联合拟合，一个是单个拟合，只需要输入拟合的模型即可。本教程适用于FF 和 PW 观测模式的产品提取。

## 1. 定义region文件函数 

```python
import re
import glob
import os
import shutil
import pandas as pd
import subprocess
def write_ds9_circle_region(ra, dec, output_dir='.', filename='src.reg',
                              radius='60"',comment='src'):
    """
    写入 DS9 circle region 文件

    参数:
    - ra, dec: 坐标（浮点型，单位为度）
    - output_dir: 输出目录（默认为当前目录）
    - filename: 文件名（默认为 src.reg）
    - radius: 源区域的半径（字符串，带单位，如 "60.000"）

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

def write_ds9_annulus_region(ra, dec, output_dir='.', filename='bkg.reg',
                              inner_radius='120.000"', outer_radius='282.000"',comment='bkg'):
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
```


## 2. 解析文件名获取obsid

```python
def parse_filename(filename: str) -> str:
    """
    从标准文件名中解析 obsid
    """
    pattern = r'ep_fxt_(\d+)_FXTAB_(\d+)'
    match = re.match(pattern, filename)
    if not match:
        raise ValueError(f"文件名 '{filename}' 格式不符合 'ep_fxt..._FXTAB...' 的预期格式！")
    obsid = match.group(1)
    return obsid

# === 示例调用 ===
event_name = 'SAXJ1808.4-3658'  # 事件名称
ra_src, dec_src = 272.114750,-36.9789721
ep_fxt_filename = 'ep_fxt_06800000810_FXTAB_02'  # 数据目录
obsid = parse_filename(ep_fxt_filename)
print(event_name, ra_src, dec_src, ep_fxt_filename, obsid)
```

## 3. 重命名att，orb，hk，mkf，clean event的文件名

```python
auil_work_dir = f'/mnt/fxtdata/{event_name}/{ep_fxt_filename}/auxil'
# ==== 搜索 fits 文件 ====
auil_fits_files = glob.glob(os.path.join(auil_work_dir, '*.fits'))

# ==== 根据文件名包含关键词判断 ====
#### 重命名att,orb文件名
for file_path in auil_fits_files:
    file_name = os.path.basename(file_path)
    if 'att' in file_name:
        new_name = 'att.fits'
    elif 'orb' in file_name:
        new_name = 'orb.fits'
    else:
        continue  # 不符合跳过
    new_path = os.path.join(auil_work_dir, new_name)
    if os.path.exists(new_path):
        print(f'Target file already exists: {new_name}, skipping.')
    else:
        shutil.move(file_path, new_path)
        print(f'Renamed {file_name} to {new_name}')

#### 重命名hk,mkf文件名

hk_work_dir = f'/mnt/fxtdata/{event_name}/{ep_fxt_filename}/fxt/hk'

hk_fits_files = glob.glob(os.path.join(hk_work_dir, '*.fits'))

for hk_file_path in hk_fits_files:
    hk_file_name = os.path.basename(hk_file_path)
    if 'hk' in hk_file_name:
        hk_new_name = 'hk.fits'
    elif 'mkf' in hk_file_name:
        hk_new_name = 'mkf.fits'
    else:
        continue  # 不符合跳过
    hk_new_path = os.path.join(hk_work_dir, hk_new_name)
    if os.path.exists(hk_new_path):
        print(f'Target file already exists: {hk_new_name}, skipping.')
    else:
        shutil.move(hk_file_path, hk_new_path)
        print(f'Renamed {hk_file_name} to {hk_new_name}')

### 重命名clean event file

clean_evt_file_dir = f'/mnt/fxtdata/{event_name}/{ep_fxt_filename}/fxt/products'
clean_evt_fits_file = glob.glob(os.path.join(clean_evt_file_dir, '*.fits'))

event_files_info_a = []
event_files_info_b = []

for evt_file_path in clean_evt_fits_file:
    evt_file_name = os.path.basename(evt_file_path)

    match = re.match(r'fxt_([ab])_' + obsid + r'_([a-zA-Z]+).*\.fits', evt_file_name)
    if not match:
        continue

    module = match.group(1)
    mode = match.group(2)
    evt_new_name = f'fxt_{module}_{obsid}_{mode}_po_cl.fits'
    evt_new_path = os.path.join(clean_evt_file_dir, evt_new_name)

    if os.path.exists(evt_new_path):
        print(f'Target file already exists: {evt_new_name}, skipping.')
    else:
        shutil.move(evt_file_path, evt_new_path)
        print(f'Renamed {evt_file_name} to {evt_new_name}')

    # 准备要记录的信息字典
    file_info = {
        'module': module,
        'mode': mode,
        'old_name': evt_file_name,
        'new_name': evt_new_name,
        'path': evt_new_path
    }
    # 2. 根据 module 的值，将信息字典追加到对应的列表中
    if module == 'a':
        event_files_info_a.append(file_info)
    elif module == 'b':
        event_files_info_b.append(file_info)

evt_new_names_a_list = [info['new_name'] for info in event_files_info_a]
evt_new_names_b_list = [info['new_name'] for info in event_files_info_b]
# 2. 初始化变量以存储最终的文件名字符串
evt_new_name_a = None
evt_new_name_b = None

# 3. 从列表中提取文件名
#    我们添加一个检查，确保列表不是空的，以避免 IndexError
if evt_new_names_a_list:
    evt_new_name_a = evt_new_names_a_list[0]  # 提取列表中的第一个元素

if evt_new_names_b_list:
    evt_new_name_b = evt_new_names_b_list[0]  # 提取列表中的第一个元素

# 现在，evt_new_name_a 和 evt_new_name_b 就是您想要的字符串格式
print("\n--- Desired Filename Strings ---")
print(f"Module 'a' file: {evt_new_name_a}")
print(f"Module 'b' file: {evt_new_name_b}")
```

## 4. 生成bkg.reg和src.reg

```python
clean_evt_file_dir = f'/mnt/fxtdata/{event_name}/{ep_fxt_filename}/fxt/products'
comment_src = f'{event_name}'
src_reg = f'src.reg'

src_reg_path = write_ds9_circle_region(ra_src, dec_src, output_dir=clean_evt_file_dir, filename=src_reg,comment=comment_src)
print(f'Region file saved to: {src_reg_path}')

# 手动修改bkg坐标
comment_bkg = f'bkg'
bkg_reg = f'bkg.reg'
bkg_reg_path = write_ds9_annulus_region(ra_src, dec_src, output_dir=clean_evt_file_dir, filename=bkg_reg,comment=comment_bkg)
print(f'Region file saved to: {bkg_reg_path}')
```

## 5.  提取图像，能谱和光变

### 5.1 FXT-A的图像，能谱和光变
```python
### fxt-a
module_1 = 'a'
module_2 = 'b'
image_a = f'fxt_{module_1}.img'
src_spec_a = f'fxt_{module_1}_src.pha'
bkg_spec_a = f'fxt_{module_1}_bkg.pha'
src_lc_a = f'fxt_{module_1}_src.lc'
bkg_lc_a = f'fxt_{module_1}_bkg.lc'


xco_extract_img_a = f'xselect_{module_1}_extract_img.xco'
xco_extract_img_filepath_a = os.path.join(clean_evt_file_dir, xco_extract_img_a)


xco_extract_lc_spec_a = f'xselect_{module_1}_extract_lc_spec.xco'
xco_extract_lc_spec_filepath_a = os.path.join(clean_evt_file_dir, xco_extract_lc_spec_a)

xselect_img_a = f"""img
yes
set mission ep
set datadir {clean_evt_file_dir}
read event {evt_new_name_a}
extract image
save image {image_a} clobber=yes
exit
no
"""

# ==== 保存到指定目录 ==== img
with open(xco_extract_img_filepath_a, "w") as f:
    f.write(xselect_img_a)
print(f"Saved XSELECT command file: {xco_extract_img_filepath_a}")



xselect_fxt_a = f"""src
yes
set mission ep
set datadir {clean_evt_file_dir}
read event {evt_new_name_a}
.
filter region {src_reg}
extract spectrum
save spectrum {src_spec_a} clobber=yes
clear region
filter region {bkg_reg}
extract spectrum
save spectrum {bkg_spec_a} clobber=yes
clear region
filter region {src_reg}
filter pha_cutoff 73 925
extract curve
save curve {src_lc_a} clobber=yes
clear region
filter region {bkg_reg}
extract curve
save curve {bkg_lc_a} clobber=yes
exit
no
"""
# ==== 保存到指定目录 ====
with open(xco_extract_lc_spec_filepath_a, "w") as f:
    f.write(xselect_fxt_a)
print(f"Saved XSELECT command file: {xco_extract_lc_spec_filepath_a}")


#### 检查如果有输出同名文件，删除



if os.path.exists(src_spec_a):
    os.remove(src_spec_a)
    print(f"Deleted source pha file: {src_spec_a}")

if os.path.exists(bkg_spec_a):
    os.remove(bkg_spec_a)
    print(f"Deleted background pha file: {bkg_spec_a}")

if os.path.exists(src_lc_a):
    os.remove(src_lc_a)
    print(f"Deleted source lc file: {src_lc_a}")

if os.path.exists(bkg_lc_a):
    os.remove(bkg_lc_a)
    print(f"Deleted background lc file: {bkg_lc_a}")

if os.path.exists(image_a):
    os.remove(image_a)
    print(f"Deleted background lc file: {image_a}")
```

### 5.2 FXT-B的图像，能谱和光变

```python
### fxt-b

image_b = f'fxt_{module_2}.img'
src_spec_b = f'fxt_{module_2}_src.pha'
bkg_spec_b = f'fxt_{module_2}_bkg.pha'
src_lc_b = f'fxt_{module_2}_src.lc'
bkg_lc_b = f'fxt_{module_2}_bkg.lc'

xco_extract_img_b = f'xselect_{module_2}_extract_img.xco'
xco_extract_img_filepath_b = os.path.join(clean_evt_file_dir, xco_extract_img_b)

xco_extract_lc_spec_b = f'xselect_{module_2}_extract_lc_spec.xco'
xco_extract_lc_spec_filepath_b = os.path.join(clean_evt_file_dir, xco_extract_lc_spec_b)

xselect_img_b = f"""img
yes
set mission ep
set datadir {clean_evt_file_dir}
read event {evt_new_name_b}
extract image
save image {image_b} clobber=yes
exit
no
"""

# ==== 保存到指定目录 ==== img
with open(xco_extract_img_filepath_b, "w") as f:
    f.write(xselect_img_b)
print(f"Saved XSELECT command file: {xco_extract_img_filepath_b}")

xselect_fxt_b = f"""src
yes
set mission ep
set datadir {clean_evt_file_dir}
read event {evt_new_name_b}
.
filter region {src_reg}
extract spectrum
save spectrum {src_spec_b} clobber=yes
clear region
filter region {bkg_reg}
extract spectrum
save spectrum {bkg_spec_b} clobber=yes
clear region
filter region {src_reg}
filter pha_cutoff 73 925
extract curve
save curve {src_lc_b} clobber=yes
clear region
filter region {bkg_reg}
extract curve
save curve {bkg_lc_b} clobber=yes
exit
no
"""

# ==== 保存到指定目录 ====
with open(xco_extract_lc_spec_filepath_b, "w") as f:
    f.write(xselect_fxt_b)
print(f"Saved XSELECT command file: {xco_extract_lc_spec_filepath_b}")

# ==== 检查并删除旧输出文件 ====
if os.path.exists(src_spec_b):
    os.remove(src_spec_b)
    print(f"Deleted source pha file: {src_spec_b}")

if os.path.exists(bkg_spec_b):
    os.remove(bkg_spec_b)
    print(f"Deleted background pha file: {bkg_spec_b}")

if os.path.exists(src_lc_b):
    os.remove(src_lc_b)
    print(f"Deleted source lc file: {src_lc_b}")

if os.path.exists(bkg_lc_b):
    os.remove(bkg_lc_b)
    print(f"Deleted background lc file: {bkg_lc_b}")

if os.path.exists(image_b):
    os.remove(image_b)
    print(f"Deleted image file: {image_b}")
```

### 5.1 运行

```python
### 先运行提取图像的xco fxt-a
cmd_img_a = f"xselect @{os.path.abspath(os.path.join(clean_evt_file_dir, xco_extract_img_a))}"
subprocess.run(cmd_img_a, shell=True, check=True,
        text=True,cwd=clean_evt_file_dir)


cmd_lc_spec_a = f"xselect  @{os.path.abspath(os.path.join(clean_evt_file_dir, xco_extract_lc_spec_a))}"
subprocess.run(cmd_lc_spec_a, shell=True, check=True,
        text=True,cwd=clean_evt_file_dir)

### 先运行提取图像的xco fxt-b
cmd_img_b = f"xselect @{os.path.abspath(os.path.join(clean_evt_file_dir, xco_extract_img_b))}"
subprocess.run(cmd_img_b, shell=True, check=True,
        text=True,cwd=clean_evt_file_dir)


cmd_lc_spec_b = f"xselect  @{os.path.abspath(os.path.join(clean_evt_file_dir, xco_extract_lc_spec_b))}"
subprocess.run(cmd_lc_spec_b, shell=True, check=True,
        text=True,cwd=clean_evt_file_dir)

print('提取图像，能谱和光变成功')
```

## 6. fxtdas生成arf和rmf文件
### 6.1 生成FXT-A的arf和rmf文件

```python
### fxt-a
expo_map_a = f'fxt_{module_1}_expo.fits'
expogen_cmd_a = f'fxtexpogen mkffile="{hk_work_dir}/mkf.fits" '\
            f'evtfile="{clean_evt_file_dir}/{evt_new_name_a}" '\
            f'outfile="{clean_evt_file_dir}/{expo_map_a}" clobber=yes'

try:
    result = subprocess.run(
        expogen_cmd_a,
        check=True,
        capture_output=True,
        text=True,
        shell=True
    )
    print("fxtexpogen 输出:", result.stdout)
    print(f"已保存在: {clean_evt_file_dir}/{expo_map_a}")
except subprocess.CalledProcessError as e:
    print("执行 fxtexpogen 时出错：")
    print("命令:", e.cmd)
    print("返回码:", e.returncode)
    print("输出:", e.stdout)
    print("错误信息:", e.stderr)

## 2. fxtarfgen 
# specfile=source.pha expfile=fxta-expo.fits psfcor=1 outfile=source.arf

arf_a = f'fxt_{module_1}.arf'
arfgen_cmd_a = (
    f'fxtarfgen '
    f'specfile="{clean_evt_file_dir}/{src_spec_a}" '
    f'expfile="{clean_evt_file_dir}/{expo_map_a}" '
    f'psfcor=1 '
    f'outfile="{clean_evt_file_dir}/{arf_a}" '
    f'clobber=yes'
)
try:
    result = subprocess.run(
        arfgen_cmd_a,
        check=True,
        capture_output=True,
        text=True,
        shell=True
    )
    print("fxtarfgen 输出:", result.stdout)
    print(f"已保存在: {clean_evt_file_dir}/{arf_a}")
except subprocess.CalledProcessError as e:
    print("执行 fxtarfgen 时出错：")
    print("命令:", e.cmd)
    print("返回码:", e.returncode)
    print("输出:", e.stdout)
    print("错误信息:", e.stderr)

### 3. fxtrmfgen 
# specfile=source.pha outfile=source.rmf

rmf_a = f'fxt_{module_1}.rmf'
rmfgen_cmd_a = (
    f'fxtrmfgen '
    f'specfile="{clean_evt_file_dir}/{src_spec_a}" '
    f'outfile="{clean_evt_file_dir}/{rmf_a}" '
    f'clobber=yes'
)
try:
    result = subprocess.run(
        rmfgen_cmd_a,
        check=True,
        capture_output=True,
        text=True,
        shell=True
    )
    print("fxtrmfgen 输出:", result.stdout)
    print(f"已保存在: {clean_evt_file_dir}/{rmf_a}")
except subprocess.CalledProcessError as e:
    print("执行 fxtrmfgen 时出错：")
    print("命令:", e.cmd)
    print("返回码:", e.returncode)
    print("输出:", e.stdout)
    print("错误信息:", e.stderr)
```

### 6.2 生成FXT-B的arf和rmf文件

```python
### fxt-b
expo_map_b = f'fxt_{module_2}_expo.fits'
expogen_cmd_b = f'fxtexpogen mkffile="{hk_work_dir}/mkf.fits" ' \
                f'evtfile="{clean_evt_file_dir}/{evt_new_name_b}" ' \
                f'outfile="{clean_evt_file_dir}/{expo_map_b}" clobber=yes'

try:
    result = subprocess.run(
        expogen_cmd_b,
        check=True,
        capture_output=True,
        text=True,
        shell=True
    )
    print("fxtexpogen 输出:", result.stdout)
    print(f"已保存在: {clean_evt_file_dir}/{expo_map_b}")
except subprocess.CalledProcessError as e:
    print("执行 fxtexpogen 时出错：")
    print("命令:", e.cmd)
    print("返回码:", e.returncode)
    print("输出:", e.stdout)
    print("错误信息:", e.stderr)

# 2. fxtarfgen
arf_b = f'fxt_{module_2}.arf'
arfgen_cmd_b = (
    f'fxtarfgen '
    f'specfile="{clean_evt_file_dir}/{src_spec_b}" '
    f'expfile="{clean_evt_file_dir}/{expo_map_b}" '
    f'psfcor=1 '
    f'outfile="{clean_evt_file_dir}/{arf_b}" '
    f'clobber=yes'
)
try:
    result = subprocess.run(
        arfgen_cmd_b,
        check=True,
        capture_output=True,
        text=True,
        shell=True
    )
    print("fxtarfgen 输出:", result.stdout)
    print(f"已保存在: {clean_evt_file_dir}/{arf_b}")
except subprocess.CalledProcessError as e:
    print("执行 fxtarfgen 时出错：")
    print("命令:", e.cmd)
    print("返回码:", e.returncode)
    print("输出:", e.stdout)
    print("错误信息:", e.stderr)

# 3. fxtrmfgen
rmf_b = f'fxt_{module_2}.rmf'
rmfgen_cmd_b = (
    f'fxtrmfgen '
    f'specfile="{clean_evt_file_dir}/{src_spec_b}" '
    f'outfile="{clean_evt_file_dir}/{rmf_b}" '
    f'clobber=yes'
)
try:
    result = subprocess.run(
        rmfgen_cmd_b,
        check=True,
        capture_output=True,
        text=True,
        shell=True
    )
    print("fxtrmfgen 输出:", result.stdout)
    print(f"已保存在: {clean_evt_file_dir}/{rmf_b}")
except subprocess.CalledProcessError as e:
    print("执行 fxtrmfgen 时出错：")
    print("命令:", e.cmd)
    print("返回码:", e.returncode)
    print("输出:", e.stdout)
    print("错误信息:", e.stderr)
```

## 7. group 光子数
### 7.1 FXT-A

```python
## fxt-a
grp_outfile_a = f'fxt_{module_1}.grp'
grp_min_bins = 25  # 按需修改

cmd_grp_a = (
    f"grppha "
    f"infile={clean_evt_file_dir}/{src_spec_a} "
    f"outfile={clean_evt_file_dir}/{grp_outfile_a} "
    f"comm='group min {grp_min_bins} "
    f"& chkey backfile {bkg_spec_a} "
    f"& chkey respfile {rmf_a} "
    f"& chkey ancrfile {arf_a} "
    f"& show keywords "
    f"& exit' "
    f"clobber=yes"
)
if os.path.exists(grp_outfile_a):
    os.remove(grp_outfile_a)
    print(f"Deleted existing file: {grp_outfile_a}")

try:
    result = subprocess.run(cmd_grp_a, check=True, capture_output=True, text=True, shell=True)
    print("Command output:", result.stdout)
except subprocess.CalledProcessError as e:
    print("Error occurred:")
    print("Command:", e.cmd)
    print("Return code:", e.returncode)
    print("Output:", e.stdout)
    print("Error:", e.stderr)
```

### 7.2 FXT-B

```python
## fxt-b
grp_outfile_b = f'fxt_{module_2}.grp'
# grp_min_bins = 10  # 按需修改

cmd_grp_b = (
    f"grppha "
    f"infile={clean_evt_file_dir}/{src_spec_b} "
    f"outfile={clean_evt_file_dir}/{grp_outfile_b} "
    f"comm='group min {grp_min_bins} "
    f"& chkey backfile {bkg_spec_b} "
    f"& chkey respfile {rmf_b} "
    f"& chkey ancrfile {arf_b} "
    f"& show keywords "
    f"& exit' "
    f"clobber=yes"
)
if os.path.exists(grp_outfile_b):
    os.remove(grp_outfile_b)
    print(f"Deleted existing file: {grp_outfile_b}")

try:
    result = subprocess.run(cmd_grp_b, check=True, capture_output=True, text=True, shell=True)
    print("Command output:", result.stdout)
except subprocess.CalledProcessError as e:
    print("Error occurred:")
    print("Command:", e.cmd)
    print("Return code:", e.returncode)
    print("Output:", e.stdout)
    print("Error:", e.stderr)
```

##  8. 自动写一个xcm

```python
xspec_joint_fit = f"""data 1:1 {grp_outfile_a}
data 2:2 {grp_outfile_b}
ign 1:**-0.5,10.0-**
ign 2:**-0.5,10.0-**
ignore bad
cpd /xw
setpl en
abun wilm
show rate
"""
with open(f"{clean_evt_file_dir}/xspec_joint_fit_fxta_fxtb.xcm", "w") as f:
    f.write(xspec_joint_fit)



###  xcm
xspec_single_fit = f"""data {grp_outfile_a}
ign **-0.5,10.0-**
ignore bad
cpd /xw
setpl en
abun wilm
show rate
"""
with open(f"{clean_evt_file_dir}/xspec_single_fit.xcm", "w") as f:
    f.write(xspec_single_fit)
```
