#!/usr/bin/env bash
set -euo pipefail

# =========================
#  portopen (po) — 端口放行工具
#  - 交互菜单 + 命令行子命令
#  - IPv4/IPv6 可选
#  - Docker DOCKER-USER 转发链可选
#  - 持久化到 iptables-persistent
#  - /etc/portopen.conf 可配置（支持 init 初始化）
# =========================

C0="\033[0m"; C1="\033[1;32m"; C2="\033[1;36m"; CERR="\033[1;31m"
say(){ echo -e "${C1}>>${C0} $*"; }
warn(){ echo -e "${CERR}[WARN]${C0} $*"; }
pause(){ read -rp "$(echo -e ${C2}按回车继续...${C0})" _; }
need_root(){ if [[ $EUID -ne 0 ]]; then echo "请用 sudo 运行"; exit 1; fi; }
has(){ command -v "$1" >/dev/null 2>&1; }

# ---------- 默认配置（可在 /etc/portopen.conf 覆盖） ----------
CFG_FILE="/etc/portopen.conf"
DEFAULT_PROTOCOL="both"           # both|tcp|udp
DEFAULT_SOURCE="0.0.0.0/0"        # 来源 CIDR
ENABLE_IPV6="no"                  # yes|no
ENABLE_DOCKER="no"                # yes|no
QUICK_PORTS="443"                 # 一键放行固定端口（逗号或空格分隔）
SCAN_LISTEN="yes"                 # 一键时是否扫描本机监听端口
PORT_WHITELIST="443 8443 4443 2096 2095 2087 2083 2053 30000 31698 24981"

default_cfg_content() {
cat <<'CONF'
# portopen 配置
DEFAULT_PROTOCOL="both"        # both|tcp|udp
DEFAULT_SOURCE="0.0.0.0/0"     # 来源 CIDR
ENABLE_IPV6="no"               # yes|no
ENABLE_DOCKER="no"             # yes|no

# 一键放行的固定端口（逗号或空格分隔）
QUICK_PORTS="443 24981"

# 一键时是否扫描本机监听端口（仅白名单端口）
SCAN_LISTEN="yes"

# 监听扫描白名单（防止误开太多端口）
PORT_WHITELIST="443 8443 4443 2096 2095 2087 2083 2053 30000 31698 24981"
CONF
}

load_cfg(){
  if [[ -f "$CFG_FILE" ]]; then
    # shellcheck source=/etc/portopen.conf
    . "$CFG_FILE"
  fi
}

init_cfg(){
  need_root
  if [[ -f "$CFG_FILE" ]]; then
    say "已存在配置文件：$CFG_FILE"
  else
    mkdir -p "$(dirname "$CFG_FILE")"
    default_cfg_content > "$CFG_FILE"
    say "已创建默认配置：$CFG_FILE"
  fi
  if has nano; then nano "$CFG_FILE"
  elif has vi; then vi "$CFG_FILE"
  else
    warn "未检测到编辑器（nano/vi）。可稍后手动编辑：sudo nano $CFG_FILE"
  fi
}

# ---------- 持久化 ----------
ensure_persistent() {
  if ! has netfilter-persistent; then
    export DEBIAN_FRONTEND=noninteractive
    apt-get update -y >/dev/null 2>&1 || true
    apt-get install -y iptables-persistent netfilter-persistent >/dev/null 2>&1 || true
    systemctl enable netfilter-persistent >/dev/null 2>&1 || true
  fi
}
persist_rules() {
  if has netfilter-persistent; then
    netfilter-persistent save >/dev/null 2>&1 || true
  else
    mkdir -p /etc/iptables
    iptables-save  > /etc/iptables/rules.v4
    ip6tables-save > /etc/iptables/rules.v6 2>/dev/null || true
  fi
}

# ---------- iptables helpers ----------
insert_pos_v4(){ iptables -L INPUT --line-numbers 2>/dev/null | awk "/REJECT/ {print \$1; exit}"; }
insert_pos_v6(){ ip6tables -L INPUT --line-numbers 2>/dev/null | awk "/REJECT/ {print \$1; exit}"; }

add_rule_v4(){
  local proto="$1" port="$2" src="$3"
  iptables -C INPUT -p "$proto" -s "$src" --dport "$port" -j ACCEPT >/dev/null 2>&1 && { echo "[SKIP] v4 exists: $proto $port from $src"; return; }
  local pos; pos="$(insert_pos_v4)"; [[ -z "$pos" ]] && pos=1
  iptables -I INPUT "$pos" -p "$proto" -s "$src" --dport "$port" -j ACCEPT
  echo "[ADD ] v4 $proto $port from $src (pos $pos)"
}
del_rule_v4(){
  local proto="$1" port="$2" src="$3"
  iptables -C INPUT -p "$proto" -s "$src" --dport "$port" -j ACCEPT >/dev/null 2>&1 && {
    iptables -D INPUT -p "$proto" -s "$src" --dport "$port" -j ACCEPT
    echo "[DEL ] v4 $proto $port from $src"; return; }
  echo "[SKIP] v4 not found: $proto $port from $src"
}

add_rule_v6(){
  local proto="$1" port="$2"
  ! has ip6tables && return 0
  ip6tables -C INPUT -p "$proto" --dport "$port" -j ACCEPT >/dev/null 2>&1 && { echo "[SKIP] v6 exists: $proto $port"; return; }
  local pos; pos="$(insert_pos_v6)"; [[ -z "$pos" ]] && pos=1
  ip6tables -I INPUT "$pos" -p "$proto" --dport "$port" -j ACCEPT
  echo "[ADD ] v6 $proto $port (pos $pos)"
}
del_rule_v6(){
  local proto="$1" port="$2"
  ! has ip6tables && return 0
  ip6tables -C INPUT -p "$proto" --dport "$port" -j ACCEPT >/dev/null 2>&1 && {
    ip6tables -D INPUT -p "$proto" --dport "$port" -j ACCEPT
    echo "[DEL ] v6 $proto $port"; return; }
  echo "[SKIP] v6 not found: $proto $port"
}

# Docker: DOCKER-USER（转发方向）
ensure_docker_user(){
  if ! iptables -nL DOCKER-USER >/dev/null 2>&1; then
    iptables -N DOCKER-USER 2>/dev/null || true
    iptables -C FORWARD -j DOCKER-USER >/dev/null 2>&1 || iptables -I FORWARD 1 -j DOCKER-USER
  fi
}
add_rule_docker(){
  local proto="$1" port="$2"
  ensure_docker_user
  iptables -C DOCKER-USER -p "$proto" --dport "$port" -j ACCEPT >/dev/null 2>&1 || iptables -I DOCKER-USER 1 -p "$proto" --dport "$port" -j ACCEPT
  echo "[DOCKER] allow forward $proto dpt:$port"
}
del_rule_docker(){
  local proto="$1" port="$2"
  iptables -C DOCKER-USER -p "$proto" --dport "$port" -j ACCEPT >/dev/null 2>&1 && iptables -D DOCKER-USER -p "$proto" --dport "$port" -j ACCEPT || true
}

# ---------- 核心操作 ----------
scan_listen_ports(){
  local arr=()
  if has ss; then
    mapfile -t lines < <(ss -tulpenH 2>/dev/null | awk '{print $1,$5}')
    for l in "${lines[@]}"; do
      local proto="$(awk "{print \$1}" <<<"$l")"
      local addr="$(awk "{print \$2}" <<<"$l")"
      local p="${addr##*:}"
      [[ "$p" =~ ^[0-9]+$ ]] || continue
      for w in $PORT_WHITELIST; do
        if [[ "$p" == "$w" ]]; then arr+=("$p"); break; fi
      done
    done
  fi
  printf "%s\n" "${arr[@]}" | sort -n | uniq
}

do_add_ports(){
  local ports="$1" proto="$2" src="$3" ipv6="$4" docker="$5"
  IFS=', ' read -r -a A <<<"$ports"
  for p in "${A[@]}"; do
    [[ -z "$p" ]] && continue
    case "$proto" in
      tcp|udp)
        add_rule_v4 "$proto" "$p" "$src"
        [[ "$ipv6" == "1" ]] && add_rule_v6 "$proto" "$p"
        [[ "$docker" == "1" ]] && add_rule_docker "$proto" "$p"
      ;;
      both)
        add_rule_v4 tcp "$p" "$src"; add_rule_v4 udp "$p" "$src"
        [[ "$ipv6" == "1" ]] && { add_rule_v6 tcp "$p"; add_rule_v6 udp "$p"; }
        [[ "$docker" == "1" ]] && { add_rule_docker tcp "$p"; add_rule_docker udp "$p"; }
      ;;
    esac
  done
  persist_rules
}
do_del_ports(){
  local ports="$1" proto="$2" src="$3" ipv6="$4" docker="$5"
  IFS=', ' read -r -a A <<<"$ports"
  for p in "${A[@]}"; do
    [[ -z "$p" ]] && continue
    case "$proto" in
      tcp|udp)
        del_rule_v4 "$proto" "$p" "$src"
        [[ "$ipv6" == "1" ]] && del_rule_v6 "$proto" "$p"
        [[ "$docker" == "1" ]] && del_rule_docker "$proto" "$p"
      ;;
      both)
        del_rule_v4 tcp "$p" "$src"; del_rule_v4 udp "$p" "$src"
        [[ "$ipv6" == "1" ]] && { del_rule_v6 tcp "$p"; del_rule_v6 udp "$p"; }
        [[ "$docker" == "1" ]] && { del_rule_docker tcp "$p"; del_rule_docker udp "$p"; }
      ;;
    esac
  done
  persist_rules
}

# ---------- CLI ----------
usage(){
cat <<USAGE
Usage:
  portopen init                  # 初始化/编辑配置文件 /etc/portopen.conf
  portopen quick                 # 按配置一键放行
  portopen add <ports>   [--tcp|--udp|--both] [--source <CIDR>] [--ipv6] [--docker]
  portopen remove <ports>[--tcp|--udp|--both] [--source <CIDR>] [--ipv6] [--docker]
  portopen list
  portopen reload                # 重新加载配置文件
  portopen menu                  # 打开交互菜单（默认）

说明：
- <ports> 支持逗号或空格分隔，如：443,24981 或 "443 24981"
- 默认协议来自 DEFAULT_PROTOCOL（默认 both），来源 DEFAULT_SOURCE（默认 0.0.0.0/0）
- --ipv6 / --docker 可覆盖配置开关；命令行参数优先于配置文件
USAGE
}

print_cfg(){
  echo "配置文件: $CFG_FILE"
  echo "DEFAULT_PROTOCOL=$DEFAULT_PROTOCOL"
  echo "DEFAULT_SOURCE=$DEFAULT_SOURCE"
  echo "ENABLE_IPV6=$ENABLE_IPV6"
  echo "ENABLE_DOCKER=$ENABLE_DOCKER"
  echo "QUICK_PORTS=$QUICK_PORTS"
  echo "SCAN_LISTEN=$SCAN_LISTEN"
  echo "PORT_WHITELIST=$PORT_WHITELIST"
}

cli(){
  need_root; ensure_persistent; load_cfg
  local action="${1:-menu}"; [[ $# -gt 0 ]] && shift || true
  case "$action" in
    init) init_cfg ;;
    list)
      iptables -L INPUT -n --line-numbers
      echo "----"; iptables -nL DOCKER-USER 2>/dev/null || true
      ;;
    reload)
      load_cfg; print_cfg ;;
    quick)
      declare -A S
      IFS=', ' read -r -a FIX <<<"$QUICK_PORTS"
      for p in "${FIX[@]}"; do [[ -n "$p" ]] && S["$p"]=1; done
      if [[ "${SCAN_LISTEN,,}" == "yes" ]]; then
        while read -r p; do [[ -n "$p" ]] && S["$p"]=1; done < <(scan_listen_ports || true)
      fi
      local PORTS=""
      for k in "${!S[@]}"; do PORTS+="$k,"; done; PORTS="${PORTS%,}"
      if [[ -z "$PORTS" ]]; then warn "没有要放行的端口（检查 QUICK_PORTS 或 SCAN_LISTEN）"; exit 0; fi
      say "一键放行端口：$PORTS"
      local proto="$DEFAULT_PROTOCOL"
      local src="$DEFAULT_SOURCE"
      local ipv6="$([[ "${ENABLE_IPV6,,}" == "yes" ]] && echo 1 || echo 0)"
      local docker="$([[ "${ENABLE_DOCKER,,}" == "yes" ]] && echo 1 || echo 0)"
      do_add_ports "$PORTS" "$proto" "$src" "$ipv6" "$docker"
      iptables -L INPUT -n --line-numbers | sed -n "1,30p"
      ;;
    add|remove)
      local ports="${1:-}"; [[ -z "$ports" ]] && { usage; exit 1; }; shift || true
      local proto="$DEFAULT_PROTOCOL" src="$DEFAULT_SOURCE"
      local ipv6="$([[ "${ENABLE_IPV6,,}" == "yes" ]] && echo 1 || echo 0)"
      local docker="$([[ "${ENABLE_DOCKER,,}" == "yes" ]] && echo 1 || echo 0)"
      while [[ $# -gt 0 ]]; do
        case "$1" in
          --tcp) proto="tcp" ;;
          --udp) proto="udp" ;;
          --both) proto="both" ;;
          --source) shift; src="${1:-}";;
          --ipv6) ipv6=1 ;;
          --docker) docker=1 ;;
          *) break ;;
        esac; shift || true
      done
      [[ "$action" == "add" ]] && do_add_ports "$ports" "$proto" "$src" "$ipv6" "$docker" \
                               || do_del_ports "$ports" "$proto" "$src" "$ipv6" "$docker"
      iptables -L INPUT -n --line-numbers | sed -n "1,30p"
      ;;
    menu|*)
      menu ;;
  esac
}

# ---------- 交互菜单 ----------
menu(){
  load_cfg
  while true; do
    clear
    echo -e "${C2}PORTOPEN — 端口放行（po 快捷启动）${C0}"
    print_cfg
    echo
    echo "1) 初始化/编辑配置文件"
    echo "2) 一键放行（QUICK_PORTS + 可选监听扫描）"
    echo "3) 放行端口（TCP/UDP/IPv6/Docker 可选）"
    echo "4) 移除端口（TCP/UDP/IPv6/Docker 可选）"
    echo "5) 查看当前规则"
    echo "6) 重新加载配置文件"
    echo "0) 退出"
    read -rp "请选择：" m
    case "$m" in
      1) init_cfg; pause ;;
      2) cli quick; pause ;;
      3)
        read -rp "端口（逗号或空格分隔）： " p
        read -rp "协议 [both/tcp/udp]（默认 ${DEFAULT_PROTOCOL}）： " pr; pr="${pr:-$DEFAULT_PROTOCOL}"
        read -rp "来源CIDR（默认 ${DEFAULT_SOURCE}）： " s; s="${s:-$DEFAULT_SOURCE}"
        read -rp "同时写入 IPv6? [y/N]（当前 ${ENABLE_IPV6}）： " a; [[ "${a,,}" == "y" ]] && i6=1 || i6=$([[ "${ENABLE_IPV6,,}" == "yes" ]] && echo 1 || echo 0)
        read -rp "为 Docker 转发放行? [y/N]（当前 ${ENABLE_DOCKER}）： " b; [[ "${b,,}" == "y" ]] && dk=1 || dk=$([[ "${ENABLE_DOCKER,,}" == "yes" ]] && echo 1 || echo 0)
        do_add_ports "$p" "$pr" "$s" "$i6" "$dk"; iptables -L INPUT -n --line-numbers | sed -n "1,30p"; pause
        ;;
      4)
        read -rp "端口（逗号或空格分隔）： " p
        read -rp "协议 [both/tcp/udp]（默认 ${DEFAULT_PROTOCOL}）： " pr; pr="${pr:-$DEFAULT_PROTOCOL}"
        read -rp "来源CIDR（默认 ${DEFAULT_SOURCE}）： " s; s="${s:-$DEFAULT_SOURCE}"
        read -rp "同时移除 IPv6? [y/N]： " a; [[ "${a,,}" == "y" ]] && i6=1 || i6=0
        read -rp "同时移除 Docker 规则? [y/N]： " b; [[ "${b,,}" == "y" ]] && dk=1 || dk=0
        do_del_ports "$p" "$pr" "$s" "$i6" "$dk"; iptables -L INPUT -n --line-numbers | sed -n "1,30p"; pause
        ;;
      5) iptables -L INPUT -n --line-numbers | sed -n "1,60p"; echo "----"; iptables -nL DOCKER-USER 2>/dev/null || true; pause ;;
      6) load_cfg; say "已重新加载配置"; print_cfg; pause ;;
      0) exit 0 ;;
    esac
  done
}

# ---------- main ----------
if [[ "${1:-}" == "-h" || "${1:-}" == "--help" ]]; then usage; exit 0; fi
cli "$@"
