# 示例：Web 服务器 / API

服务器的特殊关注点是**生命周期**：代理需要在后台启动服务器、验证其是否已就绪、与之交互，然后将其干净地关闭。如果使用前台运行的 `npm start` 命令并阻塞终端，对代理来说是毫无用处的。

## 应遵循的结构

一个好的服务器运行技能应具备：

1. **前置条件与设置**——与任何项目相同。
2. **运行**——采用下文所述的后台启动模式，而不是阻塞式的命令。
3. **验证**——使用 `curl` 或类似工具确认服务器确实已启动。
4. **停止**——提供一种能够干净终止后台进程的方法。

如果后台启动、就绪检查和冒烟测试这串操作超过几行代码，可将其放入技能目录下的 `smoke.sh` 脚本中，并在 `SKILL.md` 中注明“运行 `smoke.sh` 脚本”。只需一条命令，通过其退出码即可判断服务器是否正常运行。

## 后台启动模式

不要这样写：

> ```bash
> npm start
> ```这样会阻塞。改为展示如何在后台启动、等待服务就绪，并在稍后获取进程 ID：

```bash
npm start &> /tmp/server.log &
SERVER_PID=$!
```

```bash
# 等待服务启动（可根据需要调整超时时间和端口）
for i in {1..30}; do
  curl -sf http://localhost:3000/health > /dev/null && break
  sleep 1
done
```

然后是验证步骤：

```bash
curl http://localhost:3000/health
# -> {"status":"ok"}
```停止：

```bash
kill $SERVER_PID
# $! 是 npm 包装进程的 PID，而 npm 不会将 SIGTERM 信号转发给它启动的服务器——只有杀死端口的监听进程才能可靠地释放该端口：
lsof -ti:3000 -sTCP:LISTEN | xargs -r kill
```

优先使用已捕获的 PID 或端口号，而不是使用 `pkill -f "<pattern>"`。像 `pkill -f "next|vite|node"` 这样的宽泛模式可能会匹配代理自身的命令行，并错误地终止运行这些命令的会话。

## 值得记录的细节

- **使用的端口。** 明确指出，并说明如何覆盖（`PORT=4000 npm start`）。
- **“已就绪”的标志是什么。** 例如特定的日志行，或用于检查服务状态的健康端点。
- **所需的环境变量。** 如数据库连接字符串、API密钥等——如果列表较长，可提供一个`.env`模板。
- **热重载与生产模式的区别。** 如果两者有显著差异，应说明在什么情况下使用哪种模式。
- **依赖的服务。** 如果服务器需要 Redis、Postgres 等，可以指向一个用于启动这些服务的 `docker-compose` 文件，或者直接列出 `docker run` 命令。

## 示例片段

以下是一个典型 Node API 的“运行”部分示例：

> ## 运行
>
> 在后台启动开发服务器：
>
> ```bash
> npm run dev &> /tmp/api.log &
> ```
>
> 服务器监听 3000 端口。等待其就绪后进行验证：
>
> ```bash
> for i in {1..20}; do
>   curl -sf http://localhost:3000/health && break
>   sleep 0.5
> done
> curl http://localhost:3000/health
> # -> {"status":"ok","version":"1.2.3"}
> ```
>
> 日志输出至 `/tmp/api.log`。可通过杀死该端口的监听进程来停止服务（`npm run dev &` 后的 `$!` 是 npm 的包装进程，而 npm 不会将 SIGTERM 信号传递给它所启动的服务器）：
>
> ```bash
> lsof -ti:3000 -sTCP:LISTEN | xargs -r kill
> ```
>
> ### 环境变量
>
> | 变量            | 是否必填 | 默认值 | 备注                     |
> |-----------------|----------|--------|--------------------------|
> | `DATABASE_URL`  | 是       | —      | PostgreSQL 连接字符串    |
> | `PORT`          | 否       | `3000` |                          |
> | `LOG_LEVEL`     | 否       | `info` | `debug` / `info` / `warn` / `error` |

