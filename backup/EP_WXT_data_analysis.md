目标：流水线产品不可用，我需要重新提取数据，每一次观测数据含有event文件，需要通过xselect软件从event中提取数据产品，数据产品包括能谱（包括源和背景能谱）和光变曲线（源和背景的光变曲线）。

思路：
- 1. 我知道目标源的坐标，可以生成源和背景的区域文件（src1.reg和bkg1.reg）；
- 2. 使用xselect读取event文件，导入reg提取能谱和光变曲线。

目的：`处理值班认证的2-3级数据`

产品：
- 1. 自动使用`lcurve`画出净光变曲线，未扣除背景的光变曲线和背景的光变曲线3*1面板。
- 2. 自动使用`ELISA`拟合能谱，默认使用tbabs()*powerlaw()
- 3. 如果`ELISA`拟合结果较差，自动生成了.xcm文档，可以进入`Xspec`拟合

### <span style="color:red">先进入`elisa`的Python虚拟环境</span>

[`ELISA`安装教程](https://astro-elisa.readthedocs.io/en/stable/installation.html)

### 定义生成region文件

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
