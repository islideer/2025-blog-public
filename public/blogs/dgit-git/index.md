---
title: "dgit：跑在 Cloudflare Workers 上的 Git 服务器"
author: "我"
tags:
  - Git
  - Cloudflare
  - 分布式系统
  - 开源项目
description: "一个把每个 Git 仓库变成分布式小服务器的开源项目：无服务器、无文件系统、无 GitHub，却能完整跑起 git clone / push / fetch。"
---

## 一句话介绍

[dgit](https://github.com/littledivy/dgit)（Durable **git**）是一个**跑在 Cloudflare Workers 上的 Git 服务器**。它的核心理念是：

> 每个 Git 仓库都是一个 **Durable Object**（持久化对象）——一个自带名字、自带私有 SQLite 数据库的小型服务器。它来保存仓库的对象和引用、对标准 git 客户端说 smart HTTP 协议、并渲染一个 cgit 风格的可视化界面。

这个项目 2026 年 8 月 18 日刚创建，几天内就获得了 200+ star，作者 [LittleDivy](https://github.com/littledivy) 是 Cloudflare 生态里非常活跃的开发者。

## 它解决了什么问题

传统的 Git 服务器（GitHub、GitLab、Gitea）都需要你**养一台服务器**：操作系统、文件系统、数据库、备份、扩容……而 dgit 的思路完全不同：

- **没有 origin 服务器** —— 代码直接跑在 Cloudflare 的分布式边缘网络上
- **没有文件系统** —— 数据存在每个仓库自己的 SQLite 里（可选把 pack 文件卸载到 R2 对象存储）
- **没有 GitHub 在关键路径上** —— 不依赖任何第三方 Git 平台
- **冷仓库几乎零成本** —— 没人访问的仓库不消耗资源
- **天然隔离** —— 每个仓库是独立对象，一个仓库被刷爆不会拖慢其他仓库

## 它是怎么做到的

dgit 用 **TypeScript 从零实现了 git 协议**，包括：

- **pkt-line 帧格式** —— git 网络协议的基础
- **packfile 解析** —— 支持 ofs-delta 和 ref-delta 两种增量压缩
- **pack 生成** —— 基于流式 SHA-1
- **commit/tree/tag 编解码**
- **Myers diff 算法** —— 用于差异计算和 blame

全部代码只有一个外部依赖（`pako`，用于 zlib 解压）。整个项目 TypeScript 约 333KB。

### 存储与性能设计

| 机制 | 说明 |
|------|------|
| **push 直存** | 客户端推上来的 pack 字节**原样保存**，用索引记录每个对象 id → pack、偏移、delta 基，保留客户端压缩而非重新编码 |
| **R2 卸载** | 绑定 R2 bucket 后，pack 字节写进 R2，SQLite 只留索引，突破单单元格数据库容量上限 |
| **clone 缓存** | 完整 clone 第一次构建后缓存在 R2，之后直接从 Worker 流式返回，不再加载单元格 |
| **增量 fetch** | 协商时排除客户端已有的对象闭包，浅克隆边界正确截断，只下载缺失部分 |

浅克隆（`--depth`）、瘦包、side-band 进度、强制更新、删除引用……行为都和 GitHub 一模一样。

### 自带 cgit 风格 Web 界面

不只是个裸 Git 服务器，dgit 还自带完整的仓库浏览界面：

- 摘要页、引用列表、日志（带搜索和按路径历史）
- 目录树、文件浏览（带**语法高亮**）、blame
- commit 和任意区间 diff
- `format-patch` 输出（可直接 `git am` 应用）
- 任意引用的 tar.gz / zip 快照下载
- README 渲染的关于页、Atom 订阅源、提交活动统计
- 仓库可设置描述、所有者、分区、**私有权**（隐藏 + 全部读取需要 push token）

## 如何部署

### 部署到 Cloudflare Workers（免费额度即可）

```sh
npm install
npx wrangler r2 bucket create dgit-pack-cache   # 可选：R2 存储卸载 + clone 缓存
npx wrangler deploy
npx wrangler secret put GIT_TOKEN   # 推送密码
```

然后就能用了：

```sh
git remote add origin https://<your-host>/myrepo.git
git push -u origin main
```

### 自托管到自己的机器（celld）

dgit 也支持用 [celld](https://celld.dev) 跑在**你自己的机器**上，绕过托管平台的请求限制：每个仓库的 SQLite 数据库会复制到你的 bucket，节点随意重启，仓库都位级一致地恢复。

## 它还是个库

dgit 同时发布为 npm 包 [`durable-git`](https://www.npmjs.com/package/durable-git)，整个 Worker 只有三行核心代码：

```ts
import { createDurableGit, secretsEqual } from "durable-git";
export { RepoCell, Registry } from "durable-git";

export default createDurableGit({
  async authorize({ repo, op, private: priv, credentials, env }) {
    if (op === "read" && !priv) return true;
    return secretsEqual(credentials?.pass ?? "", await lookupDeployKey(env, repo));
  },
  onPush(event, env) {
    return fetch("https://ci.example.com/hook", { method: "POST", body: JSON.stringify(event) });
  },
});
```

- `authorize` 接管所有请求的授权（完全替换内置 token 策略）
- `onPush` 在每次 push 后触发（可以做 CI Webhook）
- 每个仓库还带 JSON 内容 API，可 headless 运行接自己的前端

## 对普通人的意义

虽然 dgit 目前是给**极客和自托管爱好者**玩的项目（没有 release 版本，PR 也关闭、通过邮件贡献），但它展示了一种很酷的可能性：

> **Git 托管可以不用服务器** —— 边缘计算 + 持久化对象，让"每个仓库都是一个小服务器"成为现实。

对个人开发者来说，这意味着将来可以花几乎为零的成本，在 Cloudflare 上托管自己的私有 Git 仓库，拥有完整的 Web 浏览界面，数据完全自控。

## 快速档案

| 项 | 值 |
|----|----|
| 项目 | [littledivy/dgit](https://github.com/littledivy/dgit) |
| 描述 | Git forge on Durable Objects |
| 语言 | TypeScript（+少量 Shell） |
| 许可证 | MIT |
| Star | 200+（创建仅数天） |
| 主页 | <https://git.littledivy.com> |
| npm 包 | [`durable-git`](https://www.npmjs.com/package/durable-git) |