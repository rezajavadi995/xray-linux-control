<p align="center">
  <strong>English</strong> | <a href="README.fa.md">فارسی</a>
</p>

# XRAY / V2RAY Control Interface (Linux)

<p align="center"> <b>
  ⭐️ If you like this project, please give it a star on GitHub! ⭐️
</b>
</p>

A lightweight, interactive **Bash control interface** for managing **Xray / V2Ray**
on modern Linux systems using `systemd`.

This tool provides a clear terminal UI to start/stop the core service, manage SOCKS
proxy tunneling (terminal or system-wide), inspect real vs tunneled IP addresses,
and verify tunnel health with geo-location awareness.

Designed for **real-world usage**, not demos.

---

## ✨ Features

- Automatic detection of Xray or V2Ray core
- Automatic detection of `config.json` location
- SOCKS port detection from configuration
- Start / Stop core service
- Enable proxy for current terminal session
- Enable system-wide proxy
- Network status overview:
  - Real IP
  - Tunnel IP
  - Country & city (GeoIP)
- Tunnel health check
- One-click V2Ray installation
- Import `config.json` directly from stdin
- Clean, colorized TUI (Text User Interface)

---

## 🧠 Requirements

This script is **not universal by design**.  
It targets standard Linux systems with GNU tooling and `systemd`.

### Supported Systems
- Ubuntu / Debian / Kali Linux
- Fedora
- Arch Linux
- Rocky / Alma Linux

### Required Components
- Init system: **systemd**
- GNU userland tools:
  - `bash`
  - `curl`
  - `grep` (GNU, with `-P` support)
  - `sed`
  - `timeout`
- `sudo` access

### Not Supported
- Alpine Linux (BusyBox)
- OpenRC / runit based systems
- Minimal Docker images
- WSL without systemd

---

## 🚀 Installation



```bash
sudo tee /usr/local/bin/Xray > /dev/null <<'EOF'
#!/usr/bin/env bash

SOCKS_HOST="127.0.0.1"
DEFAULT_PORT="10808"
TIMEOUT=8

RED="\033[1;31m"
GREEN="\033[1;32m"
YELLOW="\033[1;33m"
BLUE="\033[1;34m"
CYAN="\033[1;36m"
MAGENTA="\033[1;35m"
GRAY="\033[1;90m"
NC="\033[0m"

CORE=""
CONFIG=""
PORT=""
REAL_IP=""
SOCKS_IP=""
COUNTRY=""
CITY=""
TERMINAL_TUNNEL="OFF"

pause() {
    echo
    read -rp "⏎ Enter برای ادامه..."
}

detect_core() {
    if systemctl list-units --type=service | grep -q xray.service; then
        CORE="xray"
    elif systemctl list-units --type=service | grep -q v2ray.service; then
        CORE="v2ray"
    elif command -v xray >/dev/null 2>&1; then
        CORE="xray"
    elif command -v v2ray >/dev/null 2>&1; then
        CORE="v2ray"
    else
        CORE=""
    fi
}

detect_config() {
    local paths=(
        "/usr/local/etc/xray/config.json"
        "/etc/xray/config.json"
        "/usr/local/etc/v2ray/config.json"
        "/etc/v2ray/config.json"
    )
    for p in "${paths[@]}"; do
        [[ -f "$p" && -s "$p" ]] && CONFIG="$p" && return
    done
    CONFIG=""
}

detect_port() {
    if [[ -n "$CONFIG" ]]; then
        PORT=$(grep -oE '"port"[[:space:]]*:[[:space:]]*[0-9]+' "$CONFIG" | head -n1 | grep -oE '[0-9]+')
    fi
    [[ -z "$PORT" ]] && PORT="$DEFAULT_PORT"
}

service_active() {
    [[ -n "$CORE" ]] && systemctl is-active --quiet "$CORE"
}

start_service() {
    [[ -z "$CORE" ]] && echo -e "${YELLOW}⚠ Core not detected${NC}" || sudo systemctl start "$CORE"
}

stop_service() {
    [[ -n "$CORE" ]] && sudo systemctl stop "$CORE" 2>/dev/null && sudo pkill -f "$CORE" 2>/dev/null
}

enable_terminal_proxy() {
    export http_proxy="socks5h://$SOCKS_HOST:$PORT"
    export https_proxy="socks5h://$SOCKS_HOST:$PORT"
    export all_proxy="socks5h://$SOCKS_HOST:$PORT"
}

disable_terminal_proxy() {
    unset http_proxy https_proxy all_proxy
}

enable_system_proxy() {
    sudo sed -i '/_proxy=/d' /etc/environment
    echo "http_proxy=\"socks5h://$SOCKS_HOST:$PORT\"" | sudo tee -a /etc/environment >/dev/null
    echo "https_proxy=\"socks5h://$SOCKS_HOST:$PORT\"" | sudo tee -a /etc/environment >/dev/null
    echo "all_proxy=\"socks5h://$SOCKS_HOST:$PORT\"" | sudo tee -a /etc/environment >/dev/null
    sudo systemctl restart systemd-logind
}

disable_system_proxy() {
    sudo sed -i '/_proxy=/d' /etc/environment
    sudo systemctl restart systemd-logind
}

detect_terminal_tunnel() {
    timeout 4 curl -s --socks5-hostname "$SOCKS_HOST:$PORT" https://api.ipify.org >/dev/null \
        && TERMINAL_TUNNEL="ON" || TERMINAL_TUNNEL="OFF"
}

collect_network_info() {
    REAL_IP=$(timeout $TIMEOUT curl -s https://api.ipify.org)

    GEO_JSON=$(timeout $TIMEOUT curl -s --socks5-hostname "$SOCKS_HOST:$PORT" https://api.ip.sb/geoip)

    SOCKS_IP=$(echo "$GEO_JSON" | grep -oP '"ip"\s*:\s*"\K[^"]+')
    COUNTRY=$(echo "$GEO_JSON" | grep -oP '"country"\s*:\s*"\K[^"]+')
    CITY=$(echo "$GEO_JSON" | grep -oP '"city"\s*:\s*"\K[^"]+')

    detect_terminal_tunnel
}

network_test() {
    clear
    collect_network_info

    echo -e "${CYAN}════════ NETWORK STATUS ════════${NC}"
    echo -e " Tunnel IP  : ${GREEN}${SOCKS_IP:-OFF}${NC}"
    echo -e " Location   : ${YELLOW}${CITY:-Unknown}, ${COUNTRY:-Unknown}${NC}"
    echo -e " Real IP    : ${BLUE}${REAL_IP:-Unknown}${NC}"
    echo
    service_active && echo -e " Service    : ${GREEN}ACTIVE${NC}" || echo -e " Service    : ${RED}STOPPED${NC}"
    [[ "$TERMINAL_TUNNEL" == "ON" ]] && echo -e " Terminal   : ${GREEN}TUNNELED${NC}" || echo -e " Terminal   : OFF"
    grep -q "_proxy=" /etc/environment && echo -e " System     : ${GREEN}TUNNELED${NC}" || echo -e " System     : OFF"
}

install_v2ray_script() {
    clear
    echo -e "${CYAN}⬇ Installing Core${NC}"
    sudo bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
}

import_config_json() {
    clear
    echo -e "${CYAN}📥 Paste config.json (Ctrl+D to finish)${NC}"
    TMP=$(mktemp)
    cat > "$TMP"
    detect_core
    [[ -z "$CORE" ]] && echo -e "${RED}✖ Core not installed${NC}" && rm -f "$TMP" && return
    TARGET="/usr/local/etc/$CORE/config.json"
    sudo mkdir -p "$(dirname "$TARGET")"
    sudo mv "$TMP" "$TARGET"
    echo -e "${GREEN}✔ Config saved → $TARGET${NC}"
}

menu() {
    detect_core
    detect_config
    detect_port
    collect_network_info

    clear
    echo -e "${GRAY}════════════════════════════════════════════${NC}"
    echo -e "${MAGENTA}   XRAY / V2RAY CONTROL INTERFACE${NC}"
    echo -e "${GRAY}════════════════════════════════════════════${NC}"

    echo -e " CORE        : ${YELLOW}${CORE:-None}${NC}"
    echo -e " CONFIG      : ${YELLOW}${CONFIG:-None}${NC}"
    echo -e " SOCKS       : ${YELLOW}$SOCKS_HOST:$PORT${NC}"
    echo -e " REAL IP     : ${BLUE}${REAL_IP:-Unknown}${NC}"
    echo -e " TUNNEL IP   : ${GREEN}${SOCKS_IP:-OFF}${NC}"
    echo -e " LOCATION    : ${YELLOW}${CITY:-Unknown}, ${COUNTRY:-Unknown}${NC}"

    service_active && SVC="${GREEN}ON${NC}" || SVC="${RED}OFF${NC}"
    [[ "$TERMINAL_TUNNEL" == "ON" ]] && TPROXY="${GREEN}ON${NC}" || TPROXY="${RED}OFF${NC}"
    grep -q "_proxy=" /etc/environment && SPROXY="${GREEN}ON${NC}" || SPROXY="${RED}OFF${NC}"

    echo -e " SERVICE     : $SVC | TERMINAL : $TPROXY | SYSTEM : $SPROXY"

    echo -e "${GRAY}────────────────────────────────────────────${NC}"
    echo " 1) ▶ Start Core Service"
    echo " 2) ■ Stop Core Service"
    echo " 3) Enable Terminal Tunnel"
    echo " 4) Disable Terminal Tunnel"
    echo " 5) Enable System Tunnel"
    echo " 6) Disable System Tunnel"
    echo " 7) Network / Geo / Status Test"
    echo " 8) Install Core"
    echo " 9) Import config.json"
    echo " 0) Exit"
    echo
    read -rp "➤ Select: " c

    case "$c" in
        1) start_service ;;
        2) stop_service ;;
        3) enable_terminal_proxy ;;
        4) disable_terminal_proxy ;;
        5) enable_system_proxy ;;
        6) disable_system_proxy ;;
        7) network_test ;;
        8) install_v2ray_script ;;
        9) import_config_json ;;
        0) exit 0 ;;
    esac
    pause
}

while true; do menu; done
EOF

```
---
## Make it executable:
```bash
sudo chmod +x /usr/local/bin/Xray
```
---
## ▶ Running the Program
```bash
Xray

