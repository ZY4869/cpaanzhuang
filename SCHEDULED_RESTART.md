# 定时重启（按北京时间 04:00）

本仓库的 `cliproxyapi-installer` 会安装并生成 `systemd --user` 服务：`cliproxyapi.service`。  
如果你希望 **每天按北京时间 04:00 定时重启**，推荐用 `systemd timer` 来做定时任务。

## 关键换算（推荐做法）

很多 VPS 的系统时区默认是 `UTC`。在 **不修改 VPS 系统时区** 的前提下：

- 北京时间 `04:00` = `20:00 UTC`（前一天）

因此，定时器可以用下面的 `OnCalendar`：

- `OnCalendar=*-*-* 20:00:00`（当 VPS 系统时区为 `UTC` 时）

> 说明：有些系统的 `systemd` 不支持在 timer 里用 `Timezone=Asia/Shanghai`，此时最稳的是直接按 UTC 写死（如上）。

---

## 一键创建并启用（VPS 时区为 UTC）

直接在 VPS 里执行下面这一条命令，会创建/覆盖：

- `~/.config/systemd/user/cliproxyapi-restart.service`
- `~/.config/systemd/user/cliproxyapi-restart.timer`

并启用定时器（北京时间每天 04:00 重启一次）。

```bash
bash -lc 'set -euo pipefail
install -d "$HOME/.config/systemd/user"
cat > "$HOME/.config/systemd/user/cliproxyapi-restart.service" <<EOF
[Unit]
Description=Restart CLIProxyAPIPlus (user)

[Service]
Type=oneshot
ExecStart=systemctl --user restart cliproxyapi.service
EOF

cat > "$HOME/.config/systemd/user/cliproxyapi-restart.timer" <<EOF
[Unit]
Description=Scheduled restart for CLIProxyAPIPlus (user)

[Timer]
# VPS 时区为 UTC：北京时间 04:00 = UTC 20:00
OnCalendar=*-*-* 20:00:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now cliproxyapi-restart.timer
systemctl --user list-timers --all | grep -F cliproxyapi-restart'
```

## 脚本版本（保存为文件运行）

如果你更喜欢用脚本方式（你上面贴的脚本整理版），把下面内容保存为 `~/setup-cliproxyapi-restart-timer.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

# 用法：
#   ./setup-cliproxyapi-restart-timer.sh                      # 默认：按北京时间 04:00（VPS 时区为 UTC 时等价为 20:00 UTC）
#   ./setup-cliproxyapi-restart-timer.sh "*-*-* 20:00:00"     # 手动指定（适合 VPS 时区为 UTC）
#   ./setup-cliproxyapi-restart-timer.sh "Mon *-*-* 20:00:00" # 每周一（UTC 20:00 = 北京时间次日 04:00）

SYSTEM_TZ="$(timedatectl show -p Timezone --value 2>/dev/null || true)"

# 默认：在不改 VPS 系统时区（常见 UTC）的前提下，按“北京时间 04:00”执行
DEFAULT_ON_CALENDAR="*-*-* 04:00:00"
if [[ "$SYSTEM_TZ" == "UTC" || "$SYSTEM_TZ" == "Etc/UTC" ]]; then
  DEFAULT_ON_CALENDAR="*-*-* 20:00:00"
fi

ON_CALENDAR="${1:-$DEFAULT_ON_CALENDAR}"
SYSTEMCTL_BIN="$(command -v systemctl || true)"

if [[ -z "$SYSTEMCTL_BIN" ]]; then
  echo "ERROR: 找不到 systemctl（你的系统可能不是 systemd）。"
  exit 1
fi

mkdir -p "$HOME/.config/systemd/user"

if [[ ! -f "$HOME/.config/systemd/user/cliproxyapi.service" ]]; then
  echo "WARNING: 未发现 $HOME/.config/systemd/user/cliproxyapi.service"
  echo "         请先安装并启用 cliproxyapi.service。"
fi

cat > "$HOME/.config/systemd/user/cliproxyapi-restart.service" <<EOF
[Unit]
Description=Restart CLIProxyAPIPlus (user)

[Service]
Type=oneshot
ExecStart=$SYSTEMCTL_BIN --user restart cliproxyapi.service
EOF

cat > "$HOME/.config/systemd/user/cliproxyapi-restart.timer" <<EOF
[Unit]
Description=Scheduled restart for CLIProxyAPIPlus (user)

[Timer]
OnCalendar=$ON_CALENDAR
Persistent=true

[Install]
WantedBy=timers.target
EOF

if ! "$SYSTEMCTL_BIN" --user daemon-reload; then
  echo "ERROR: systemctl --user 无法连接（常见报错：Failed to connect to bus）。"
  echo "解决：先执行 sudo loginctl enable-linger \$(whoami)，然后重新登录 SSH 再运行本脚本。"
  exit 1
fi

"$SYSTEMCTL_BIN" --user enable --now cliproxyapi-restart.timer

echo "OK: 已启用定时重启：OnCalendar=$ON_CALENDAR"
"$SYSTEMCTL_BIN" --user list-timers --all | grep -F cliproxyapi-restart || true
```

然后执行：

```bash
chmod +x ~/setup-cliproxyapi-restart-timer.sh
~/setup-cliproxyapi-restart-timer.sh
```

## 验证与日志

```bash
# 看下一次触发（这里会按 VPS 的系统时区显示，若系统是 UTC，会显示 20:00 UTC）
systemctl --user list-timers --all | grep -F cliproxyapi-restart

# 看定时器状态
systemctl --user status cliproxyapi-restart.timer --no-pager

# 看每次“重启动作”的执行日志
journalctl --user -u cliproxyapi-restart.service -n 50 --no-pager

# 看主服务日志
journalctl --user -u cliproxyapi.service -n 50 --no-pager
```

## 取消定时重启（删除 timer）

```bash
systemctl --user disable --now cliproxyapi-restart.timer
rm -f ~/.config/systemd/user/cliproxyapi-restart.timer ~/.config/systemd/user/cliproxyapi-restart.service
systemctl --user daemon-reload
```

---

## 可选：让定时器在 VPS 重启后也能按时跑（推荐）

`systemd --user` 的定时器通常需要“用户登录”后才会启动。  
如果你希望 VPS 重启后无需再次登录也能按时执行，请开启 lingering：

```bash
sudo loginctl enable-linger $(whoami)
loginctl show-user $(whoami) -p Linger
```
