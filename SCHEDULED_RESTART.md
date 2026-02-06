# 定时重启说明（上海时间每日 04:00）

`cliproxyapi-installer` 现在会在**首次安装**时自动创建并启用下面两个 `systemd --user` 单元：

- `~/.config/systemd/user/cliproxyapi-restart.service`
- `~/.config/systemd/user/cliproxyapi-restart.timer`

默认目标是：**每天按上海时间（Asia/Shanghai）04:00 重启 `cliproxyapi.service`**。

## 当前默认行为

- **首次安装**：自动 `enable --now cliproxyapi-restart.timer`
- **升级安装**：刷新 timer / service 文件，但**不改变**当前启用状态
- **主服务未运行时**：到点后仍会执行 `restart cliproxyapi.service`，把服务拉起
- **自动启用失败时**：不阻断主安装，只输出修复命令

## 时间换算规则

为兼容不同版本的 `systemd`，安装器**不使用** `Timezone=Asia/Shanghai`。

实际写入规则如下：

- 如果 VPS 系统时区是 `UTC` 或 `Etc/UTC`，写入：`OnCalendar=*-*-* 20:00:00`
- 其他时区，写入：`OnCalendar=*-*-* 04:00:00`

> 说明：`20:00 UTC` 等于次日上海时间 `04:00`。

## 常用检查命令

```bash
# 查看 timer 是否启用
systemctl --user is-enabled cliproxyapi-restart.timer

# 查看下一次触发时间
systemctl --user list-timers --all | grep -F cliproxyapi-restart

# 查看 timer 状态
systemctl --user status cliproxyapi-restart.timer --no-pager

# 查看重启任务日志
journalctl --user -u cliproxyapi-restart.service -n 50 --no-pager

# 查看主服务日志
journalctl --user -u cliproxyapi.service -n 50 --no-pager
```

## 自动启用失败时的修复方式

如果安装时看到 `systemctl --user` 无法连接或 timer 启用失败，按下面顺序执行：

```bash
sudo loginctl enable-linger $(whoami)
systemctl --user daemon-reload
systemctl --user enable --now cliproxyapi-restart.timer
systemctl --user list-timers --all | grep -F cliproxyapi-restart
```

如果提示 `Failed to connect to bus`，通常需要重新登录当前 SSH 会话后再执行一次。

## 手动重建 / 覆盖 timer

如果你想手动重建定时器，可以直接执行：

```bash
bash -lc 'set -euo pipefail
install -d "$HOME/.config/systemd/user"

cat > "$HOME/.config/systemd/user/cliproxyapi-restart.service" <<EOF
[Unit]
Description=Restart CLIProxyAPIPlus (user)

[Service]
Type=oneshot
ExecStart=$(command -v systemctl) --user restart cliproxyapi.service
EOF

cat > "$HOME/.config/systemd/user/cliproxyapi-restart.timer" <<EOF
[Unit]
Description=Daily restart for CLIProxyAPIPlus at Asia/Shanghai 04:00

[Timer]
# 如果 VPS 时区是 UTC，请保持 20:00:00；否则可改为 04:00:00
OnCalendar=*-*-* 20:00:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now cliproxyapi-restart.timer
systemctl --user list-timers --all | grep -F cliproxyapi-restart'
```

如果你的 VPS **不是 UTC**，请把上面 timer 里的 `OnCalendar=*-*-* 20:00:00` 改成 `OnCalendar=*-*-* 04:00:00`。

## 取消定时重启

```bash
systemctl --user disable --now cliproxyapi-restart.timer
rm -f ~/.config/systemd/user/cliproxyapi-restart.timer
rm -f ~/.config/systemd/user/cliproxyapi-restart.service
systemctl --user daemon-reload
```

## 关于 VPS 重启后的生效

`systemd --user` 定时器通常要求用户会话可用。如果你希望 VPS 重启后即使不重新登录，也能继续按时执行，建议启用 lingering：

```bash
sudo loginctl enable-linger $(whoami)
loginctl show-user $(whoami) -p Linger
```
