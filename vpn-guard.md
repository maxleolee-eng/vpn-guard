# vpn-guard

更新时间：2026-05-19 23:18 +0800

## 原始任务

为本机正在运行的 Clash Verge Rev 做一个常驻系统代理守护，要求：

1. 每隔七天更新订阅，并重新加载更新后的订阅，确保 UI 上能正确显示更新时间。
2. 监控当前节点延迟，一旦超过 `400ms`、出现 `error`、超时等异常，就在包含 `日本` 的节点中，通过三次真实 URL 测试计算平均延迟并排序，切换到最快节点。
3. 一旦 Clash 崩溃，迅速重启，然后刷新包含 `日本` 的节点，按同样方法排序后选择最快日本节点。
4. 确保 Clash 网络设置为 TUN 虚拟网卡模式，代理模式选择 `规则`。

本次归档要求：

- 在 `/Users/max/Desktop/claudespace/` 下新建 `vpn-guard/`。
- 将本对话全部关键内容、执行细节和结论写入 `vpn-guard.md`。
- 将生成的几个文件都存入该目录。

## 本机实际状态

实测本机 Clash Verge Rev 已在运行：

- UI 进程：`/Applications/Clash Verge.app/Contents/MacOS/clash-verge`
- 核心进程：`/Applications/Clash Verge.app/Contents/MacOS/verge-mihomo`
- 数据目录：`/Users/max/Library/Application Support/io.github.clash-verge-rev.clash-verge-rev`
- Mihomo 控制 socket：`/tmp/verge/verge-mihomo.sock`
- 当前运行配置：`mode=rule`
- 当前 TUN：`tun=true`
- 当前 TUN 设备：`utun4`
- mixed port：`7897`

当前远程订阅 profile：

- uid：`Rcvqn9FttS2T`
- name：`CordCloud_Clash_1778957052.yaml`
- file：`Rcvqn9FttS2T.yaml`

安全处理：归档中不写订阅 URL、密码、token、节点密码或私钥。

## 关键发现

之前本机已有一个旧任务：

- `/Users/max/Library/LaunchAgents/LOCAL_CLASH_VPN_Update.plist`
- `/Users/max/.openclaw/workspace/scripts/local_clash_vpn_update.sh`

但它是隔离测试用 Mihomo，使用独立端口 `127.0.0.1:17898` 和控制口 `127.0.0.1:19091`，并明确不接管系统 TUN。因此这次没有复用它，避免混淆系统 Clash Verge 实例。

源码和本机日志确认：

- `verge://refresh-proxy-config` 不是无参更新命令。
- 无参调用会在 Clash Verge 日志里出现 `missing url parameter in deep link`。
- 外部 deep link 主要用于导入订阅，不是已有 profile 的内部 `update_profile` 命令。
- 因此守护脚本不依赖这个 deep link，而是直接读取现有 `profiles.yaml` 中当前 remote profile 的订阅地址，下载同一订阅，校验后重建运行配置并通过 Mihomo API reload。

## 实现文件

主脚本：

- `clash-verge-guardian`
- 原始安装位置：`/Users/max/.local/bin/clash-verge-guardian`
- 归档副本：`/Users/max/Desktop/claudespace/vpn-guard/clash-verge-guardian`

LaunchAgent：

- `local.clash-verge.guardian.plist`
- 原始安装位置：`/Users/max/Library/LaunchAgents/local.clash-verge.guardian.plist`
- 归档副本：`/Users/max/Desktop/claudespace/vpn-guard/local.clash-verge.guardian.plist`

状态和日志：

- 原始目录：`/Users/max/.local/state/clash-verge-guardian/`
- 归档副本：
  - `latest-report.json`
  - `guardian.log`
  - `launchd.out.log`
  - `launchd.err.log`

恢复交接记录：

- 原始位置：`/Users/max/.codex/recovery/reports/20260519-clash-verge-guardian.md`
- 归档副本：`/Users/max/Desktop/claudespace/vpn-guard/20260519-clash-verge-guardian.md`

## 守护逻辑

脚本入口：

```sh
/usr/bin/ruby /Users/max/.local/bin/clash-verge-guardian
```

常驻模式：

```sh
/usr/bin/ruby /Users/max/.local/bin/clash-verge-guardian --daemon
```

LaunchAgent 环境：

- label：`local.clash-verge.guardian`
- interval：每 `60` 秒检查一次当前节点
- subscription refresh interval：`604800` 秒，也就是 7 天
- latency threshold：`400ms`
- KeepAlive：开启
- RunAtLoad：开启

每轮守护会执行：

1. 确认 `/tmp/verge/verge-mihomo.sock` 可用。
2. 如果 API 不可用，启动或重启 Clash Verge。
3. 确认运行态是 `mode=rule`。
4. 确认 TUN 开启，默认参数为 `enable=true`、`stack=gvisor`、`auto-route=true`、`auto-detect-interface=true`、`dns-hijack=any:53`。
5. 到期时刷新订阅并重建 `clash-verge.yaml`。
6. 读取 `🚀 节点选择` 当前节点。
7. 对当前节点做三个真实 URL 测试：
   - Google：`https://www.google.com/generate_204`
   - OpenAI：`https://api.openai.com`
   - GitHub：`https://github.com`
8. 如果当前节点任一测试失败、超时，或平均/最大延迟超过 `400ms`，则找出 `🚀 节点选择` 组里所有名称包含 `日本` 的实际代理节点。
9. 并发测试所有日本节点，按成功次数、平均延迟、最大延迟排序。
10. 切换到最快节点，并关闭旧连接。

## 已完成验证

手动强制刷新订阅和测速：

```sh
/usr/bin/ruby /Users/max/.local/bin/clash-verge-guardian --once --refresh --select
```

结果：

- 订阅刷新成功：`2026-05-19T23:04:43+08:00`
- 运行配置重建成功
- runtime proxy count：`178`
- group count：`11`
- rule count：`3519`
- 运行态：`mode=rule`
- TUN：`true`
- TUN device：`utun4`
- mixed port：`7897`

第一次强制日本节点排序中发现并修复了一个脚本 bug：

- bug：URL 编码函数使用了 Ruby `tr("+", "%20")`，这不是字符串替换，而是字符映射。
- 影响：带空格和 emoji 的组名 `🚀 节点选择` 编码错误，API 切换请求没有真正切换目标组。
- 修复：改为 `gsub("+", "%20")`。
- 修复后手动 API 验证切换成功。

修复后再次强制日本节点排序：

- 选中节点：`上海联通转日本NTT6[Trojan][倍率:0.8]`
- 成功次数：`3/3`
- 平均延迟：`85ms`
- 最大延迟：`98ms`
- top 5：
  - `上海联通转日本NTT6[Trojan][倍率:0.8]`，avg `85ms`，max `98ms`
  - `上海联通转日本TE2[M][Trojan][倍率:0.8]`，avg `86ms`，max `109ms`
  - `江苏联通转日本TE3[M][Trojan][倍率:0.8]`，avg `95ms`，max `104ms`
  - `江苏联通转日本TE[M][Trojan][倍率:0.8]`，avg `95ms`，max `109ms`
  - `江苏联通转日本NTT[M][Trojan][倍率:0.8]`，avg `96ms`，max `104ms`

加载 LaunchAgent：

```sh
launchctl bootstrap gui/$(id -u) /Users/max/Library/LaunchAgents/local.clash-verge.guardian.plist
launchctl enable gui/$(id -u)/local.clash-verge.guardian
launchctl kickstart -k gui/$(id -u)/local.clash-verge.guardian
```

加载后状态：

- service：`gui/501/local.clash-verge.guardian`
- state：`running`
- pid：`18895`
- runs：`2`
- stderr log：空

常驻进程启动后自动普通检查：

- 当前节点：`上海联通转日本NTT6[Trojan][倍率:0.8]`
- `3/3` 真实 URL 测试成功
- 平均延迟约 `93-95ms`
- 最大延迟约 `108-126ms`
- 未超过 `400ms`，所以没有再次切换。

## 常用命令

查看 LaunchAgent：

```sh
launchctl print gui/$(id -u)/local.clash-verge.guardian
```

查看守护日志：

```sh
tail -f /Users/max/.local/state/clash-verge-guardian/guardian.log
```

手动跑一次普通检查：

```sh
/usr/bin/ruby /Users/max/.local/bin/clash-verge-guardian --once
```

手动强制订阅刷新和日本节点排序：

```sh
/usr/bin/ruby /Users/max/.local/bin/clash-verge-guardian --once --refresh --select
```

查看当前 Clash 模式和 TUN：

```sh
curl --unix-socket /tmp/verge/verge-mihomo.sock http://127.0.0.1/configs | jq '{mode:.mode,tun:.tun.enable,tun_device:.tun.device,mixed_port:.[\"mixed-port\"]}'
```

查看当前 `🚀 节点选择`：

```sh
curl --unix-socket /tmp/verge/verge-mihomo.sock 'http://127.0.0.1/proxies/%F0%9F%9A%80%20%E8%8A%82%E7%82%B9%E9%80%89%E6%8B%A9' | jq -r '.now'
```

停止守护：

```sh
launchctl bootout gui/$(id -u)/local.clash-verge.guardian
```

重新加载守护：

```sh
launchctl bootstrap gui/$(id -u) /Users/max/Library/LaunchAgents/local.clash-verge.guardian.plist
launchctl kickstart -k gui/$(id -u)/local.clash-verge.guardian
```

## 归档文件清单

当前目录：`/Users/max/Desktop/claudespace/vpn-guard`

- `vpn-guard.md`：本总文档
- `clash-verge-guardian`：守护脚本副本
- `local.clash-verge.guardian.plist`：LaunchAgent 副本
- `latest-report.json`：最近一次守护状态报告副本
- `guardian.log`：守护日志快照
- `launchd.out.log`：launchd stdout 快照
- `launchd.err.log`：launchd stderr 快照
- `20260519-clash-verge-guardian.md`：Codex 恢复交接记录副本

## GitHub 公开同步

已按要求将 `vpn-guard` 目录初始化为独立 Git 仓库，并同步到 GitHub 公开仓库：

- GitHub repo：`https://github.com/maxleolee-eng/vpn-guard`
- owner：`maxleolee-eng`
- visibility：`PUBLIC`
- default branch：`main`
- remote：`https://github.com/maxleolee-eng/vpn-guard.git`
- 初始公开提交：`a153c19262b879e44ba91eb008687a2dda0b1ef1`

发布前做过敏感信息检查：

- 没有加入 Clash 原始备份目录。
- 没有加入订阅 URL、节点密码、token、私钥或原始 Clash profile。
- 提交 author/committer 已改为 GitHub noreply：`248510948+maxleolee-eng@users.noreply.github.com`，避免公开本机局域网邮箱。

## 注意事项

- 归档不包含订阅 URL、节点密码、token 或任何密钥。
- 如果以后 Clash Verge 的 profile uid 或数据目录变化，需要重新读取真实 `profiles.yaml` 和 `/tmp/verge/verge-mihomo.sock` 状态。
- 如果 Clash Verge 未来版本提供稳定的外部 `update_profile` 命令，可以替换当前的直接下载和 runtime rebuild 路径。
