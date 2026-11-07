# 重置监控

AxonHub 顶栏插件。点击右上角的活动图标，在弹窗里看 Codex、Antigravity、Claude 的额度重置和重置卡。

数据来自 [AIHOT 重置监控](https://aihot.news/resets) 使用的公开接口，接入方式见 [Agent 接入 · REST API](https://aihot.news/agent?tab=api)：

| 产品 | 地址 |
| --- | --- |
| Codex | `https://aihot.news/api/v1/codex-resets` |
| Antigravity | `https://aihot.news/api/v1/antigravity-resets` |
| Claude | `https://aihot.news/api/v1/claude-resets` |

这是完整日历，不是 `/recent`。AIHOT 要求同一个来源最多 10 分钟拉一次，更频繁会返回 429。弹窗每次打开都会新建 iframe，所以快照放在本插件的 KV 里：正文一个键，`ETag`、取到时间和重试时间一个键。10 分钟内再次打开直接用这份快照，并用 `If-None-Match`；没变化时是 304。

## 429

返回 429 时不再立刻重试。响应里的 `Retry-After` 会记下来；没有这个头就等 10 分钟。已经有快照就继续显示，并写明何时可以再试。还没有快照就只显示限流说明。宿主自己返回的限流也按同样的方式处理。

## 没有 KV 写权限

能打开弹窗的人都可以读已有快照。写入需要当前用户有 `write_plugins` 或是 Owner。没有写权限时：

- 有人已经存过的日历照常显示，10 分钟内不会再请求。
- 没有快照，或快照已过期时，仍然请求 AIHOT，并在当前弹窗里显示结果。
- 不调用 `setPluginKV`。关闭弹窗后这份新结果不会留下来，页面上会说明这一点。

## 目录

```text
manifest.json
ui/resets/
format.test.mjs
```

界面里的判断在 `ui/resets/format.js`，不依赖宿主。本地测试：

```bash
node --test format.test.mjs
```

## 打包

安装 AxonHub `unstable` 分支上的打包器。分支名可以直接作为版本，不需要 `cmd/pack-plugin/vX.Y.Z` 标签：

```bash
go install github.com/looplj/axonhub/cmd/pack-plugin@unstable
```

仓库里还有 workflow、README 和测试，不能直接把仓库根交给打包器。先复制要发布的文件：

```bash
stage=$(mktemp -d)
cp manifest.json "$stage/"
cp -R ui "$stage/"
pack-plugin "$stage" -o "$PWD/reset-watch-$(python3 -c 'import json; print(json.load(open("manifest.json"))["version"])').axplugin"
```

在 GitHub 上发布 `v*` 标签时，`.github/workflows/release.yml` 把 `manifest.json` 的 `version` 写成标签版本并推回默认分支，然后做同样的打包，把 `.axplugin` 和 sha256 挂到已有 Release 上。只有标签存在但还没有 Release 时，工作流才会新建一个。市场索引不在这个流程里。

界面判断的本地测试不在打包复制的文件里：

```bash
node --test format.test.mjs
```

## 安装

1. Owner 在「插件」›「插件管理」上传 `reset-watch-0.1.0.axplugin`。
2. 启用时授予这条出站规则：`^https://aihot\.news/api/v1/(codex|antigravity|claude)-resets$`。
3. 右上角出现「重置监控」。桌面端最多直接显示两个插件按钮，更多的在溢出菜单里。

不需要插件权限。个人额度和重置卡余额不在这个接口里，仍以各产品里看到的为准。确认帖的时间不是到账时间，插件也不预测还没宣布的下一次重置。
