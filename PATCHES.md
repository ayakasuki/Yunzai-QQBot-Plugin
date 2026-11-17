# 自维护魔改说明（ayakasuki fork）

本仓库是 [TimeRainStarSky/Yunzai-QQBot-Plugin](https://github.com/TimeRainStarSky/Yunzai-QQBot-Plugin)
的自维护分支（基线上游 `60a2e87`），只做**必要修复**，保持与上游结构一致，方便以后合并上游更新。

配套 SDK 用自维护版：[ayakasuki/qq-group-bot](https://github.com/ayakasuki/qq-group-bot)（`1.1.0-taomei.1`）。

## 分支说明

| 分支 | 用途 |
|---|---|
| `master` | 上游镜像，**不要直接改** |
| `taomei/maintained` | 自维护分支，本仓库的补丁都在这 |

## 补丁清单

### 1. 按钮回调登记表持久化（`loadCallbackStore` / `saveCallbackStore`）

**问题**：`data.bot.callback` 只是内存对象——

- 机器人一重启，历史消息上的按钮全部失效；
- 旧版还给登记项加了 **300 秒**自动过期，按钮几分钟后就点不动；
- 失效后点按钮，插件走兜底的 `event.reply(1)`（**code=1 = 操作失败**），QQ 客户端就弹「机器人服务异常」。

**修复**：

- 登记时写入 `data/QQBot/callback-<self_id>.json`（2 秒防抖，带 `expire` 字段）；
- `connect()` 里 `callback: await loadCallbackStore(id)` 恢复，过期项自动丢弃；
- 有效期 `300000` → **7 天**；
- 日志：`[QQBot] 已恢复 N 个按钮回调登记`。

### 2. 登记未命中时不再回 `code=1`

```js
// 原版
} else {
  event.reply(1)      // ← 客户端据此弹「机器人服务异常」
}
// 改为：只打 warn，静默忽略（按钮已过期，重新发指令即可）
```

### 3. 交互事件回应（`makeCallback`）修正

- QQ 官方文档：`interaction_id` 取自 `INTERACTION_CREATE` 事件的 `d.id`，**不带 `INTERACTION_CREATE:` 前缀**。
  插件这里先剥掉前缀再回应（SDK 也做了同样修复，双重保险）；
- 原版 `event.reply` 的包装函数在 `catch` 里引用尚未初始化的 `data.self_id`（TDZ），
  一旦回应失败就会抛 ReferenceError 把真实错误吞掉 —— 改为不依赖 `data` 的日志。

### 4. `makeButton()` 补齐 QQ 必填字段

QQ「消息按钮」文档里这些字段是**必填**，原版缺失：

| 字段 | 原版 | 现在 |
|---|---|---|
| `render_data.visited_label` | 取 `button.clicked_text`，未传即 `undefined` | 回退成 `button.text` |
| `action.unsupport_tips` | 三个分支都没设 | 默认 `"当前客户端不支持此操作"`，可用 `button.unsupport_tips` 覆盖 |

可在每个按钮上用 `QQBot.action` / `QQBot.render_data` 继续覆盖。

## 按钮写法建议（插件侧调用方）

```js
const btn = (text) => ({
  text,
  clicked_text: text,                       // → render_data.visited_label
  callback: `#指令${text}`,                 // → action.type=1 交互回调，点一下直接触发
  QQBot: { action: { data: `#指令${text}` } } // → action.data，QQ 会回显成 resolved.button_data，
                                              //    登记表丢失时也能兜底执行
})
```

⚠️ 不要用 `input` + `send: true`：那是「指令按钮」(action type 2) 的 `enter` 自动发送，
QQ 现在不认这个字段，点完只会把文字填进输入框，用户还得手动发送。
（需要 `config/QQBot.yaml` 的 `toCallback: true`，默认就是 true。）

## 部署

```bash
git clone -b taomei/maintained https://github.com/ayakasuki/Yunzai-QQBot-Plugin.git plugins/QQBot-Plugin
cd plugins/QQBot-Plugin && pnpm i     # 会按 package.json 从 github:ayakasuki/qq-group-bot 拉魔改 SDK
```

改完插件文件后 Yunzai 会热重载（日志 `[修改插件][QQBot-Plugin]`），无需重启；
但 `index.js` 本体的改动（含本仓库全部补丁）需要 `pm2 restart` 生效。

## 升级上游

```bash
git fetch upstream && git merge upstream/master    # 冲突集中在 index.js 的四处补丁
grep -n "桃妹补丁" index.js                        # 全部补丁都带这个注释，方便定位
```
