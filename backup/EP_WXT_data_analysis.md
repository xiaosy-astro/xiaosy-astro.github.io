# EP_WXT_data_analysis

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
