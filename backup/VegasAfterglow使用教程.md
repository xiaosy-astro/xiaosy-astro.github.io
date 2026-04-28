版本 ：V1.0.0 

### 1. 查看模型有哪些物理参数

```python
import VegasAfterglow
dir(VegasAfterglow.ModelParams)
```

输出结果

```python
['A_star',
 'E_iso',
 'E_iso_w',
 'Gamma0',
 'Gamma0_w',
 'L0',
 '__class__',
 '__delattr__',
 '__dir__',
 '__doc__',
 '__eq__',
 '__format__',
 '__ge__',
 '__getattribute__',
 '__getstate__',
 '__gt__',
 '__hash__',
 '__init__',
 '__init_subclass__',
 '__le__',
 '__lt__',
 '__module__',
 '__ne__',
 '__new__',
 '__reduce__',
 '__reduce_ex__',
 '__repr__',
 '__setattr__',
 '__sizeof__',
 '__str__',
 '__subclasshook__',
 '_pybind11_conduit_v1_',
 'eps_B',
 'eps_B_r',
 'eps_e',
 'eps_e_r',
 'k_e',
 'k_g',
 'n0',
 'n_ism',
 'p',
 'p_r',
 'q',
 't0',
 'tau',
 'theta_c',
 'theta_v',
 'theta_w',
 'xi_e',
 'xi_e_r']
```

### 2. 查看模型有哪些配置

```python
from VegasAfterglow import Setups
cfg = Setups()
dir(cfg)
```

输出结果

```python
['IC_cooling',
 'KN',
 '__class__',
 '__delattr__',
 '__dir__',
 '__doc__',
 '__eq__',
 '__format__',
 '__ge__',
 '__getattribute__',
 '__getstate__',
 '__gt__',
 '__hash__',
 '__init__',
 '__init_subclass__',
 '__le__',
 '__lt__',
 '__module__',
 '__ne__',
 '__new__',
 '__reduce__',
 '__reduce_ex__',
 '__repr__',
 '__setattr__',
 '__sizeof__',
 '__str__',
 '__subclasshook__',
 '_pybind11_conduit_v1_',
 'fwd_SSC',
 'jet',
 'lumi_dist',
 'magnetar',
 'medium',
 'phi_resol',
 'rtol',
 'rvs_SSC',
 'rvs_shock',
 't_resol',
 'theta_resol',
 'z']
```

### 3. 查询拟合器

```python
from VegasAfterglow import Fitter
fitter = Fitter(data, cfg)
help(fitter.fit)
```

输出结果

```python
Help on method fit in module VegasAfterglow.runner:

fit(param_defs: Sequence[VegasAfterglow.types.ParamDef], resolution: Tuple[float, float, float] = (0.3, 1, 10), total_steps: int = 10000, burn_frac: float = 0.3, thin: int = 1, top_k: int = 10) -> VegasAfterglow.types.FitResult method of VegasAfterglow.runner.Fitter instance
    Run the MCMC sampler.

    Parameters
    ----------
    param_bounds :
        A sequence of (name, init, lower, upper) for each free parameter.
    resolution :
        (t_grid, theta_grid, phi_grid) for the coarse MCMC stage.
    total_steps :
        Total number of MCMC steps.
    burn_frac :
        Fraction of steps to discard as burn-in.
    thin :
        Thinning factor for the returned chain.
    top_k :
        Number of top fits to save in the result.

    Returns
    -------
    FitResult
```

