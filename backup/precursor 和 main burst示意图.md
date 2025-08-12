代码：

```python
import numpy as np
import matplotlib.pyplot as plt
plt.style.use('https://raw.githubusercontent.com/wangboting/python-style/main/pythonstyle.style')
plt.rcParams['font.sans-serif'] = ['SimSun']
plt.rcParams['axes.unicode_minus'] = False
plt.rcParams['font.size'] = 12
plt.rcParams['font.family'] = 'Times New Roman'

# 时间轴
t = np.linspace(-50, 100, 500)

# 背景噪声
np.random.seed(42)
background = np.random.normal(0, 0.1, len(t))

# 模拟前驱辐射 (precursor)
prec_peak_time = 0
prec_width = 3
prec_amplitude = 1.2
prec_signal = prec_amplitude * np.exp(-(t - prec_peak_time)**2 / (2 * prec_width**2))

# 模拟主暴发 (main burst)
main_peak_time = 80
main_width = 6
main_amplitude = 3.5
main_signal = main_amplitude * np.exp(-(t - main_peak_time)**2 / (2 * main_width**2))

# 总光变曲线
counts = background + prec_signal + main_signal

# 关键参数（用于标注）
R_prec = prec_amplitude
R_main = main_amplitude
tau_prec = 2 * prec_width
tau_main = 2 * main_width

t_start_prec = prec_peak_time - prec_width
t_end_prec   = prec_peak_time + prec_width
t_start_main = main_peak_time - main_width
t_end_main   = main_peak_time + main_width

dt_pk  = main_peak_time - prec_peak_time
dt_det = t_start_main - t_end_prec

# 画图（阶梯状）
plt.figure(figsize=(7, 5))
plt.step(t, counts, color='black', lw=1, where='mid')

# 新的竖线位置
vline_prec_start = t_start_prec - 4
vline_prec_end   = t_end_prec + 4
vline_main_start = t_start_main - 9
vline_main_end   = t_end_main + 5

# 竖虚线：前驱 / 主峰的 start/end
# 竖虚线
for x in (vline_prec_start, vline_prec_end, vline_main_start, vline_main_end):
    plt.axvline(x, ls='--', color='gray', lw=0.9)


for x in (prec_peak_time, main_peak_time):
    plt.axvline(x, ls='--', color='blue', lw=0.3)
# 水平虚线：R_prec / R_main
plt.hlines(R_prec+0.2, t.min(), prec_peak_time, colors='red', linestyles='--', lw=0.9)
plt.hlines(R_main+0.15, main_peak_time, t.max(), colors='red', linestyles='--', lw=0.9)

# ylim 便于放置标注（与原图类似的上下边距）
ymin = -1.0
ymax = R_main + 1.0
plt.ylim(ymin, ymax)

# --- τ 标注：用弧形双箭头放在曲线下方 ---
y_tau = ymin + 0.25  # 放在下方（与图中弧形类似）
# precursor duration arc
plt.annotate(
    "", xy=(vline_prec_start, y_tau), xytext=(vline_prec_end, y_tau),
    arrowprops=dict(arrowstyle='<->', lw=1.2, connectionstyle="arc3,rad=-0.3")
)
plt.text((vline_prec_start + vline_prec_end)/2, y_tau+0.1, r'$\tau_{\rm prec}$', ha='center', va='top')

plt.annotate(
    "", xy=(vline_main_start, y_tau), xytext=(vline_main_end, y_tau),
    arrowprops=dict(arrowstyle='<->', lw=1.2, connectionstyle="arc3,rad=-0.3")
)
plt.text((vline_main_start + vline_main_end)/2, y_tau+0.1, r'$\tau_{\rm main}$', ha='center', va='top')

# --- Δt_pk 标注：峰与峰之间的双箭头（放在上方） ---
y_pk = R_main + 0.35
plt.annotate(
    "", xy=(prec_peak_time, y_pk), xytext=(main_peak_time, y_pk),
    arrowprops=dict(arrowstyle='<->', lw=1.2,color='blue')
)
plt.text((prec_peak_time + main_peak_time)/2, y_pk + 0.04, r'$\Delta t_{\rm pk}$', ha='center', va='bottom',color='blue')

# --- Δt_det 标注：前驱结束与主峰开始之间的双箭头（放在 Δt_pk 之下） ---
y_det = R_main + 0.15
plt.annotate(
    "", xy=(vline_prec_end, y_det), xytext=(vline_main_start, y_det),
    arrowprops=dict(arrowstyle='<->', lw=1.2)
)
plt.text((vline_prec_end + vline_main_start)/2, y_det - 0.3, r'$\Delta t_{\rm q}$', ha='center', va='bottom')
# 标注 R_prec / R_main 的文字
plt.text(t.min() + 2, R_prec + 0.2, r'$R_{\rm prec}$', va='bottom', color='red')
plt.text(t.max() + 1, R_main + 0.2, r'$R_{\rm main}$', va='bottom', ha='right', color='red')

# 轴与外观
plt.xlabel("Time (s)")
plt.ylabel(r"Counts s$^{-1}$")
plt.yticks([])
plt.xticks([])
plt.tight_layout()
plt.savefig('lightcurve_prec_main_demo.pdf', dpi=400, bbox_inches="tight") 
plt.show()
```

![Image](https://github.com/user-attachments/assets/e1965b02-397c-420f-a695-6118a99b6a48)

- $R_{\mathrm{prec}}$: peak count rate of the first episode of emission  
- $R_{\mathrm{main}}$: peak count rate of the main episode of emission  
- $\tau_{\mathrm{prec}}$: duration of the precursor episode emission  
- $\tau_{\mathrm{main}}$: duration of the main episode emission  
- $\Delta t_{\mathrm{pk}}$: the separation time between the $R_{\mathrm{prec}}$ and $R_{\mathrm{main}}$  
- $\Delta t_q$: the quiescent time  
