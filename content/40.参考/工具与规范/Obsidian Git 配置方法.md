---
title: Obsidian Git 配置方法
type: howto
topic: 参考/工具与规范
tags: [参考, 工具, Obsidian, Git]
created: 2026-09-17
updated: 2026-09-17
status: active
source: team-knowledge 改造前 content/00-Inbox
---

	这里是ObsidianMac的配置方法和首次使用技巧，主要两种方式，包括：使用Obsidian可直接使用授权Git-token和配置本地Git上传公钥，通过SSH方式连接Git可用（mac适用此方式）。
#  一、Obsidian Git-token 配置方法
	此方法用于Obsidian可直接使用授权Git-token
## Step1.安装obsidian

![image-20260910102549248](img/image-20260910102549248.png)

打开clone的这个仓库。
## Step2.安装obsidian 插件

obsidian随便安装哪个git插件都行。

我安装的是 Git。 也没有配置什么。

![image-20260910091220921](img/image-20260910091220921.png)

![image-20260910102400539](img/image-20260910102400539.png)

![image-20260910102451763](img/image-20260910102451763.png)

## Step3. push提示输入token

关掉代理，obsidian上点加个文件commit，然后点push（或者使用tortiseGit在文件目录里点commit再push也行）。



![image-20260910101957404](img/image-20260910101957404.png)



![image-20260910101859265](img/image-20260910101859265.png)

## Step4.输入token，以后就不用输入了。

```
 github不让push这个。
```

push成功。以后直接在obsidian上git操作就行。
# 二、 通过SSH方式连接Git并配置插件
	建立本地公钥，上传Git公钥，此方法通过SSH方式连接Git可用。本方法参考Obsidian-Git官方文档，可通过本地SSH连接Git，然后配置Obsidian即可。
## Step1 终端生成密钥（打开 Mac 终端 Terminal）

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

## Step 2.Mac 把密钥加入 ssh‑agent

```
eval "$(ssh‑agent ‑s)"
# Mac把密钥存入钥匙串
ssh‑add ‑‑apple‑use‑keychain ~/.ssh/id_ed25519
```
## Step 3.复制公钥内容

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
## Step4.网页 GitHub 添加公钥

1. GitHub 网页右上角头像 → `Settings` → `SSH and GPG keys` → `New SSH key
2. Title：写一个标记
3. Key type：`Authentication key`
4. 大输入框粘贴刚刚复制的公钥全部文本
5. 点击 Add SSH key，完成。
## Step 5.把仓库远程地址切换为 SSH 地址（非常关键）

> 你现在仓库大概率是 HTTPS 地址，SSH 密钥不会生效。进入仓库目录执行：

```
git remote set‑url origin git@github.com:sunlingfeng70/team‑knowledge.git
```

校验是否成功：

```
git remote ‑v
```

看到输出以 `git@github.com:` 开头，代表 SSH 模式就绪。

## Step6. 测试连通性（终端执行）

```
ssh ‑T git@github.com
```

出现 `Hi xxx! You've successfully authenticated` 代表 SSH 密钥整套通了。

# 三、Obsidian使用技巧

## 找到Obsidian-Git工具
右侧 Ribbon（侧边小图标栏），找到 **分支 / 源代码图标**，点一下 → Git 源代码控制面板会在**右侧**弹出来。
- 图标样子：类似一个分叉的 git 分支符号
- 点完右侧面板：可以看到改动文件、Commit、Pull、Push 按钮

## 打开后的操作（你现在这套 SSH 环境）
右侧 Git 面板：
1. Pull：拉取 github 远端最新笔记
2. Commit all changes：提交本地改动（写提交备注）
3. Push：推送到 github
## 状态与忽略

这就是 Obsidian Git 的源码控制面板 ✅

> 右侧字母含义：

- **A = Added 新增文件**
- **M = Modified 文件被修改**
- **D = Deleted 删除**
- **U = Untracked 未追踪（Git 还没纳入版本管理）**

你现在看到大量 `U` 的文件：`app.json`、`appearance.json` 这些，**是 Obsidian 库内部配置文件**。

### ⚠️ 重点建议

**不要把 Obsidian 的内部配置文件全部提交到 Github** 多人协作仓库，只需要提交你的笔记（`.md` 文件），这些 json 配置文件每个人本地不一样，一起提交极易产生大量冲突。

### 解决办法：配置 `.gitignore`

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

## 显示PPT等文件方法
Obsidian 默认策略：**只自动识别、展示 Markdown（.md）、canvas 等笔记类文件，PPTX 这类二进制文件默认不加载显示**，虽然文件在硬盘，但是侧边树看不见。
#### 分步修复（按顺序执行）

##### 开启 Obsidian「显示全部文件类型」

1. 打开 Obsidian，进入**设置（⚙️） → Files & Links（文件与链接）**
2. 找到选项：`Detect all file extensions` ✅ **打开这个开关**（这个就是控制是否显示 pptx/pdf/png 等非 md 文件的核心选项）
##### 检查排除文件规则
1. 设置 → Files & Links，找到 `Excluded files` 确认里面**没有写 `*.pptx`**，如果有就删掉这一条过滤规则。
##### 注：
	检查一下原目录是否真实存在。
