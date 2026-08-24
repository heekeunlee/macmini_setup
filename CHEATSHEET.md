# 치트시트

맥미니 서버를 굴리면서 실제로 자주 치게 되는 명령들.
전체 구축 과정은 [README.md](README.md) 참고.

---

## 접속

```bash
ssh mini                     # 맥북에서 맥미니로
exit                         # 나오기
```

접속이 안 될 때 순서대로 확인:

```bash
tailscale status             # 맥미니가 목록에 있나
ssh -v mini                  # 자세한 접속 로그
```

그래도 안 되면 **화면 공유**로 시도 (Finder 사이드바 또는 `Cmd+K` → `vnc://100.x.x.x`).
화면이 보이면 SSH 설정 문제, 안 보이면 네트워크·전원 문제입니다.

---

## tmux

```bash
tmux new -s agent            # 새 세션 만들고 들어가기
tmux ls                      # 돌고 있는 세션 목록
tmux attach -t agent         # 세션에 다시 붙기
tmux kill-session -t agent   # 세션 종료
```

세션 안에서 (`Ctrl+b` 를 누르고 손을 뗀 뒤 다음 키):

| 키 | 하는 일 |
|---|---|
| `d` | **떼어놓기** — 계속 실행됨 |
| `[` | 스크롤 모드 (`q`로 나감) |
| `c` | 새 창 |
| `n` / `p` | 다음 / 이전 창 |
| `?` | 전체 단축키 |

> `Ctrl+b` `d` 는 계속 실행, `exit`/`/exit` 는 종료. 이 둘을 헷갈리면 작업이 죽습니다.

---

## 에이전트

```bash
cd ~/agent && claude         # 작업 폴더에서 실행
claude --continue            # 최근 대화 이어서
claude --resume              # 목록에서 골라서
tail -f ~/agent/out.log      # 로그 실시간 (launchd 사용 시)
```

Claude Code 안에서:

| 입력 | 하는 일 |
|---|---|
| `!명령어` | 셸 명령 직접 실행 |
| `/exit` | 종료 |
| `Ctrl+C` 두 번 | 강제 종료 |

---

## 상태 확인

```bash
uptime                       # 며칠째 켜져 있나
pmset -g custom              # 절전 설정 (sleep 0, autorestart 1 이어야 함)
fdesetup status              # FileVault (FileVault is Off. 이어야 함)
tailscale status             # Tailscale 연결
tailscale ip -4              # 이 기기의 Tailscale 주소
ipconfig getifaddr en0       # 로컬 IP
scutil --get LocalHostName   # 호스트명
df -h /                      # 디스크 여유
top -o cpu                   # CPU 사용량 (q로 나감)
```

방화벽:

```bash
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
```

---

## 전원

```bash
sudo shutdown -r now         # 원격 재부팅
sudo shutdown -h now         # 원격 종료 (직접 켜러 가야 하니 주의)
```

재부팅 후 **tmux 세션은 사라집니다.** 다시 만드세요.

---

## Homebrew

```bash
brew update && brew upgrade  # 전체 업데이트
brew list                    # 설치된 것
brew services list           # 백그라운드 서비스 (tailscale 등)
brew cleanup                 # 오래된 버전 정리
```

---

## launchd

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.dasepa.agent.plist   # 등록
launchctl print gui/$(id -u)/com.dasepa.agent                                    # 상태
launchctl bootout gui/$(id -u)/com.dasepa.agent                                  # 해제
```

> `.plist` 를 고쳤으면 **`bootout` 후 다시 `bootstrap`**. 파일만 고치면 반영되지 않습니다.

시스템 데몬 재시작 (sshd 등):

```bash
sudo launchctl kickstart -k system/com.openssh.sshd
```

---

## 되돌리기

무언가 잘못됐을 때.

```bash
# 암호 로그인 다시 허용
sudo rm /etc/ssh/sshd_config.d/100-local.conf
sudo launchctl kickstart -k system/com.openssh.sshd

# 절전 설정 원복
sudo pmset -a sleep 10
sudo pmset -a autorestart 0

# ssh config 다시 쓰기 (맥북에서)
rm ~/.ssh/config
printf 'Host mini\n  HostName 100.x.x.x\n  User dasepa\n  ServerAliveInterval 60\n  ServerAliveCountMax 3\n' >> ~/.ssh/config

# 서버 지문이 바뀌었다고 할 때 (맥미니를 재설치한 경우 등)
ssh-keygen -R 100.x.x.x
```

---

## 최후의 수단

네트워크로 아무것도 안 될 때는 **모니터 + 유선 USB 키보드를 직접 연결**하는 것 외에 방법이 없습니다.

복구 모드(애플 실리콘): 전원 버튼을 **길게 눌러** 시동 옵션 화면까지 대기. 이건 원격으로 불가능합니다.

그래서 케이블은 맥미니 옆에 보관합니다.
