#!/usr/bin/env bash
# 出口机一键优化（最终稳定版）
# 说明：
# - 不包含 systemd 自启/自复制动作（避免递归/冲突）
# - 等待默认路由就绪；sysctl/egrep/ethtool 查询容错
# - 可反复执行（幂等）

set -euo pipefail
export LC_ALL=C
export PATH=/sbin:/usr/sbin:/bin:/usr/bin:$PATH

log() { echo -e "[$(date +%H:%M:%S)] $*"; }

# --- 0) 等默认路由就绪（最多 30s） ---
for _ in {1..30}; do
  if ip route get 1.1.1.1 &>/dev/null; then break; fi
  sleep 1
done

# --- 1) 网卡探测 ---
IFACE="$(ip route get 1.1.1.1 2>/dev/null | awk '{for(i=1;i<=NF;i++) if($i=="dev"){print $(i+1); exit}}')"
[ -z "${IFACE:-}" ] && IFACE="$(ip -o link show | awk -F': ' '$2!="lo"{print $2; exit}')"
[ -z "${IFACE:-}" ] && { echo "[ERR] 未找到可用网卡"; exit 1; }
log "出口机: 检测到网卡接口 -> ${IFACE}"

# --- 2) sysctl 参数（出口机专用） ---
cat >/etc/sysctl.d/99-net-export.conf <<'EOF'
net.core.default_qdisc = fq
# 初始用 bbrplus，若不可用将自动回退到 bbr
net.ipv4.tcp_congestion_control = bbrplus

# 缓冲与窗口
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728

# TCP 行为
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_low_latency = 1
net.ipv4.tcp_mtu_probing = 1
net.ipv4.tcp_sack = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_fin_timeout = 10
net.ipv4.tcp_keepalive_time = 120
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_no_metrics_save = 1
net.ipv4.tcp_slow_start_after_idle = 0

# 队列与 FD
fs.file-max = 1200000
net.core.somaxconn = 4096
net.core.netdev_max_backlog = 32768

# 内存策略（bytes 更可控）
vm.swappiness = 10
vm.dirty_background_bytes = 134217728   # 128MB
vm.dirty_bytes = 536870912              # 512MB
EOF

# --- 3) BBRplus 检测并回退 ---
if ! sysctl -n net.ipv4.tcp_available_congestion_control 2>/dev/null | grep -qw bbrplus; then
  sed -i 's/tcp_congestion_control = bbrplus/tcp_congestion_control = bbr/' /etc/sysctl.d/99-net-export.conf
  log "BBRplus 不可用 -> 回退为 BBR"
fi

# 仅加载这一份，容错处理（某些键在旧核不存在）
sysctl -p /etc/sysctl.d/99-net-export.conf || true

# --- 4) 队列 fq ---
tc qdisc del dev "$IFACE" root 2>/dev/null || true
tc qdisc add dev "$IFACE" root fq limit 20000 flow_limit 200
log "已应用 fq 队列到 ${IFACE}"

# --- 5) 网卡 offload（可选、平台不支持会忽略） ---
if command -v ethtool >/dev/null 2>&1; then
  ethtool -K "$IFACE" gro on gso on tso on 2>/dev/null || true
else
  log "ethtool 不存在，跳过 offload（可选安装）"
fi

# --- 6) 适度放大 TX 队列 ---
ip link set dev "$IFACE" txqueuelen 10000 2>/dev/null || true

# --- 7) limits 幂等写入 ---
grep -qE '^\* +soft +nofile +1048576$' /etc/security/limits.conf 2>/dev/null || echo '* soft nofile 1048576' >>/etc/security/limits.conf
grep -qE '^\* +hard +nofile +1048576$' /etc/security/limits.conf 2>/dev/null || echo '* hard nofile 1048576' >>/etc/security/limits.conf
grep -qE '^\* +soft +nproc +65535$'   /etc/security/limits.conf 2>/dev/null || echo '* soft nproc 65535' >>/etc/security/limits.conf
grep -qE '^\* +hard +nproc +65535$'   /etc/security/limits.conf 2>/dev/null || echo '* hard nproc 65535' >>/etc/security/limits.conf

# --- 8) conntrack：存在则调优，不存在则尝试加载再调优 ---
enable_and_tune_conntrack() {
  if [ -w /proc/sys/net/netfilter/nf_conntrack_max ]; then
    sysctl -w net.netfilter.nf_conntrack_max=1048576
    sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=1800
    sysctl -w net.netfilter.nf_conntrack_tcp_timeout_close_wait=60
    log "conntrack 参数已应用"
    return 0
  fi
  if command -v modprobe >/dev/null 2>&1; then
    modprobe nf_conntrack 2>/dev/null || true
    modprobe nf_conntrack_ipv4 2>/dev/null || true   # 旧核可能需要
    modprobe nf_conntrack_ipv6 2>/dev/null || true
  fi
  if [ -w /proc/sys/net/netfilter/nf_conntrack_max ]; then
    sysctl -w net.netfilter.nf_conntrack_max=1048576
    sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=1800
    sysctl -w net.netfilter.nf_conntrack_tcp_timeout_close_wait=60
    log "conntrack 模块加载并完成调优"
  else
    log "未发现 nf_conntrack 接口 -> 跳过（未做 NAT 或平台限制属正常）"
  fi
}
enable_and_tune_conntrack

# --- 9) 输出自检（容错） ---
echo
sysctl net.ipv4.tcp_congestion_control || true
tc qdisc show dev "$IFACE" 2>/dev/null | head -n 2 || true
if command -v ethtool >/dev/null 2>&1; then
  ethtool -k "$IFACE" 2>/dev/null | egrep 'tso|gso|gro' || true
fi
echo

log "✅ 出口机优化完成（幂等可重复执行）"

# 0) 停止风暴，清理失败状态
systemctl disable --now net-export-guard.service 2>/dev/null || true
systemctl reset-failed net-export-guard.service 2>/dev/null || true

# 1) 写入守护脚本到 /usr/bin（注意 shebang 有 #）
cat >/usr/bin/net-export-guard.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
export PATH=/sbin:/usr/sbin:/bin:/usr/bin

# 等默认路由就绪（最多 30s）
for _ in {1..30}; do ip route get 1.1.1.1 &>/dev/null && break; sleep 1; done

# 自动探测接口
IFACE="$(ip route get 1.1.1.1 2>/dev/null | awk '{for(i=1;i<=NF;i++) if($i=="dev"){print $(i+1); exit}}')"
[ -z "${IFACE:-}" ] && IFACE="$(ip -o link show | awk -F': ' '$2!="lo"{print $2; exit}')"
[ -z "${IFACE:-}" ] && exit 0

while true; do
  # fq 保活
  if ! tc qdisc show dev "$IFACE" 2>/dev/null | grep -qE '^\s*qdisc fq '; then
    tc qdisc del dev "$IFACE" root 2>/dev/null || true
    tc qdisc add dev "$IFACE" root fq limit 20000 flow_limit 200
    logger -t net-export-guard "reapplied fq on $IFACE"
  fi

  # 拥塞算法保活（优先 bbrplus，回退 bbr）
  CUR_CC="$(sysctl -n net.ipv4.tcp_congestion_control 2>/dev/null || echo '')"
  AVAIL="$(sysctl -n net.ipv4.tcp_available_congestion_control 2>/dev/null || echo '')"
  if [ "$CUR_CC" != "bbrplus" ] && echo "$AVAIL" | grep -qw bbrplus; then
    sysctl -w net.ipv4.tcp_congestion_control=bbrplus >/dev/null || true
    logger -t net-export-guard "set cc to bbrplus"
  elif [ "$CUR_CC" != "bbr" ] && echo "$AVAIL" | grep -qw bbr; then
    sysctl -w net.ipv4.tcp_congestion_control=bbr >/dev/null || true
    logger -t net-export-guard "set cc to bbr (fallback)"
  fi

  sleep 30
done
EOF

chmod +x /usr/bin/net-export-guard.sh
sed -i 's/\r$//' /usr/bin/net-export-guard.sh

# 2) 重写 unit，用 /usr/bin 路径（覆盖任何旧内容）
cat >/etc/systemd/system/net-export-guard.service <<'EOF'
[Unit]
Description=Guard fq queue and congestion control (export path)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
Environment=PATH=/sbin:/usr/sbin:/bin:/usr/bin
ExecStart=/usr/bin/net-export-guard.sh
Restart=always
RestartSec=5s
StandardOutput=journal
StandardError=inherit

[Install]
WantedBy=multi-user.target
EOF

# 3) 如有 drop-in 覆盖旧 ExecStart，把它删掉（最常见踩雷点）
if [ -d /etc/systemd/system/net-export-guard.service.d ]; then
  rm -f /etc/systemd/system/net-export-guard.service.d/*.conf
  rmdir --ignore-fail-on-non-empty /etc/systemd/system/net-export-guard.service.d 2>/dev/null || true
fi

# 4) 重新加载并启动
systemctl daemon-reload
systemctl enable --now net-export-guard.service

# 5) 验证：应该是 active (running)，并且 ExecStart 指向 /usr/bin
systemctl status net-export-guard.service --no-pager -l
systemctl cat net-export-guard.service | sed -n '1,120p'
