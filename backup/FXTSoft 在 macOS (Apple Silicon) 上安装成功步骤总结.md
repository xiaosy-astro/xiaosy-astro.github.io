环境背景

	•	机器：Apple Silicon（M 系列）
	•	系统：macOS
	•	目标：安装 FXTSoft（x86_64）
	•	已存在：HEASoft（arm64）
	•	解决思路：Rosetta + Intel Homebrew + 环境切换
<html>
<head>

</head>
<body>

<p class="p1">下面是一份**“FXTSoft 在 macOS（Apple Silicon）上成功安装”的精简复盘清单**，按<span class="s1"><b>时间顺序 + 关键坑位</b></span>整理，适合你以后快速查阅或给同组同学参考。</p>
<p class="p2"><span class="s2"><hr></span></p>
<p class="p2"><span class="s2"><h1><b>FXTSoft 在 macOS (Apple Silicon) 上安装成功步骤总结</b></h1></span></p>
<p class="p3"><br></p>
<blockquote style="margin: 0.0px 0.0px 0.0px 15.0px; font: 19.0px 'Helvetica Neue'; color: #0e0e0e"><b>环境背景</b><b></b></blockquote>
<p class="p3"><br></p>
<p class="p2"><span class="s2"><ul><li>
<p class="p1">机器：Apple Silicon（M 系列）</p>
</li><li>
<p class="p1">系统：macOS</p>
</li><li>
<p class="p1"><span class="s1">目标：安装 </span><b>FXTSoft（x86_64）</b><b></b></p>
</li><li>
<p class="p1"><span class="s1">已存在：</span><b>HEASoft（arm64）</b><b></b></p>
</li><li>
<p class="p1"><span class="s1">解决思路：</span><b>Rosetta + Intel Homebrew + 环境切换</b></p>
</li></ul></span></p>
<p class="p2"><span class="s2"><hr></span></p>
<p class="p2"><span class="s2"><h2><b>一、前置条件（必须）</b></h2></span></p>
<p class="p3"><br></p>
<p class="p2"><span class="s2"><h3><b>1️⃣ 安装 Xcode Command Line Tools</b></h3></span></p>


<pre><code>xcode-select --install</code></pre>


<p class="p1"><span class="s1"><h3><b>2️⃣ 安装 Rosetta（x86_64 支持）</b></h3></span></p>


<pre><code>softwareupdate --install-rosetta --agree-to-license</code></pre>


<p class="p1"><span class="s1"><hr></span></p>
<p class="p1"><span class="s1"><h2><b>二、准备 x86_64 编译环境</b></h2></span></p>
<p class="p2"><br></p>
<p class="p1"><span class="s1"><h3><b>3️⃣ 安装 Intel Homebrew（/usr/local）</b></h3></span></p>


<pre><code>arch -x86_64 /bin/bash
/bin/bash -c &quot;$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)&quot;</code></pre>


<p class="p1"><span class="s1"><h3><b>4️⃣ 用 Intel brew 安装 gcc / gfortran</b></h3></span></p>


<pre><code>arch -x86_64 /usr/local/bin/brew install gcc libyaml</code></pre>


<p class="p1"><span class="s1"><hr></span></p>
<p class="p1"><span class="s1"><h2><b>三、关键环境变量（编译前）</b></h2></span></p>
<p class="p2"><br></p>
<blockquote style="margin: 0.0px 0.0px 0.0px 15.0px; font: 19.0px '.AppleSystemUIFont'; color: #0e0e0e">⚠️ <span class="s2"><b>非常重要</b></span>：必须在 x86_64 shell 中操作</blockquote>


<pre><code>arch -x86_64 /bin/bash
export PATH=&quot;/usr/local/bin:/usr/local/sbin:$PATH&quot;
hash -r</code></pre>


<p class="p1"><span class="s1"><h3><b>5️⃣ 解决 gfortran 架构冲突</b></h3></span></p>


<pre><code>export FC=/usr/local/bin/gfortran</code></pre>


<blockquote style="margin: 0.0px 0.0px 0.0px 15.0px; font: 19.0px '.AppleSystemUIFontMonospaced'; color: #0e0e0e"><span class="s1">❌ 不能使用 </span>/opt/homebrew/bin/gfortran-14<span class="s1">（arm64）</span></blockquote>
<p class="p2"><span class="s2"><hr></span></p>
<p class="p2"><span class="s2"><h2><b>四、解决 macOS 链接错误（核心坑）</b></h2></span></p>
<p class="p3"><br></p>
<p class="p2"><span class="s2"><h3><b>6️⃣ 指定 macOS SDK（否则 libSystem 找不到）</b></h3></span></p>


<pre><code>export SDKROOT=&quot;$(xcrun --sdk macosx --show-sdk-path)&quot;

export CFLAGS=&quot;-isysroot $SDKROOT&quot;
export CXXFLAGS=&quot;-isysroot $SDKROOT&quot;
export FCFLAGS=&quot;-isysroot $SDKROOT&quot;
export LDFLAGS=&quot;-Wl,-syslibroot,$SDKROOT&quot;</code></pre>


<p class="p1"><span class="s1"><h3><b>7️⃣ 确保 gfortran 运行库可被链接</b></h3></span></p>


<pre><code>GFLIBDIR=&quot;$(dirname &quot;$(gfortran -print-file-name=libgfortran.dylib)&quot;)&quot;
QMLIBDIR=&quot;$(dirname &quot;$(gfortran -print-file-name=libquadmath.dylib)&quot;)&quot;

export LDFLAGS=&quot;$LDFLAGS -L$GFLIBDIR -Wl,-rpath,$GFLIBDIR&quot;
export LDFLAGS=&quot;$LDFLAGS -L$QMLIBDIR -Wl,-rpath,$QMLIBDIR&quot;</code></pre>


<p class="p1"><span class="s1"><hr></span></p>
<p class="p1"><span class="s1"><h2><b>五、编译 FXTSoft</b></h2></span></p>
<p class="p2"><br></p>
<p class="p1"><span class="s1"><h3><b>8️⃣ 解压源码</b></h3></span></p>


<pre><code>tar zxvf fxtsoft.tar.gz
cd fxtsoftv1.20</code></pre>


<p class="p1"><span class="s1"><h3><b>9️⃣ 清理并配置</b></h3></span></p>


<pre><code>rm -rf BUILD_DIR/*
cd BUILD_DIR

./configure --prefix=/Users/btwang/workshop/applications/fxtsoft</code></pre>


<blockquote style="margin: 0.0px 0.0px 0.0px 15.0px; font: 19.0px '.AppleSystemUIFont'; color: #0e0e0e">若 <span class="s1">configure</span> 成功，会看到：</blockquote>


<pre><code>config.status: creating headas-setup
Finished</code></pre>


<p class="p1"><span class="s1"><hr></span></p>
<p class="p1"><span class="s1"><h3><b>🔟 编译与安装</b></h3></span></p>


<pre><code>make -j$(sysctl -n hw.ncpu)
make install</code></pre>


<p class="p1">成功标志：</p>


<pre><code>Finished make install</code></pre>


<p class="p1"><span class="s1"><hr></span></p>
<p class="p1"><span class="s1"><h2><b>六、运行时环境（避免与 HEASoft 冲突）</b></h2></span></p>
<p class="p2"><br></p>
<p class="p1"><span class="s1"><h3><b>11️⃣ 安装目录结构</b></h3></span></p>


<pre><code>/Users/btwang/workshop/applications/fxtsoft/
└── x86_64-apple-darwin25.2.0/</code></pre>


<p class="p1"><span class="s1"><h3><b>12️⃣ FXTSoft 专用 CALDB</b></h3></span></p>


<pre><code>/Users/btwang/workshop/applications/fxtsoft/CALDB_v1.21</code></pre>


<p class="p1"><span class="s1"><hr></span></p>
<p class="p1"><span class="s1"><h2><b>七、最终环境切换方案（推荐）</b></h2></span></p>
<p class="p2"><br></p>
<p class="p1"><span class="s1"><h3><b>在<span class="Apple-converted-space"> </span></b></h3><h3><b>~/.zshrc</b></h3><h3><b><span class="Apple-converted-space"> </span>中定义</b></h3></span></p>


<pre><code>heasoft   # 激活 HEASoft (arm64)
fxtsoft  # 激活 FXTSoft (x86_64)
heaoff   # 清空 HEASARC 环境</code></pre>


<p class="p1">（完整 <span class="s1">.zshrc</span> 见你上一条确认版本）</p>
<p class="p2"><span class="s2"><hr></span></p>
<p class="p2"><span class="s2"><h2><b>八、常见错误与对应修复</b></h2></span></p>



错误信息 | 原因 | 解决
-- | -- | --
gfortran unusable | arm64 / x86_64 混用 | 用 /usr/local/bin/gfortran
library 'System' not found | 未设置 SDKROOT | -isysroot $SDKROOT
AST configure failed | Fortran 链接失败 | 加 libgfortran / libquadmath
HEASoft / FXTSoft 混乱 | HEADAS/CALDB 冲突 | 用函数切换




<p class="p1"><span class="s1"><hr></span></p>
<p class="p1"><span class="s1"><h2><b>九、经验总结（关键认知）</b></h2></span></p>
<p class="p1"><span class="s1"><ul><li>
<p class="p1"><b>Apple Silicon ≠ Intel</b><b></b></p>
</li><li>
<p class="p1"><b>HEASoft ≠ FXTSoft（即使同属 HEASARC 体系）</b><b></b></p>
</li><li>
<p class="p1"><b>永远不要在一个 shell 里 source 两个 headas-init.sh</b><b></b></p>
</li><li>
<p class="p1"><b>FXTSoft 在 macOS 上必须用 Rosetta（x86_64）</b></p>
</li></ul></span></p>
<p class="p1"><span class="s1"><hr></span></p>
