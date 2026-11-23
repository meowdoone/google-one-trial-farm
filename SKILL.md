---
name: google-one-trial-farm
description: 用 MuMu Pro 模拟器 + gost 代理 + google-one-tools 给新 Google 账号批量刷 Google AI Pro 12 个月免费试用。用户提到"刷号""下一个号""换IP""拉收银台""google one 试用"时使用。
---

# Google One AI Pro 试用批量刷号流程

目标：每个新 Google 账号领取 `g1.2tb.ai.subs_variant`（AI Pro 5TB，12 个月免费）。
每号一轮完整循环：换 IP → 删重建模拟器 → 等用户登录 → 验证 → 拉收银台。

## 关键路径与工具

- MuMu CLI：`/Applications/MuMuPlayer Pro.app/Contents/MacOS/mumu-cli`（子命令 show/info/create/delete/open/close/restart/config）
- adb：`~/.local/bin/adb`（软链到 MuMu 自带 adb）
- gost：`~/.local/bin/gost`（监听端口，注意沙箱权限）
- 脚本目录：`~/Desktop/Work/google-one-tools`，用 `python3.12`（frida==16.7.19；系统 python3 3.9 跑不动）
  - 先 `export PATH="$HOME/.local/bin:$PATH"`
  - `probe_env.py <serial>` 环境自检
  - `scan_all.py <serial>` 无风险扫描 offers
  - `launch_checkout.py --sku g1.2tb.ai.subs_variant --offer 12months-eft <serial>`

## IP 池与代理

- 池文件：`~/Desktop/iproyal-proxies.txt`，20 行，格式 `IP:12323:user:pass`
- 第 N 个号用第 N 个 IP
- 换 IP：
  ```bash
  pkill -f "gost -L http://:12333" && sleep 2
  ~/.local/bin/gost -L "http://:12333" -F "http://<user>:<pass>@<IP>:12323" > /tmp/gost.log 2>&1 &
  sleep 3; curl -s --max-time 15 -x http://127.0.0.1:12333 https://api.ipify.org
  ```
  出口 IP 必须等于目标 IP 才算切换成功。gost 日志在 `/tmp/gost.log`。

## 模拟器生命周期（每次全新实例）

```bash
CLI="/Applications/MuMuPlayer Pro.app/Contents/MacOS/mumu-cli"
"$CLI" close 0; sleep 3; "$CLI" delete 0
"$CLI" create --type phone
"$CLI" config 0 -s '{"vmRootEnable":true}'
"$CLI" open 0
```

然后后台等待开机（循环 120 次×5s）：
- 读 `~/Library/Application Support/com.netease.mumu.nemux-global/vms/0/setting/vm.json` 的 `adbPort`
- `adb connect 127.0.0.1:<port>`，直到 `getprop sys.boot_completed` 返回 1

**注意：adb 端口每次重建/重启都会变（5555/26624/26625 都出现过）**，永远以 vm.json 实测为准，出现 device offline 就重查端口。

开机后配置：
```bash
S=127.0.0.1:<port>
adb -s $S root          # MuMu 支持 adb root，确认 uid=0
adb -s $S shell settings put global http_proxy 10.0.2.2:12333
adb -s $S shell settings get global http_proxy   # 应显示 10.0.2.2:12333
```
（10.0.2.2 是 QEMU slirp 网关，指回宿主机 12333 的 gost。MuMu CLI/config 无代理设置项，只能这样走 Android 全局代理。）

## 验证与拉收银台

用户登录完成后（必须等用户在模拟器窗口手动登录，包括 Play 首次设置+付款方式）：

```bash
export PATH="$HOME/.local/bin:$PATH"; cd ~/Desktop/Work/google-one-tools
python3.12 probe_env.py $S        # 应全绿，hasJava:true
python3.12 scan_all.py $S         # 扫 offers，结果存 scan_result.json
python3.12 launch_checkout.py --sku g1.2tb.ai.subs_variant --offer 12months-eft $S
```

成功判据：Google One app 里显示 AI Pro 5TB、member since 当天、5TB 配额。

## 踩坑清单

1. **python 版本**：必须 `python3.12`，frida 必须 16.7.19（17.x 在 MuMu 上 Java bridge 坏）。
2. **responseCode 7 (BILLING_UNAVAILABLE)**：新号 Play 首次设置没走完/没加付款方式就会报。登录环节要提醒用户完成 Play 设置并加卡（卡号见本地卡片表，手动在模拟器 UI 输入）。
3. **launchBillingFlow code=0 但屏幕黑/无收银台**：看 logcat（LockToPortraitUiBuilderHostActivity、ProxyBillingActivity resultCode），必要时重拉一次。code=0 也可能已消耗试用资格，别无脑重复拉。
4. **adb 端口漂移** / device offline：重查 vm.json 的 adbPort 再 connect。
5. **截图**：`adb -s $S exec-out screencap -p > /tmp/x.png` 直接读图。
6. **限频**：真实账号每账号每小时模拟器会话不超过 3 次，出现人机验证立刻停。
7. scan 的 offer tokens 在不同账号间实测相同（账号无关），一次扫描结果可参考，但每个新号仍应跑一遍 scan_all 确认资格。
