# vnc-firefox

```
#打开远程桌面
name: VNC Server with noVNC and Cloudflared

on:
  workflow_dispatch:  # Manual trigger
  
jobs:
  setup-vnc:
    runs-on: ubuntu-latest
    steps:
      - name: Install VNC and Desktop Environment
        run: |
          sudo apt-get update
          sudo DEBIAN_FRONTEND=noninteractive apt-get install -y \
            xfce4 \
            xfce4-goodies \
            tightvncserver \
            novnc \
            python3 \
            python3-pip \
            git

      - name: Configure VNC
        run: |
          mkdir -p ~/.vnc
          echo "password" | vncpasswd -f > ~/.vnc/passwd
          chmod 600 ~/.vnc/passwd
          echo '#!/bin/bash
          xrdb $HOME/.Xresources
          startxfce4 &' > ~/.vnc/xstartup
          chmod +x ~/.vnc/xstartup

      - name: Start VNC Server
        run: |
          vncserver :1 -geometry 1280x800 -depth 24
          sleep 5  # Wait for VNC server to start properly

      - name: Install and Setup noVNC
        run: |
          git clone https://github.com/novnc/noVNC.git
          cd noVNC
          ./utils/novnc_proxy --vnc localhost:5901 &
          sleep 5  # Wait for noVNC to start

      - name: Install Cloudflared
        run: |
          # Download and install Cloudflared
          wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
          sudo dpkg -i cloudflared-linux-amd64.deb

      - name: Start Cloudflared
        run: |
          # Start Cloudflared tunnel without authentication
          cloudflared tunnel --url http://localhost:6080 &
          sleep 5  # Wait for tunnel to establish
          
          # Print the tunnel URL
          echo "Cloudflared tunnel is running. Check the logs above for the tunnel URL."

      - name: Keep Alive and Monitor
        run: |
          echo "VNC Server started with:"
          echo "noVNC port: 6080"
          echo "VNC port: 5901"
          echo "Default VNC password: password"
          
          # Keep the action running and monitor services
          while true; do
            echo "=== Status Update $(date) ==="
            ps aux | grep vnc
            ps aux | grep cloudflared
            ps aux | grep novnc
            sleep 300
          done
```

```
#打开远程浏览器
#docker启动参数 5800 是https网页链接打开端口 ，5900 vnc端口，启动这个就是把5900已经打开到5800了
name: Firefox VNC Server

on:
  workflow_dispatch:
  
jobs:
  setup-vnc:
    runs-on: ubuntu-latest
    steps:
      - name: Install dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y \
            apt-transport-https \
            ca-certificates \
            curl \
            gnupg \
            lsb-release

      - name: Install Docker using official script
        run: |
          curl -fsSL https://get.docker.com -o get-docker.sh
          sh get-docker.sh

      - name: Run Firefox in Docker
        run: |
          docker run -d --name firefox \
            -e TZ=Asia/Hong_Kong \
            -e DISPLAY_WIDTH=1920 \
            -e DISPLAY_HEIGHT=1080 \
            -e KEEP_APP_RUNNING=1 \
            -e ENABLE_CJK_FONT=1 \
            -e VNC_PASSWORD=admin \
            -p 5800:5800 -p 5900:5900 \
            -v /data/firefox/config:/config:rw \
            --shm-size 2g \
            jlesage/firefox

      - name: Install Cloudflared
        run: |
          wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
          sudo dpkg -i cloudflared-linux-amd64.deb

      - name: Start Cloudflared
        run: |
          # 转发 noVNC 端口 5800
          cloudflared tunnel --url http://localhost:5800 &
          sleep 5
          
          # 获取并显示隧道URL
          echo "等待Cloudflared隧道URL..."
          sleep 10
          ps aux | grep cloudflared

      - name: Keep Alive and Monitor
        run: |
          echo "=== Firefox VNC Server Information ==="
          echo "noVNC端口: 5800"
          echo "VNC端口: 5900"
          echo "VNC密码: admin"
          echo "说明: 使用浏览器连接时，主机地址使用上面cloudflared生成的域名"
          echo "端口使用: 复制cloudflared URL后的端口号"
          
          while true; do
            echo "=== Status Update $(date) ==="
            docker ps
            sleep 300
          done
```
