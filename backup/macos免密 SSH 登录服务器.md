在 macOS 上，免密 SSH 登录服务器 的标准方法是通过 SSH 公钥认证（public key authentication）.一次配置好之后，你就可以直接输入：
```bash
ssh username@server.address
```
#### 1. 在本地生成密钥

在 macOS 终端输入：

```bash
ssh-keygen -t rsa -b 4096 -C "btwang@server"
```
一路回车即可

输出类似：
```bash
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/yourname/.ssh/id_rsa):
```
生成的文件：
	•	私钥：\~/.ssh/id_rsa（保密！）
	•	公钥：\~/.ssh/id_rsa.pub

#### 2. 上传公钥到服务器
```bash
ssh-copy-id username@server.address
```


#### 3. 免密登录
```bash
ssh username@server.address
```


#### 提交slurm作业

```bash
sbatch run_vegas.slurm
squeue -u $USER
```