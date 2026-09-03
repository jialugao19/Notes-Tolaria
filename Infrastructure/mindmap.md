---
type: Note
---
# Mindmap

一个以服务器端 Markdown 文件为唯一数据源、通过 Web 浏览器远程访问，并支持 Markdown 与思维导图双向同步编辑的个人研究知识系统。

**服务器持久化**：Markdown 实际保存在服务器磁盘，而不是浏览器缓存。

**多设备 Web 访问**：任意电脑通过浏览器访问，不需要本地同步 Vault。

**文件浏览能力**：网页能读取服务器目录、打开不同 Markdown 文件。

**双向编辑**：Markdown 改动实时反映到 mindmap，mindmap 结构操作也回写 Markdown。

**自动保存**：浏览器编辑后自动写回服务器。

**外部修改感知**：如果服务器上的 Python、Git、VS Code 等修改 Markdown，网页能检测并刷新。

**并发/冲突保护**：避免两台电脑同时编辑把文件覆盖掉。

**Git 兼容**：服务器目录本身最好可以直接是 Git repo。

- 代码目录：workspace-auto/mindmap
- 数据文件：workspace-auto/mindmap-data/research.md
- origin：git@github.com:jialugao19/mindmap.git
- upstream：[https://github.com/kvaps/stackmind.git](https://github.com/kvaps/stackmind.git)
- user service：mindmap.service
- 容器：mindmap
- 状态：active / running / healthy
- 宿主机监听：127.0.0.1:8080


宿主机 /home/maomao/workspace-auto/mindmap-data/research.md

代码与数据完全分开：

- Git、Node、Vue 和部署脚本位于项目目录。
- Markdown 不在 Git 仓库中。
- 镜像更新不会把 Markdown 打包进镜像。
- 删除或重建容器不会删除宿主机数据。
- 不能同时运行两个 mindmap 容器写同一个文件。

主要实现文件：

- HTTP/API/SSE：workspace-auto/mindmap/server/app.ts:1
- 文件校验、监听和原子写：workspace-auto/mindmap/server/document.ts:1
- 浏览器同步状态机：workspace-auto/mindmap/src/stores/document.ts:23
- Markdown 编辑器：workspace-auto/mindmap/src/components/Editor.vue:1
- Markdown → Mindmap source map：workspace-auto/mindmap/src/utils/md2mind.ts:1
- Surgical patch：workspace-auto/mindmap/src/utils/surgical.ts:1
- 编辑拦截：workspace-auto/mindmap/src/components/MindMap.vue:1
- 操作能力矩阵：workspace-auto/mindmap/src/utils/mindmapCapabilities.ts:5
- 容器镜像：workspace-auto/mindmap/Containerfile:1
- Quadlet：workspace-auto/mindmap/deploy/mindmap.container:1


启动过程：

1. user systemd 启动 mindmap.service。
2. Podman 使用 localhost/mindmap:latest 创建容器。
3. 映射宿主机数据目录到 /data。
4. 容器内 Node 监听 0.0.0.0:8080。
5. Podman 只把它发布到宿主机 127.0.0.1:8080。
6. Node 初始化 DocumentManager。
7. 清理超过 24 小时的合法残留写入临时文件。
8. 校验 /data/research.md。
9. 监听 /data 父目录。
10. 启动每 3 秒一次的 reconciliation。
11. HTTP 服务开始响应。
12. Podman 每 30 秒调用 /readyz。


浏览器加载流程，打开页面时：

1. 浏览器载入 Vue 静态文件。
2. 删除旧版 stackmind:document localStorage 项，且不读取其中正文。
3. 建立 /api/events SSE。
4. 请求 GET /api/document。
5. 服务端读取并校验当前 research.md。
6. 返回正文、SHA-256 revision、sequence、modifiedAt。
7. CodeMirror 显示 Markdown。
8. Markdown 被解析成新的 Mindmap projection。
9. 状态变成 clean。

服务器版本始终优先于浏览器旧状态。
