# 团队知识库 (Team Knowledge Base)

团队唯一的文档知识库。基于 **Obsidian** 管理，用 **GitHub** 作为文件管理与协作中枢，团队成员通过 **Obsidian Git 插件** 自动同步。

> ⚠️ 本仓库为 **公有仓库**：任何人可读。**绝不提交**密钥、密码、Token、内部敏感数据。

## 目录结构

| 目录 | 用途 |
|------|------|
| `00-Inbox/` | 快速随手记（未分类） |
| `10-Projects/` | 进行中/历史项目文档 |
| `20-Areas/` | 长期领域知识 |
| `30-Resources/` | 参考资料、素材 |
| `40-Archive/` | 归档 |
| `_templates/` | 文档模板 |
| `_attachments/` | 图片/附件统一存放 |

首页入口见 [`index.md`](index.md)。

## 🚀 新成员 5 步上手

### 1. 安装 Obsidian
到 <https://obsidian.md> 下载安装。

### 2. Clone 仓库并作为 Vault 打开
```bash
git clone https://github.com/sunlingfeng70/team-knowledge.git
```
在 Obsidian 中：**Open folder as vault** → 选择刚 clone 的 `team-knowledge` 目录。

### 3. 安装 Obsidian Git 插件
设置 → 第三方插件 → 关闭安全模式 → 浏览社区插件 → 搜索 **Obsidian Git** → 安装并启用。

### 4. 配置访问令牌 (PAT) 与自动同步
1. 在 GitHub 生成 **fine-grained PAT**（仅本仓库 Read/Write 权限）：
   GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token → 勾选本仓库 Content: Read/Write。
2. 在 Obsidian Git 插件设置里填入认证（HTTPS + PAT）。
3. 设置自动备份间隔 `Auto backup interval (minutes)` = **10**，自动拉取间隔 `Auto pull interval (minutes)` = **10**。

### 5. 开始记录
从 `00-Inbox` 随手记，用 `_templates` 模板创建正式文档。

## 🤝 同步三原则（务必遵守）

1. **先 pull 再编辑**：开始编辑前先同步（Obsidian Git → Pull），避免基于旧版本修改。
2. **小步提交**：文档改完及时保存提交，不要攒一大批。
3. **避免同文件并发编辑**：和他人同时改同一文件会产生**合并冲突**。协作前先沟通，或错峰编辑。

### 遇到合并冲突怎么办？
- Obsidian Git 插件会在状态栏提示冲突。
- 打开冲突文件，Git 标记 `<<<<<<<` / `=======` / `>>>>>>>` 之间是需要手动取舍的内容。
- 保留正确内容、删掉标记行，保存并提交。
- 不确定时**找技术同学协助**，切勿乱删内容。

## 🔒 安全红线

- 本仓库**公有可见**，禁止提交：API 密钥、密码、Token、内部敏感信息。
- 误提交敏感信息后，仅删除文件不够，需按 GitHub 官方流程**历史清理**或重置 Token。

## 📝 命名规范（建议）

- 文件名：小写英文 + 短横线，如 `meeting-2025-01-05.md`。
- 附件图片放 `_attachments/`，正文用 `![[附件名]]` 引用。
- 文档顶部写 frontmatter（tags / created / status）。

## 📄 License

[MIT](LICENSE)
