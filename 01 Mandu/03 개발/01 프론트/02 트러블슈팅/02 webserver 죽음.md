#  SSH 세션 종료시 Webserver 같이 종료됨
문제 파악 : SSH 세션 종료시 해당 계정에 nohup 실행중인 웹 서버가 같이 죽음
문제 원인
```bash
grep -E "KillUserProcesses" /etc/systemd/logind.conf
#KillUserProcesses=no 
```
Default가 `KillUserProcesses=yes` 로 동작중

## 해결 방안 
systemd 서비스로 실행하기 

### systemd 서비스로 실행 방법
#### 0) 준비: 로그 디렉토리 / 권한
```bash 
sudo mkdir -p /opt/logs
sudo chown -R hyun:hyun /opt/logs
```
#### 1) systemd 서비스 파일 생성 
```bash
sudo tee /etc/systemd/system/frontend.service > /dev/null <<'EOF'
[Unit]
Description=Next.js Frontend (prod) on 3300
After=network.target

[Service]
Type=simple
User=hyun
WorkingDirectory=/home/hyun/Mandu/frontend

Environment=NODE_ENV=production
Environment=PORT=3300
Environment=HOSTNAME=127.0.0.1

ExecStart=/usr/bin/npm run start -- -p 3300 -H 127.0.0.1

Restart=always
RestartSec=3

# 로그: 날짜 포함 파일로 저장
StandardOutput=append:/opt/logs/frontend_%Y%m%d.log
StandardError=append:/opt/logs/frontend_%Y%m%d.log

# 안전장치(권장)
KillSignal=SIGINT
TimeoutStopSec=20

[Install]
WantedBy=multi-user.target
EOF
```
#### 2) 서비스 적용 / 시작 
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now frontend.service
```