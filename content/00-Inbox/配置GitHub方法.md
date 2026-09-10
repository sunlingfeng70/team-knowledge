## 一、部署过程
1，目前Obsidian没有Auto授权Git-token方法，需要三方测试使用。
2，还没有找到其它三方方法。
3，建立本地公钥，上传Git公钥，此方法通过SSH方式连接Git可用，具体步骤如下。
## 二、步骤
### 1，官方资料：
参考Obsidian-Git官方文档，可通过本地SSH连接Git，然后配置Obsidian即可。
![[Pasted image 20260910144626.png|177]]
### 2，Git-SSH配置步骤
#### 步骤 1：终端生成密钥（打开 Mac 终端 Terminal）

把命令里邮箱替换成你 GitHub 注册邮箱，执行：

```
ssh‑keygen ‑t ed25519 ‑C "你的github邮箱@xxx.com"
```

> 老设备不支持 ed25519 才用 rsa 版本：
> 
> ```
> ssh‑keygen ‑t rsa ‑b 4096 ‑C "你的github邮箱@xxx.com"
> ```



1. `Enter a file in which to save the key`：**直接回车，使用默认保存路径**

> 不要输入文件名，直接回车。

2. `Enter passphrase (empty for no passphrase)`：

> ❗**建议直接回车（不设置密钥密码）**。 如果你这里设置了密码，Obsidian‑Git 每次 Push/Pull 都要求输入密钥密码，会卡住插件。 👉家庭个人电脑，直接两次回车，passphrase 留空。

输出类似：

```
Generating public/private ed25519 key pair.
Enter file in which to save the key (/Users/xxx/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```

> 完成后生成一对文件： 私钥：`~/.ssh/id_ed25519`（本地保留，绝不外泄） 公钥：`~/.ssh/id_ed25519.pub`（复制这个文件内容粘贴到 GitHub 网页）稀土掘金

#### 步骤 2：Mac 把密钥加入 ssh‑agent

```
eval "$(ssh‑agent ‑s)"
# Mac把密钥存入钥匙串
ssh‑add ‑‑apple‑use‑keychain ~/.ssh/id_ed25519
```
#### 步骤 3：复制公钥内容

```
# mac直接复制公钥到剪贴板
pbcopy < ~/.ssh/id_ed25519.pub
```

> 剪贴板已经拿到完整公钥，不要手动编辑、不要加换行空格。
> 找到文件上传则使用以下命令
```
cat ~/.ssh/id_ed25519.pub
```
#### 方法 2：图形界面打开文件夹（可视化找文件）

1. 打开访达 (Finder)，按快捷键 `Shift + Command + G`
2. 在弹出输入框粘贴路径：

```
~/.ssh
```

3. 回车，直接进入 ssh 文件夹
4. 在里面找到文件：`id_ed25519.pub`
#### 步骤 4：网页 GitHub 添加公钥

1. GitHub 网页右上角头像 → `Settings` → `SSH and GPG keys` → `New SSH key
2. Title：写一个标记
3. Key type：`Authentication key`
4. 大输入框粘贴刚刚复制的公钥全部文本
5. 点击 Add SSH key，完成。
#### 步骤 5：把仓库远程地址切换为 SSH 地址（非常关键）

> 你现在仓库大概率是 HTTPS 地址，SSH 密钥不会生效。进入仓库目录执行：

```
git remote set‑url origin git@github.com:sunlingfeng70/team‑knowledge.git
```

校验是否成功：

```
git remote ‑v
```

看到输出以 `git@github.com:` 开头，代表 SSH 模式就绪。

#### 测试连通性（终端执行）

```
ssh ‑T git@github.com
```

出现 `Hi xxx! You've successfully authenticated` 代表 SSH 密钥整套通了。

## Obsidian使用技巧

### 找到Obsidian-Git工具
右侧 Ribbon（侧边小图标栏），找到 **分支 / 源代码图标**，点一下 → Git 源代码控制面板会在**右侧**弹出来。
- 图标样子：类似一个分叉的 git 分支符号
- 点完右侧面板：可以看到改动文件、Commit、Pull、Push 按钮

### 打开后的操作（你现在这套 SSH 环境）
右侧 Git 面板：
1. Pull：拉取 github 远端最新笔记
2. Commit all changes：提交本地改动（写提交备注）
3. Push：推送到 github
### 状态与忽略

这就是 Obsidian Git 的源码控制面板 ✅

> 右侧字母含义：

- **A = Added 新增文件**
- **M = Modified 文件被修改**
- **D = Deleted 删除**
- **U = Untracked 未追踪（Git 还没纳入版本管理）**

你现在看到大量 `U` 的文件：`app.json`、`appearance.json` 这些，**是 Obsidian 库内部配置文件**。

#### ⚠️ 重点建议

**不要把 Obsidian 的内部配置文件全部提交到 Github** 多人协作仓库，只需要提交你的笔记（`.md` 文件），这些 json 配置文件每个人本地不一样，一起提交极易产生大量冲突。

#### 解决办法：配置 `.gitignore`

在你的 `team-knowledge` 库根目录新建文件，名字就叫 `.gitignore`，写入下面内容：

```
# Obsidian 配置文件，不上传
.obsidian/
# 缓存
.obsidian/cache
# 系统文件
.DS_Store
```

保存。

> 作用：告诉 Git 忽略 `.obsidian` 文件夹下所有配置，不再出现一大堆 U 未追踪文件。