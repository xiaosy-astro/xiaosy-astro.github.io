### 打印能量

```python
ECLAIRs_pluse1 = Data(
    name=f"{event_name}_pluse1",
    erange=[4.0, 20.0],
    specfile=f'./{folder_name}/GRB250419A_PEO_1er_episode.grp',
    respfile=f'./{folder_name}/GRB250419A_PEO_rmf_1er_episode.fits',
    ancrfile=f'./{folder_name}/ECL-RSP-ARF_20220515T01.fits',
    group='opt'
)
print(ECLAIRs_pluse1.channel_emean)
```

### 计算能量流量
```
import numpy as np
Emax = round(max(ECLAIRs_pluse1.channel_emean))
print(f"Emax={Emax}")
Flux = posterior.flux(emin=4,emax=Emax,cl=1.,energy=True,hdi=False,comps=False)
median_flux = Flux.median['GRB 250419A_pluse1'].value
lower_flux, upper_flux = Flux.intervals['GRB 250419A_pluse1'][0].value, Flux.intervals['GRB 250419A_pluse1'][1].value
exponent = np.floor(np.log10(abs(median_flux)))
print(f'{median_flux/10**(exponent):.2f} \
      (+{(upper_flux - median_flux)/10**(exponent):.2f}, \
    {(lower_flux-median_flux)/10**(exponent):.2f}) e{exponent:.0f} erg/cm^2/s')
```