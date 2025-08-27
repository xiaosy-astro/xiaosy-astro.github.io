### IXPE在其任务的前2.5年中观测到的所有黑洞X射线双星

<img width="482" height="244" alt="Image" src="https://github.com/user-attachments/assets/c5edc004-49e3-4fa9-802e-12a7d58ccb49" />

原文链接 [IXPE View of BH XRBs during the First 2.5 Years of the Mission](https://www.mdpi.com/2075-4434/12/5/54)

### Python将该图复现

```python
import numpy as np
from astropy.coordinates import SkyCoord
import astropy.units as u
from dustmaps.sfd import SFDQuery
import matplotlib.pyplot as plt
plt.style.use('https://raw.githubusercontent.com/wangboting/python-style/main/pythonstyle.style')
plt.rcParams['axes.unicode_minus'] = False
plt.rcParams['font.size'] = 12
plt.rcParams['font.family'] = 'Times New Roman'

from dustmaps.sfd import SFDQuery



# 初始化 SFD
sfd = SFDQuery()

# 在银道坐标系中采样
l = np.linspace(-180, 180, 1000)
b = np.linspace(-90, 90, 500)
ll, bb = np.meshgrid(l, b)
coords = SkyCoord(l=-ll*u.deg, b=bb*u.deg, frame='galactic')
ebv = sfd(coords)   # 得到 E(B−V) 值

# 画 Mollweide 投影
plt.figure(figsize=(10,5))
ax = plt.subplot(111, projection="mollweide")
im = ax.pcolormesh(np.radians((ll)), np.radians(bb), ebv,
                   shading='auto', cmap='Blues', vmin=0, vmax=2,alpha=0.9)

plt.colorbar(im, orientation='vertical', pad=0.05, label="E(B-V)",location="right",extend='max')
ax.tick_params(axis='both', colors='red')
xtick_locs = np.radians([-150, -120, -90, -60, -30, 0, 30, 60, 90, 120, 150])  # Radians for plotting
xtick_labels = ['150°', '120°', '90°', '60°', '30°', '0°', '330°', '300°', '270°', '240°', '210°']
ax.set_xticks(xtick_locs)
ax.set_xticklabels(xtick_labels)
ax.set_xlabel('l [deg]')
ax.set_ylabel('b [deg]')
ax.grid(True)
def wrap_l(l):
    return 360-l if l > 180 else -l
# 定义源的信息：名字、l、b、颜色
sources = [
    ("Cyg X-1",        71.3349982655144,  +3.0668346317201, "orange"),
    ("Cyg X-3",        79.84549,          +0.70006,         "blue"),
    ("LMC X-1",       280.2030024388124, -31.5158204428701, "green"),
    ("LMC X-3",       273.5764427008667, -32.0814329516362, "red"),
    ("4U 1957+115",    51.3076815317130,  -9.3301753261750, "purple"),
    ("SS 433",         39.6941044854067,  -2.2445979222389, "yellow"),
    ("Swift J1727.8-1613",  8.64165916,  +10.2553516,       "pink"),
    ("GX 339-4",      338.9391581007216,  -4.3264729352953, "brown"),
    ("4U 1630-47",    336.911278,         +0.250229,        "skyblue"),
    ("Swift J151857.0–572147", 321.815571, -0.00253675,     "cyan"),
]

# 统一绘制
for name, l, b, color in sources:
    ax.scatter(
        np.radians(wrap_l(l)), np.radians(b),
        color=color, marker='*', s=150,
        edgecolors='black', linewidths=0.8,
        label=name
    )

# 图例
ax.legend(
    loc='upper center',
    bbox_to_anchor=(0.5, -0.1),  # 横向居中，纵向往下挪
    fontsize='small',
    ncol=4,
    frameon=False
)
plt.savefig('All_BH_XRBs_observed_by_IXPE.pdf',dpi=400,bbox_inches="tight")
plt.show()
```

### 结果

<img width="723" height="427" alt="Image" src="https://github.com/user-attachments/assets/c913c5ed-1b24-499a-ad5c-40770c8a9161" />

### 最新结果

<img width="723" height="430" alt="Image" src="https://github.com/user-attachments/assets/393a5a54-96b0-4f4a-8b00-de04bc1214c4" />