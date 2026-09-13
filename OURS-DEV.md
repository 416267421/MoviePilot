# MoviePilot v2 自有 fork 二开工作流

- fork：https://github.com/416267421/MoviePilot（fork 自 jxxghp/MoviePilot）
- 工作分支：`v2`（上游 v2 已冻结在 v2.15.6，2026-08-16 最后 commit，作者全力转 v3）
- 本地：`~/dev/MoviePilot-ours`（浅克隆 depth 50）
- remote：`origin` = 我们的 fork（PAT 走 https，token 在 ~/.hermes/.env GITHUB_TOKEN），`upstream` = 官方仓库

## 已落库的补丁（2026-09-13）

| commit | 内容 |
|---|---|
| `3a1c052` | spider 彩虹桥补丁：`app/modules/indexer/spider/__init__.py`，ptchdbits.co 无 browse 时注入带参数地址（inclbookmarked=0&incldead=0&spstate=0），绕开置顶堆 |
| `0056d24` | vendor brushflowlowfreq 4.3.2 + 单字符别名跳过补丁：`app/plugins/brushflowlowfreq/`（.gitignore 已放行该目录） |

两个补丁原为 docker cp 进容器 /app/app/ 的热修（容器重建即丢），现已永久在 git 里。

## 日常二开流程

1. **改码**：在 ~/dev/MoviePilot-ours 的 v2 分支上直接改（或开 feature 分支再并回）。
2. **commit**：写清楚补丁用途和来源日期。
3. **push**：`git push origin v2`（如需代理：`export https_proxy=http://192.168.8.200:7893`）。
4. **构建镜像**（在飞牛 8.201 或本地有 docker 的机器上）：
   ```bash
   cd ~/dev/MoviePilot-ours
   docker build -f docker/Dockerfile -t moviepilot:ours-v2.15.6-p1 .   # p1=补丁号，加新补丁就 +1
   ```
   注意构建要连网拉基础镜像和 pip 包，需要代理的话给 docker build 配 build-arg 或 daemon 代理（HTTP_PROXY=192.168.8.200:7893）。
5. **替换生产**（飞牛 /vol1/1000/docker/moviepilot/）：
   - 改 docker-compose.yml 的 image 为 `moviepilot:ours-v2.15.6-p1`
   - `docker compose down && docker compose up -d`
   - 配置 /config/app.env 和 PG（moviepilot-pg）都在卷里，不受影响
6. **验证**：进容器确认补丁文件在（`grep ptchdbits /app/app/modules/indexer/spider/__init__.py`、`grep "本地版" /app/app/plugins/brushflowlowfreq/__init__.py`），再看日志刷流轮次是否正常。

## 重要注意

- **容器重建不再需要 docker cp 补丁**——补丁已在镜像里。这是本 fork 存在的核心意义（2026-09-11 容器重建丢补丁踩过坑）。
- **插件更新风险**：brushflowlowfreq 是 vendor 进镜像的市场插件，如果在 MP 界面里点了"更新插件"，会用市场版覆盖 /app/app/plugins/ 里的补丁版（运行时覆盖，重启后恢复镜像版）。更新前先评估是否要重新 vendor 新版+补丁。
- **同步上游**：v2 已冻结基本不会有新 commit，若要检查：`git fetch upstream v2 && git log v2..upstream/v2 --oneline`。
- **不碰官方仓库**：只 push 到 origin，永不 push 到 upstream。
- v3 迁移评估（上游 v3 稳定半年后再说）：看 /tmp/zcode-mp2-dev-scout.md（2026-09-12 侦察报告）。
