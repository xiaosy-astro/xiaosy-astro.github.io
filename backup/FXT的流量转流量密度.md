### 定义流量转流量密度的计算的函数

```python
import os
import numpy as np
import pandas as pd
from astropy.io import fits
from astropy.time import Time
import pprint
def compute_integral_energy(beta, E1, E2):
    if beta == 1:
        return np.log(E2 / E1)
    else:
        return (E2**(1 - beta) - E1**(1 - beta)) / (1 - beta)

def compute_energy_flux_density_and_error(F, sigma_F, beta, sigma_beta, E1, E2, E_obs):
    integral = compute_integral_energy(beta, E1, E2)
    e_term = E_obs ** (-beta)
    f_E = F / integral * e_term
    df_dF = e_term / integral
    df_dbeta = -np.log(E_obs) * F / integral * e_term
    sigma_f_E = np.sqrt((df_dF * sigma_F)**2 + (df_dbeta * sigma_beta)**2)
    return f_E, sigma_f_E  # 单位：erg/cm²/s/keV

def compute_xray_energy_flux_density(
    input_fits,
    output_file, t0_iso,
    beta_X=1.9,
    sigma_beta_X=0.2,
    E1=0.3,  # keV
    E2=10.0,  # keV
    E_obs=1.0,  # keV
    rate_to_flux=1e-11
):
    """  
        从 Swift-XRT 光变 FITS 计算 X-ray flux density（单位 mJy）及误差，并保存到文件。
    
    参数：
    ----------
    input_fits : str
        光变数据的 FITS 文件路径
    output_file : str
        输出文件路径
    t0_iso : str
        爆发 T0 的 UTC 时间（ISO 格式），如 '2020-01-01T00:00:00'
    beta_X : float
        X 射线谱指数 beta_X
    sigma_beta_X : float
        beta_X 的误差
    nu1, nu2 : float
        积分上下限频率 (Hz)
    nu_obs : float
        要计算 flux density 的频率 (Hz)
    rate_to_flux : float
        计数率到物理 flux 的转换因子（erg/cm^2/s）
    """
    # === Step 1: 读取 FITS 数据和头部 ===
    with fits.open(input_fits) as hdul:
        data = hdul[1].data
        header = hdul[1].header
        time = data['TIME']
        time_err = data['TIME_ERR']
        rate = data['RATE']
        rate_err = data['ERROR']

    # === Step 2: 计算 DATE-OBS 与 T0 差值（秒） ===
    mjd_obs = header.get('MJD-OBS')
    if mjd_obs is None:
        raise ValueError("FITS header 中缺少 'MJD-OBS' 关键字")

    t0 = Time(t0_iso, format='isot', scale='utc')
    mjd_t0 = t0.mjd
    delta_seconds = (mjd_obs - mjd_t0) * 86400.0

    # === Step 3: 相对 T0 的时间 ===
    time_from_t0 = delta_seconds + time

    # === Step 4: 筛选误差较小的点 ===
    valid_mask = rate_err < (rate / 2.0)

    # 初始化结果数组为 NaN
    N = len(rate)
    flux = np.full(N, np.nan)
    flux_err = np.full(N, np.nan)
    flux_density = np.full(N, np.nan)
    flux_density_err = np.full(N, np.nan)

    # === Step 5: 计算 flux 和 flux density ===
    if np.any(valid_mask):
        flux[valid_mask] = rate[valid_mask] * rate_to_flux
        flux_err[valid_mask] = rate_err[valid_mask] * rate_to_flux
        for i in np.where(valid_mask)[0]:
            f_E, f_E_err = compute_energy_flux_density_and_error(
                flux[i], flux_err[i],
                beta=beta_X,
                sigma_beta=sigma_beta_X,
                E1=E1, E2=E2, E_obs=E_obs
            )
            flux_density[i] = f_E
            flux_density_err[i] = f_E_err

    # === 保存结果 ===
    # === Step 6: 构建 DataFrame ===
    df = pd.DataFrame({
        'Time': time_from_t0,
        'Time_err': time_err,
        'Rate': rate,
        'Rate_err': rate_err,
        'flux': flux,
        'flux_err': flux_err,
        'flux_density_E': flux_density,
        'flux_density_E_err': flux_density_err,
    })

    # === Step 7: 计算 mJy 并保存 ===
    conversion_factor = 1e26 / 241797944177033445  # 将 erg/cm²/s/keV 转为 mJy
    df['flux_density_mJy'] = df['flux_density_E'] * conversion_factor
    df['flux_density_mJy_err'] = df['flux_density_E_err'] * conversion_factor

    df.to_csv(output_file, sep='\t', index=False, float_format='%.5e')
    print(f"计算完成，已保存到 {output_file}（剔除高误差点）")
```

### 举例

```python
event_name = 'GRB250419A'  # 或根据每个 obsid 自定义路径
output_dir = f"../{event_name}"
os.makedirs(output_dir, exist_ok=True)

obsid_config = {
    '06800000548': {
        't0': '2025-04-19T02:29:32',
        'rate_to_flux': 2.6718611987381705e-11,
        'photon_index': 1.76,
        'photon_index_err': 0.2
    }
}
output_dict_file = os.path.join(output_dir, "obsid_config_dict.txt")

# 使用 pprint 保持结构清晰
with open(output_dict_file, "w") as f:
    f.write("obsid_config = ")
    pprint.pprint(obsid_config, stream=f, width=120)

print(f"✅ obsid_config 字典已保存为 TXT 文件：{output_dict_file}")

# 3. 遍历每个 obsid，进行计算

all_dfs = []
for obsid, config in obsid_config.items():
    input_file_fxt = os.path.join(output_dir, f'{obsid}_net_lightcurve_a.fits')
    output_file_fxt = os.path.join(output_dir, f'{obsid}_FXT_lc_flux_with_density.csv')
    
    photon_index = config['photon_index']
    photon_index_err = config['photon_index_err']
    beta_X = photon_index - 1.0
    sigma_beta_X = photon_index_err
    
    # 调用函数
    compute_xray_energy_flux_density(
        input_fits=input_file_fxt,
        output_file=output_file_fxt,
        t0_iso=config['t0'],
        beta_X=beta_X,
        sigma_beta_X=sigma_beta_X,
        E1=0.5,
        E2=10.0,
        E_obs=1.0,
        rate_to_flux=config['rate_to_flux']
    )
        # 加载每个结果 CSV，并添加 obsid 标记列
    df = pd.read_csv(output_file_fxt, sep='\t')
    df['obsid'] = obsid
    all_dfs.append(df)

# 合并所有 DataFrame
df_merged = pd.concat(all_dfs, ignore_index=True)

# 保存合并后的文件
merged_output_file = os.path.join(output_dir, f'{event_name}_merged_FXT_lc_flux_with_density.csv')
df_merged.to_csv(merged_output_file, sep='\t', index=False, float_format='%.5e')
print(f"\n✅ 所有观测结果已合并并保存为：{merged_output_file}")

```