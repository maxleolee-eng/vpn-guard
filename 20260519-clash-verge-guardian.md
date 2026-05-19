# Clash Verge Guardian Handoff - 2026-05-19

## Goal

Create a persistent local guardian for the Mac's running Clash Verge Rev instance.

Required behavior:

- refresh the current remote subscription every 7 days and make the updated time visible to Verge UI
- monitor the currently selected node and treat delay above 400 ms, errors, or timeout as abnormal
- when abnormal, test nodes containing `日本` against three real URLs: Google, OpenAI, and GitHub
- rank Japan nodes by successful probe count and average delay, then switch `🚀 节点选择` to the fastest
- if the Clash core/API crashes, restart Clash Verge and repeat Japan-node selection
- keep Mihomo in TUN virtual-interface mode and Clash mode `rule`

## Files

- Guardian script: `/Users/max/.local/bin/clash-verge-guardian`
- LaunchAgent: `/Users/max/Library/LaunchAgents/local.clash-verge.guardian.plist`
- State and logs: `/Users/max/.local/state/clash-verge-guardian/`
- Current Verge data directory: `/Users/max/Library/Application Support/io.github.clash-verge-rev.clash-verge-rev`
- Mihomo Unix control socket: `/tmp/verge/verge-mihomo.sock`

## Implementation Notes

- The installed Clash Verge Rev app does not expose the internal `update_profile` Tauri command externally.
- Source inspection showed `verge://refresh-proxy-config` is not an update command for an existing profile; deep links import subscriptions and require a `url` query parameter.
- The guardian therefore reads the current remote profile URL from existing `profiles.yaml`, downloads the same profile without logging the URL, validates the downloaded Clash YAML, rebuilds `clash-verge.yaml`, reloads Mihomo through `PUT /configs?force=true`, and updates the profile `updated` timestamp.
- After subscription refresh, the guardian kickstarts the Clash Verge LaunchAgent so the UI process re-reads `profiles.yaml`.
- The script keeps `enable_tun_mode: true`, `enable_auto_launch: true`, runtime `mode: rule`, and runtime `tun.enable: true`.
- Existing isolated task `LOCAL_CLASH_VPN_Update` was left untouched because it uses separate local ports and does not manage the system Verge TUN instance.

## Verification

- Manual forced refresh succeeded at `2026-05-19T23:04:43+08:00`.
- Refreshed runtime contained `178` proxies, `11` groups, and `3519` rules.
- Current runtime after reload: `mode=rule`, `tun=true`, `tun_device=utun4`, `mixed_port=7897`.
- Manual forced Japan-node ranking selected `上海联通转日本NTT6[Trojan][倍率:0.8]`.
- Winning manual probe result: `3/3` successes, average `85 ms`, max `98 ms`.
- Post-launchd daemon cycle checked the same current node with `3/3` successes, average `93 ms`, max `108 ms`, so no switch was needed.
- LaunchAgent loaded as `gui/501/local.clash-verge.guardian`, state `running`, PID `18895`.

## Operations

Run one immediate check:

```sh
/usr/bin/ruby /Users/max/.local/bin/clash-verge-guardian --once
```

Force subscription refresh and Japan-node ranking:

```sh
/usr/bin/ruby /Users/max/.local/bin/clash-verge-guardian --once --refresh --select
```

Check LaunchAgent:

```sh
launchctl print gui/$(id -u)/local.clash-verge.guardian
```

Tail guardian log:

```sh
tail -f /Users/max/.local/state/clash-verge-guardian/guardian.log
```
