
# 🛡️ Secure-Subconverter-Worker (安全订阅转换中转)

> 原项目地址
[sub-convert](https://github.com/jwyGithub/sub-convert) 
> 参考项目
[subconverter-cloudflare](https://github.com/jwyGithub/subconverter-cloudflare)

由AI编写：

基于 Cloudflare Worker 的 Subconverter 隐私保护与协议增强代理。

在使用第三方公开的 Subconverter（订阅转换）服务时，你是否担心真实的节点 IP、密码、UUID 被后端服务器偷偷记录？你是否遇到过使用新协议（Hysteria2 / VLESS）时，节点在转换后离奇消失，或者变成了“已过滤掉 XX 条线路”？

本项目是一个部署在 Cloudflare Worker 上的“防弹衣”。它会在你的节点发送给第三方后端之前，**对所有的真实 IP、域名和密码进行强力混淆伪装**，并在后端处理完毕后**无缝还原**。让你可以放心地使用任何公开的转换后端，而不用担心节点泄露！

## ✨ 核心特性 / Features

- 🔒 **隐私保护（防盗窃）**：自动识别并提取订阅中的真实 Server IP、UUID 和 Password。在发送给 Subconverter 前将其替换为随机生成的虚拟域名和 UUID，转换完成后再将其还原。第三方后端**永远只能看到假数据**。
- 🚀 **全协议支持（无惧新协议）**：完美支持 `SS (包含严格的 SS2022)`、`SSR`、`Vmess`、`Trojan`、`VLESS`、`Hysteria`、`Hysteria2 (hy2)`、`TUIC v5` 等各种现代协议。
- 🛡️ **突破机场面板限制（防过滤）**：内置 `Clash.Meta` UA 伪装。彻底解决部分机场面板自动过滤 Hysteria2/VLESS 节点，并下发“过滤掉XX条线路”假节点的问题。
- 🌍 **Emoji 与 UTF-8 完美兼容**：重写了底层的 Base64 编解码引擎。即便你的节点名称里写满了 `🇭🇰`、`🇺🇸` 等 Emoji 或者特殊中文字符，也绝对不会导致解析崩溃或节点丢失。
- ⚡ **无感极速**：基于 Cloudflare 全球边缘网络与 KV 内存级键值存储，临时映射表“阅后即焚”，无状态、不留痕、毫秒级响应。

---

## 🏗️ 原理说明 / How it works

1. **请求拦截**：你的客户端向本 Worker 发起订阅请求。
2. **下载与伪装**：Worker 使用 `Clash.Meta` 身份向你的机场拉取原始订阅（Base64 / YAML / 甚至单节点拼接）。
3. **数据脱敏**：将真实的 `Server IP` 替换为随机 `.com` 域名，将 `Password/UUID` 替换为随机串。并在 Cloudflare KV 中临时建立【假数据 <-> 真数据】的映射字典。
4. **安全转换**：将**全都是假节点**的订阅链接发送给你配置的 Subconverter 后端进行排版和分流规则生成。
5. **逆向还原**：拿到后端返回的配置后，根据 KV 字典将真实的 IP 和密码替换回去，并进行纯净 Base64 编码，最终返回给你的客户端。
6. **阅后即焚**：清理 KV 中的临时映射数据。

---

## 🚀 部署指南 / Deployment

### 1. 准备工作
- 一个 [Cloudflare](https://dash.cloudflare.com/) 账号。
- 在 Cloudflare 中创建一个 **KV 命名空间**（例如命名为 `SUB_KV`）。

### 2. 创建 Worker
1. 在 Cloudflare 面板进入 `Workers & Pages` -> `Create application` -> `Create Worker`。
2. 为 Worker 起个名字并点击 `Deploy`。
3. 点击 `Edit code`，将本项目中的 `worker.js` 代码全部复制并覆盖进去，点击 `Save and deploy`。

### 3. 配置环境变量与 KV 绑定
在 Worker 的 `Settings` -> `Variables` 中，进行以下配置：

#### ⚙️ 环境变量 (Environment Variables)
| 变量名 | 必填 | 默认值 / 示例 | 说明 |
| --- | --- | --- | --- |
| `BACKEND` | **是** | `https://sub.id9.cc` | 转换后端地址。可填写你信任的、或任意公开的 Subconverter API 地址。 |
| `FRONTEND` | 否 | *(内置在线前端)* | 订阅转换的前端 Web UI 页面地址。 |
| `REMOTE_CONFIG` | 否 | `在线规则 \| https://...` | 前端页面中的远程分流规则列表。格式：`名称\|URL`，多行隔开。 |
| `LOCK_BACKEND` | 否 | `true / false ` | 是否在前端页面锁定“后端地址”下拉框，防止用户篡改。不添加默认为true. |

#### 🗄️ KV 命名空间绑定 (KV Namespace Bindings)
| 变量名 (Variable name) | 绑定的 KV 空间 (KV namespace) |
| --- | --- |
| `SUB_BUCKET` | 选择你在第 1 步创建的 KV 空间（如 `SUB_KV`） |

---

## 💡 使用方法 / Usage

部署完成后，直接在浏览器中访问你的 Worker 域名（例如 `https://your-worker-name.your-subdomain.workers.dev`），即可看到自带的 Subconverter 前端转换界面。

- 将你的机场订阅链接粘贴进去。
- 选择需要的客户端（如 Clash、Sing-box 等）。
- 点击生成订阅链接，将生成的链接放入你的代理软件中即可！

> **进阶用法**：在生成的订阅链接 URL 参数中，你可以通过增加 `&bd=https://其他后端地址` 来临时覆盖全局的 `BACKEND` 设置。

---

## 🛠️ 疑难杂症 (Troubleshooting)

**Q: 为什么我的 Hysteria2 / VLESS 节点转换后全没了？**
A: 请确保代码最新。本项目已内置 UA 伪装及向后端强制注入 `client=clash-meta` 参数的功能，可穿透 99% 机场面板的协议过滤机制。

**Q: 为什么 SS 节点转换后报错或连不上？**
A: 这是因为 SS2022 等新版 Shadowsocks 对密码 Base64 长度有严格校验。本项目已内置智能识别，对 SS 协议采用“仅混淆域名，保留合法密钥”的策略，完美解决 SS2022 解析报错问题。

**Q: 转换包含 Emoji 中文的节点总是失败？**
A: JS 原生的 Base64 不支持 UTF-8。本项目底层使用 `TextEncoder/TextDecoder` 重构了 Base64 编解码引擎，完美兼容包含各类特殊符号的订阅。

---

## 📜 声明与协议 / License

本项目基于 MIT 协议开源。
代码仅供学习与研究 Cloudflare Workers 技术使用，请勿用于任何非法用途。

---
