# Clash Linux 使用说明

这个项目用于在 Linux 服务器、容器、远程开发环境里启动 Clash，并提供两种使用方式：

- 浏览器 Dashboard：用于网页里选择节点、切换模式。
- 终端 CLI：像 `proxy_on`、`proxy_off` 一样直接在终端里操作。

## 第一步：配置订阅或本地 YAML

进入项目目录，复制配置文件：

```bash
cp .env.example .env
vim .env
```

`.env` 里 `CLASH_URL` 和 `CLASH_CONFIG_FILE` 二选一。

如果使用订阅链接：

```bash
export CLASH_URL='https://example.com/subscription'
# export CLASH_CONFIG_FILE='conf/my-clash.yaml'
export CLASH_SECRET=''
```

如果使用本地 Clash YAML：

```bash
export CLASH_URL=''
export CLASH_CONFIG_FILE='my-clash.yaml'
export CLASH_SECRET=''
```

`CLASH_SECRET` 可以留空，启动时会自动生成。

## 第二步：启动 Clash

执行：

```bash
bash start.sh
```

启动成功后会生成最终配置：

```text
conf/config.yaml
```

并启动 Clash API：

```text
http://127.0.0.1:9090
```

当前终端立即加载命令：

```bash
source /etc/profile.d/clash.sh
```

新开的终端一般会自动加载，不需要再手动 `source`。

## 第三步：开启或关闭终端代理

开启当前终端代理：

```bash
proxy_on
```

效果：当前终端里的 `curl`、`git clone`、`pip` 等命令会走 Clash 代理。

关闭当前终端代理：

```bash
proxy_off
```

效果：取消当前终端的代理环境变量。

注意：`proxy_on` / `proxy_off` 只影响当前终端。新开一个终端后，如果要使用代理，需要再次执行 `proxy_on`。

## 第四步：使用 VS Code 端口转发打开 Dashboard

如果你是在 VS Code Remote、云服务器、容器、Notebook 里运行本项目，浏览器通常不能直接访问服务器里的 `9090` 端口。需要先在 VS Code 里做端口转发。

操作步骤：

1. 在 VS Code 底部或侧边栏打开 `端口 / Ports` 面板。
2. 添加端口 `9090`。
3. 看到 `9090` 转发到 `localhost:9090`，说明端口转发成功。

![VS Code 转发 9090 端口](asserts/image1.png)

4. 转发成功后，在本地浏览器打开：

```text
http://localhost:9090/ui
```

打开后会看到 YACD 添加后端的页面：

![YACD 添加 Clash 后端](asserts/image.png)

填写方式：

1. `API Base URL` 填：

```text
http://127.0.0.1:9090
```

2. `Secret(optional)` 填 `bash start.sh` 启动时输出的 Secret。
3. `Label(optional)` 可以不填。
4. 点击 `Add`。
5. 点击下面新增的后端卡片进入 Dashboard。

进入 Dashboard 后，就可以在网页里查看连接、切换模式、选择节点。

## 第五步：使用终端 CLI 切换节点

统一命令是：

```bash
proxy
```

查看帮助：

```bash
proxy
```

查看当前状态：

```bash
proxy status
```

效果：显示当前 Clash 模式，以及每个策略组当前选择的节点。

查看当前模式：

```bash
proxy mode
```

切换到规则模式：

```bash
proxy mode Rule
```

效果：按照 Clash 规则分流。

切换到全局代理模式：

```bash
proxy mode Global
```

效果：所有流量走全局节点。

切换到直连模式：

```bash
proxy mode Direct
```

效果：所有流量直连，不走代理节点。

查看所有可切换策略组：

```bash
proxy groups
```

效果：列出类似 `GLOBAL`、`🔰国外流量`、`🎬Netflix` 这样的策略组。

查看某个策略组里的节点：

```bash
proxy nodes "🔰国外流量"
```

效果：列出该策略组下面所有可选节点，当前节点后面会有 `*`。

测试某个策略组里的节点延迟：

```bash
proxy delay "🔰国外流量"
```

效果：显示该策略组下每个节点的延迟，方便选择更快的节点。

切换节点：

```bash
proxy set "🔰国外流量" "香港—E3"
```

效果：把 `🔰国外流量` 这个策略组切换到 `香港—E3` 节点。Dashboard 刷新后也能看到同步结果。

打开交互菜单：

```bash
proxy menu
```

效果：通过菜单选择模式、策略组和节点，不需要记完整命令。选择节点时会先并发测试延迟，并在节点列表里显示延迟。

## 节点测速太慢时

节点很多时可以提高并发数、降低超时时间：

```bash
export CLASH_DELAY_PARALLEL=16
export CLASH_DELAY_TIMEOUT=2000
proxy delay "🔰国外流量"
```

说明：

- `CLASH_DELAY_PARALLEL=16`：同时测试 16 个节点。
- `CLASH_DELAY_TIMEOUT=2000`：单个节点最多等待 2000 毫秒。

## 常用维护命令

重启 Clash，不重新下载订阅：

```bash
bash restart.sh
```

停止 Clash：

```bash
bash shutdown.sh
```

停止后，如果当前终端开启过代理，再执行：

```bash
proxy_off
```

查看日志：

```bash
tail -n 100 logs/clash.log
tail -f logs/clash.log
```

## Dashboard 和终端 CLI 的关系

Dashboard 和 `proxy` 命令使用的是同一个 Clash API。

- 在终端执行 `proxy set ...` 切换节点后，刷新 Dashboard 可以看到变化。
- 在 Dashboard 里切换节点后，执行 `proxy status` 可以看到变化。
- 两边是同步的。

## 文件生成说明

启动时的配置生成流程：

```text
订阅链接或本地 YAML
        |
        v
temp/clash.yaml              原始订阅/原始 YAML
        |
        v
temp/clash_config.yaml       标准化后的 Clash 配置
        |
        v
temp/config.yaml             模板 + 代理配置合并结果
        |
        v
conf/config.yaml             Clash 实际运行配置
```

Clash 实际使用的是：

```text
conf/config.yaml
```

## 常见问题

### 终端提示没有 `proxy` 命令

执行：

```bash
source /etc/profile.d/clash.sh
```

### `proxy status` 提示无法连接 Clash API

先看 Clash 是否启动：

```bash
tail -n 100 logs/clash.log
```

然后重新启动：

```bash
bash start.sh
source /etc/profile.d/clash.sh
```

### Dashboard 打不开

确认已经在 VS Code 里转发端口 `9090`，然后访问：

```text
http://localhost:9090/ui
```

### Dashboard 已打开但无法连接后端

检查填写是否正确：

```text
API Base URL: http://127.0.0.1:9090
Secret: start.sh 输出的 Secret
```

### ping 不通 Google

正常。`ping` 使用 ICMP，不代表 HTTP/HTTPS 代理不可用。建议这样测试：

```bash
proxy_on
curl -I https://www.google.com
```

## 安全提醒

- 本项目不提供订阅链接，请自行准备合法可用的 Clash 订阅或 YAML。
- `.env`、订阅链接、Secret 都是敏感信息，不要提交到公开仓库。
- 使用本项目产生的网络行为由使用者自行负责。

## 许可

本项目沿用 GPL v3.0 许可，详情见 [LICENSE](LICENSE)。
