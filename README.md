# 맥미니 헤드리스 서버 구축

> 언박싱 직후 상태에서 출발해, **모니터·키보드 없이 24시간 돌아가는 SSH 서버 겸 AI 에이전트 호스트**를 만들기까지의 전체 기록.

맥북에서 `ssh mini` 한 줄로 접속하고, 에이전트가 재부팅과 정전을 넘기고도 살아 있는 상태가 목표입니다.
실제로 세팅하면서 막혔던 지점과 해결책을 그대로 반영했습니다 → [삽질 기록](#삽질-기록)

| | |
|---|---|
| 총 소요 | 2~3시간 (Homebrew 설치 대기 포함) |
| 단계 | 11단계 (0~10) |
| 필요한 것 | 맥북, 맥미니, 랜선 |
| 검증 환경 | Apple Silicon Mac mini, macOS Tahoe |

---

## 목차

| 단계 | 내용 | 끝나면 |
|---|---|---|
| [0](#0-출발선-정리) | 출발선 정리 | 유선 연결, 초기 셋업 완료 |
| [1](#1-절대-잠들지-않게-만들기) | 절전 완전 해제 | 맥미니가 절대 안 잠듦 |
| [2](#2-이름-붙이고-원격-접속-열기) | 호스트명 + SSH 켜기 | 맥북에서 접속 가능 |
| [3](#3-암호-없애기-ssh-키) | SSH 키 등록 | `ssh mini` 한 줄, 무암호 |
| [4](#4-개발-환경-얹기) | Homebrew + 도구 | 개발 서버 |
| [5](#5-집-밖에서도-접속하기) | Tailscale | 어디서든 접속 |
| [6](#6-에이전트-올리기) | Claude Code | 에이전트 실행 |
| [7](#7-ssh를-끊어도-계속-돌게-하기) | tmux / launchd | 창을 닫아도 계속 실행 |
| [8](#8-재부팅과-정전에서-스스로-돌아오기) | 자동 복구 | 무인 서버 |
| [9](#9-보안-마무리) | 방화벽, 암호 로그인 차단 | 키를 가진 기기만 |
| [10](#10-최종-점검) | 최종 점검 | 완성 |

---

## 읽는 법

**단계 순서에는 실제 의존 관계가 있습니다.** 원격 로그인을 켜야 SSH가 되고, Homebrew가 있어야 도구를 깔고, Node가 있어야 에이전트가 돕니다. 건너뛰지 마세요.

명령어 블록 위의 주석은 **어느 기계에서 치는 명령인지** 나타냅니다. 가장 헷갈리는 부분이니 매번 확인하세요.

프롬프트만 보면 현재 위치를 알 수 있습니다.

| 프롬프트 | 위치 |
|---|---|
| `dasepa@macbook ~ %` | 맥북 |
| `dasepa@mini ~ %` | 맥미니 (SSH로 들어간 상태) |

이 문서는 계정 이름 `dasepa`, 호스트명 `mini` 기준입니다. 다르게 지었다면 해당 부분을 바꿔서 쓰세요.

---

## 0. 출발선 정리

### 랜선을 꽂으세요

서버로 쓸 거라면 Wi-Fi 말고 **유선 이더넷**을 강력히 권합니다. 무선은 절전에서 연결이 끊기거나 공유기 재부팅 후 재연결이 늦어서, "분명 켜뒀는데 접속이 안 되는" 상황의 대부분을 차지합니다.

유선이 확인됐다면 **Wi-Fi는 아예 꺼두세요.** 켜두면 유선이 잠깐 끊길 때 맥이 조용히 Wi-Fi로 갈아타면서 IP가 바뀌고, 그러면 접속이 갑자기 안 됩니다.

```bash
# 맥미니에서 — 어느 장치가 이더넷인지 확인
networksetup -listallhardwareports

# 실제로 인터넷이 나가는 통로 확인
route get default | grep interface
```

### 초기 셋업에서의 선택

| 항목 | 권장 | 이유 |
|---|---|---|
| 계정 이름 | 짧은 영문 소문자 (`dasepa`) | 홈 폴더 경로가 됨. 나중에 변경이 매우 번거로움 |
| Apple ID | **기존 계정** | 새로 만들면 기기 간 아무것도 공유 안 됨 |
| iCloud 키체인 | **건너뛰기** | 아래 참고 |
| Siri / 화면 사용 시간 / 분석 공유 | 끄기 | 서버에 불필요 |
| FileVault | **끄기** | [8단계](#8-재부팅과-정전에서-스스로-돌아오기) 참고. 나중에 해제하는 것보다 처음부터 안 켜는 게 빠름 |

> **iCloud 키체인을 건너뛰는 이유**
> 맥북·아이폰의 모든 비밀번호와 패스키를 이 기계에도 동기화하는데, 자동 로그인을 켜면 전원이 들어오는 순간 키체인 잠금까지 함께 풀립니다. 방치된 기계가 모든 계정 암호를 열린 채로 들고 있는 셈이 됩니다.
> 건너뛰어도 **로컬 키체인은 그대로 작동합니다** — Claude Code 토큰, git 자격 증명, SSH 키 모두 정상. iCloud를 통한 *기기 간 동기화*만 안 하는 것이고, `시스템 설정 › Apple 계정 › iCloud › 암호 및 키체인`에서 언제든 되돌릴 수 있습니다.

---

## 1. 절대 잠들지 않게 만들기

맥은 한가하면 잠들고, **잠든 맥은 SSH를 받지 않습니다.** 이 단계를 건너뛰면 나머지가 전부 무의미합니다.

### GUI

`시스템 설정 › 에너지` 에서 세 개를 켭니다.

- 디스플레이가 꺼져 있을 때 자동으로 잠자기 방지
- 정전 후 자동으로 시동
- 네트워크 접속 시 깨우기

### 터미널

GUI 스위치가 놓치는 항목이 있어서 명령어로 한 번 더 못 박습니다.

```bash
# 맥미니에서 직접
sudo pmset -a sleep 0          # 시스템 잠자기 끄기
sudo pmset -a disksleep 0      # 디스크 잠자기 끄기
sudo pmset -a womp 1           # 네트워크로 깨우기
sudo pmset -a autorestart 1    # 정전 후 자동 재시작
```

<details>
<summary>명령어 뜯어보기</summary>

| 조각 | 뜻 |
|---|---|
| `sudo` | **s**uper**u**ser **do** — 관리자 권한으로 실행 |
| `pmset` | **P**ower **M**anagement **set**tings |
| `-a` | **a**ll — 모든 전원 상태(배터리·어댑터·UPS)에 적용 |

- `sleep 0` / `disksleep 0` — 시간 단위는 **분**이고, `0`은 "0분 뒤"가 아니라 **"절대 안 함(never)"** 을 뜻합니다. pmset의 특별한 값이라 반대로 해석하기 쉬운 부분입니다.
- `womp` — **W**ake **O**n **M**agic **P**acket. 여기서 `1`은 켜기.
- `autorestart 1` — 전기가 돌아오면 자동으로 다시 켜짐.

되돌리려면 값만 바꿔서 다시 치면 됩니다: `sudo pmset -a sleep 10`

</details>

**확인**

```bash
pmset -g custom | grep -E "sleep|womp|autorestart"
```

`sleep 0`, `autorestart 1` 이면 성공. (`-g`는 **g**et — 읽기만 하니 `sudo`가 필요 없습니다.)

---

## 2. 이름 붙이고 원격 접속 열기

### 이름을 짧게

기본 이름은 `이희근의 Mac mini` 같은 형태라 주소로 쓰기 나쁩니다. `mini`로 바꾸면 `mini.local` 로 접속할 수 있습니다.

```bash
# 맥미니에서 직접
sudo scutil --set ComputerName mini
sudo scutil --set LocalHostName mini
sudo scutil --set HostName mini
```

<details>
<summary>왜 이름을 세 번이나 바꾸나</summary>

맥에는 이름이 세 개 있고, 서로 다른 계층에서 쓰입니다.

| 항목 | 어디서 쓰이나 | 규칙 |
|---|---|---|
| `ComputerName` | 사람에게 보이는 이름. AirDrop, 공유 목록 | 한글·공백·이모지 가능 |
| `LocalHostName` | **네트워크 주소.** `mini.local` 의 앞부분 | 영문·숫자·하이픈만 |
| `HostName` | 유닉스 내부 호스트명. 터미널 프롬프트에 표시 | 영문·숫자·하이픈 |

맥이 유닉스 위에 애플 고유 기술(Bonjour)을 얹은 구조라 각 층이 자기 이름 체계를 갖고 있습니다.

**GUI에서 이름만 바꾸면 안 되는 이유**가 여기 있습니다. `ComputerName`만 바뀌고, `LocalHostName`은 공백·한글을 못 쓰니 맥이 `ihuigeun-ui-MacBookPro` 같은 형태로 자동 변환하며, `HostName`은 아예 비어 있는 채로 남습니다.

`.local` 접미사는 **Bonjour(mDNS)** 가 붙여주는 것입니다. DNS 서버 없이 "mini 있어?" 하고 네트워크에 방송을 뿌리는 방식이라, 방송이 닿는 **같은 공유기 안에서만** 통합니다. [5단계](#5-집-밖에서도-접속하기)에서 주소를 바꾸는 이유입니다.

</details>

### 원격 접속 켜기

`시스템 설정 › 일반 › 공유` 에서 두 개를 켭니다.

- **원격 로그인** — 이게 SSH 서버입니다. 켜면 아래에 접속 주소가 표시됩니다
- **화면 공유** — 비상구. SSH가 막혔을 때 맥북에서 맥미니 화면을 볼 수 있습니다. 헤드리스로 갈 거라 **이게 없으면 나중에 손쓸 방법이 없습니다**

원격 로그인 옆 ⓘ 에서 **"다음 사용자만 접근 허용"** 을 고르고 본인 계정을 넣으세요.

> 이 목록은 **기기가 아니라 맥미니의 사용자 계정**을 고르는 것입니다. 맥북에서 접속하든 아이폰에서 접속하든 결국 `dasepa` 계정으로 로그인하는 것이라, "맥북"이나 "아이폰"을 추가할 수는 없습니다.

### IP 메모

```bash
ipconfig getifaddr en0   # 0단계에서 확인한 이더넷 장치 이름
```

`mini.local` 이 안 될 때를 대비한 대안입니다. 공유기 관리 페이지에서 **고정 IP를 예약**(DHCP 예약)해두면 재부팅해도 주소가 안 바뀝니다.

---

## 3. 암호 없애기 (SSH 키)

여기서부터 맥미니 키보드를 손에서 놓습니다.

### 첫 접속

```bash
# 맥북에서
ssh dasepa@mini.local
```

처음이면 fingerprint를 확인합니다 — `yes` 입력. 지문이 `~/.ssh/known_hosts` 에 저장되고 다음부터는 안 묻습니다.
(나중에 이 질문이 또 뜨면 서버가 바뀌었다는 신호이니 의심해봐야 합니다.)

프롬프트가 `dasepa@mini ~ %` 로 바뀌면 성공. `exit` 로 나옵니다.

### 키 만들기

<details>
<summary>원리</summary>

지금은 접속할 때마다 **암호를 네트워크로 보내서** 확인받습니다. 비밀이 매번 오가고, 자동화된 무차별 대입에 노출됩니다.

키 방식은 열쇠 한 쌍을 씁니다.

- **개인키** (`id_ed25519`) — 내 맥북에만 있고 **절대 밖으로 나가지 않음**
- **공개키** (`id_ed25519.pub`) — 맥미니에 맡겨둠. 남에게 보여도 무방

접속하면 맥미니가 문제를 내고, 맥북이 개인키로 서명해 답합니다. 맥미니는 보관 중인 공개키로 검증합니다. **개인키 자체는 한 번도 전송되지 않습니다.**

공개키가 자물쇠, 개인키가 열쇠라고 생각하면 쉽습니다. 자물쇠는 아무나 봐도 되지만 열쇠는 나만 갖고 있죠.

</details>

```bash
# 맥북에서
ls ~/.ssh/id_ed25519.pub                 # 이미 있는지 확인

ssh-keygen -t ed25519 -C "macbook"       # 없으면 생성 — 질문 3개 전부 엔터
ssh-copy-id dasepa@mini.local            # 맥미니에 공개키 맡기기 (여기서 마지막으로 암호)
```

- `-t` = **t**ype, 알고리즘. `ed25519`는 RSA보다 짧고 빠르면서 안전해서 현재 기본 선택
- `-C` = **C**omment, 이름표. 나중에 맥미니에 여러 기기 키가 쌓였을 때 구분용
- passphrase는 비워도 됩니다. 나중에 `ssh-keygen -p` 로 추가 가능

`ssh-copy-id` 의 진짜 가치는 **파일 권한을 알아서 맞춰준다**는 점입니다. SSH는 `.ssh` 폴더와 `authorized_keys` 권한이 정확하지 않으면 키를 조용히 무시하는데, 손으로 복사하면 여기서 대부분 막힙니다.

### 별명 만들기

```bash
# 맥북에서
printf 'Host mini\n  HostName mini.local\n  User dasepa\n  ServerAliveInterval 60\n  ServerAliveCountMax 3\n' >> ~/.ssh/config
```

`ServerAliveInterval 60` 은 60초마다 신호를 보내 **가만히 두면 연결이 끊기는 문제**를 막아줍니다. 서버 작업에서 체감이 큰 옵션입니다.

> 원래 이 자리에 heredoc(`cat >> file <<'EOF'`)을 쓰는 안내가 많은데, `EOF`를 직접 타이핑해야 끝난다는 걸 모르면 `heredoc>` 프롬프트에서 갇힙니다. `printf` 한 줄이 붙여넣기 한 번으로 끝나고 중복될 위험도 없습니다.
> 잘못 들어갔다면 `rm ~/.ssh/config` 로 지우고 위 명령을 **한 번만** 다시 실행하세요.

**확인**

```bash
ssh mini    # 암호를 묻지 않고 바로 들어가면 성공
```

> ⚠️ 개인키(`id_ed25519`)는 절대 남에게 주거나 어딘가에 올리면 안 됩니다. `.pub` 만 공유하는 겁니다.

---

## 4. 개발 환경 얹기

이제부터 `ssh mini` 로 들어간 상태에서 작업합니다.

### 모니터를 떼는 시점

전원과 랜선만 꽂아두면 됩니다. 다만 뽑기 전에 두 가지를 확인하세요.

1. `ssh mini` 가 암호 없이 되는지
2. **화면 공유가 실제로 접속되는지** — 맥북 Finder 사이드바에서 맥미니 클릭, 또는 `Cmd+K` → `vnc://mini.local`

모니터를 뗀 뒤 화면 공유 해상도가 이상하면 맥이 가상 화면을 만든 것입니다. **HDMI 더미 플러그**(헤드리스 어댑터, 5천 원선)를 꽂으면 정상 해상도로 돌아옵니다. SSH만 쓸 거라면 없어도 무방합니다.

**케이블은 맥미니 옆에 그대로 두세요.** 복구 모드 진입(전원 버튼 길게 누르기)과 대형 macOS 업데이트 후의 확인 화면은 네트워크로 처리할 수 없습니다. 키보드는 블루투스 말고 **유선 USB** 가 안전합니다.

### Homebrew

```bash
# ssh mini 안에서
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Xcode 명령줄 도구를 함께 받느라 **10~20분** 걸릴 수 있습니다. 멈춘 것처럼 보여도 기다리세요.

설치가 끝나면 화면 마지막의 **`==> Next steps:`** 에 실행할 명령 두 줄이 나옵니다. **그걸 그대로 복사해 실행하세요.** 보통 이 형태입니다.

```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

작은따옴표 `'…'` 가 중요합니다. 이래야 `$(...)` 가 지금 실행되지 않고 **글자 그대로** 파일에 저장돼서, 앞으로 터미널을 열 때마다 실행됩니다. 이 부분이 깨지면 새 터미널을 열 때마다 `(eval):1: no such file or directory: export HOMEBREW_PREFIX=...` 에러가 뜹니다.

> Homebrew는 아직 없는 상태에서 `homebrew` 라고 쳐도 당연히 안 됩니다. 설치 후에도 명령어 이름은 **`brew`** 입니다.

### 도구 설치

```bash
brew install git node uv tmux ripgrep fd jq
```

| 패키지 | 용도 |
|---|---|
| `node` | Claude Code가 얹히는 런타임 |
| `git` | 코드·설정을 맥북과 주고받는 통로 |
| `tmux` | [7단계](#7-ssh를-끊어도-계속-돌게-하기) 주인공 |
| `uv` | 파이썬 패키지·가상환경 관리 |
| `ripgrep` `fd` `jq` | 에이전트가 파일 검색·JSON 처리에 실제로 쓰는 도구 |

**확인**: `brew --version`, `node --version`

---

## 5. 집 밖에서도 접속하기

여기까지는 같은 공유기 안에서만 됩니다. `192.168.x.x` 는 사설 IP라 인터넷에서 닿을 수 없습니다.

> ### ⛔ 포트 포워딩은 하지 마세요
> 검색하면 "공유기에서 22번 포트를 열어라"는 안내가 많이 나옵니다. **하지 마세요.** SSH 포트를 인터넷에 노출하는 순간, 몇 시간 안에 전 세계에서 자동화된 침입 시도가 들어옵니다. 서버 운영 경험이 쌓이기 전에 건드릴 영역이 아닙니다.

Tailscale은 내 기기들끼리만 통하는 사설 네트워크를 만들어 줍니다. 공유기 설정을 전혀 건드리지 않고, 외부에 아무 포트도 열지 않으면서, 어디서든 접속됩니다. 개인 사용은 무료입니다.

### 맥미니 — 명령줄 버전

GUI 앱 대신 Homebrew 버전을 쓰면 **시스템 데몬으로 돌아서 로그인하지 않은 상태에서도 동작**합니다. 서버에는 이쪽이 맞고, 모니터 없이 SSH만으로 끝납니다. ([8단계](#8-재부팅과-정전에서-스스로-돌아오기)의 무인 복구가 이것 덕분에 됩니다.)

```bash
# ssh mini 안에서
brew install tailscale
sudo brew services start tailscale
sudo tailscale up
```

마지막 명령이 `To authenticate, visit:` 와 함께 URL을 출력합니다. **그 URL을 맥북 브라우저에 붙여넣어** 로그인하면(구글 계정 가능) SSH 창의 명령이 저절로 완료되고 `Success.` 가 뜹니다.

설치 중 `Warning: Taking root:admin ownership…` 은 데몬을 root 권한으로 돌리기 위한 정상 과정입니다.

### 맥북 — GUI 앱

<https://tailscale.com/download/mac> 에서 받아 설치하고 **맥미니와 같은 계정**으로 로그인하세요. 맥북은 메뉴 막대에서 상태를 볼 수 있는 GUI 쪽이 편합니다.

### 연결 확인

```bash
# 맥북에서
tailscale status
```

```
100.xx.xxx.xxx   macbook   you@   macOS   -
100.xxx.xxx.xx   mini      you@   macOS   -    ← 이 주소를 씁니다
```

이 `100.x.x.x` 주소는 **기기에 고정되어 바뀌지 않습니다.**

### 별명을 Tailscale 주소로

집 안팎 구분 없이 `ssh mini` 하나로 통일됩니다.

```bash
# 맥북에서 — 100.x.x.x 자리에 위에서 확인한 주소를 넣으세요
rm ~/.ssh/config
printf 'Host mini\n  HostName 100.x.x.x\n  User dasepa\n  ServerAliveInterval 60\n  ServerAliveCountMax 3\n' >> ~/.ssh/config
```

주소가 바뀌었으니 다음 접속 때 fingerprint를 한 번 더 물어볼 수 있습니다 — `yes`.

집 안에서 느려질 걱정은 안 하셔도 됩니다. 같은 네트워크에 있으면 Tailscale이 알아서 **직접 연결**합니다.

### 테스트

맥북 **Wi-Fi를 끄고 휴대폰 핫스팟**에 연결한 뒤 `ssh mini`. 집 밖 환경을 흉내 낸 검증입니다.

### 아이폰 (선택)

App Store에서 **Tailscale** 앱 + SSH 앱(**Termius** 무료)을 깔고 같은 계정으로 로그인하면 밖에서 폰으로 서버 상태를 확인할 수 있습니다. 급할 때 확인용으로 생각하세요 — 작은 화면에서 터미널 작업은 고통스럽습니다.

---

## 6. 에이전트 올리기

```bash
# ssh mini 안에서
npm install -g @anthropic-ai/claude-code
mkdir -p ~/agent
cd ~/agent
claude
```

### SSH 환경에서의 로그인

맥미니에는 브라우저를 띄울 화면이 없으니:

1. 터미널에 **URL이 출력**됨
2. 그 URL을 **맥북 브라우저**에 붙여넣고 인증
3. 받은 **코드를 다시 SSH 창에 붙여넣기**

### 작업 폴더에서 실행하는 이유

Claude Code는 **실행한 폴더를 작업 범위로 삼습니다.** 홈 폴더에서 실행하면 Downloads, Documents, Library까지 전부 범위에 들어가 느려지고, 의도치 않은 파일을 건드릴 여지가 생깁니다.

`git init` 까지 해두면 에이전트가 파일을 고칠 때 이력이 남아 되돌릴 수 있습니다. 에이전트를 자율적으로 굴릴 계획이라면 안전벨트에 가깝습니다.

<details>
<summary>Claude Code 안에서 셸 명령 쓰기</summary>

Claude Code 프롬프트에 친 것은 Claude가 해석해서 실행합니다. 셸 명령을 직접 날리려면 앞에 **`!`** 를 붙이세요 (`!ls -la`). 완전히 나가려면 **Ctrl+C 두 번** 또는 `/exit`.

</details>

---

## 7. SSH를 끊어도 계속 돌게 하기

지금 상태에서 SSH 창을 닫으면 실행 중이던 에이전트도 같이 죽습니다.

<details>
<summary>왜 죽나 — SIGHUP</summary>

유닉스에는 오래된 규칙이 있습니다. **터미널 연결이 끊기면 그 터미널에서 실행된 프로그램에게 `SIGHUP` 신호가 날아가고 대부분 종료됩니다.**

`SIGHUP` = **SIG**nal **H**ang **UP**. 이름 그대로 **전화를 끊는다**는 뜻입니다. 전화선과 모뎀으로 서버에 접속하던 시절, 전화를 끊으면 커널이 "이 사용자 나갔다"고 알리던 신호예요. 지금은 전화선을 안 쓰지만 이름과 동작은 그대로 남았습니다. SSH가 끊기는 것도 커널 입장에서는 같은 "전화 끊기"입니다.

**tmux의 해법**은 서버와 창을 분리하는 것입니다.

```
맥북 터미널  ──SSH──▶  [tmux 서버]  ──▶  claude 실행 중
   (그냥 보는 창)          ↑
                     실제로 여기서 돎
```

tmux 서버는 맥미니 안에서 독립적으로 돌고, SSH로 붙은 터미널은 **들여다보는 창**일 뿐입니다. 창을 닫아도 서버는 멀쩡합니다. 전화가 끊긴 게 아니라 **모니터를 뽑은 것**에 가까워요.

</details>

### 방법 A — tmux (먼저 이걸로)

```bash
# ssh mini 안에서
tmux new -s agent      # agent라는 이름의 세션 만들고 들어가기
cd ~/agent
claude

# 빠져나오기: Ctrl+b 를 누르고 손을 뗀 다음 d
```

화면 맨 아래 **초록색 막대**가 보이면 tmux 안입니다.

```bash
tmux ls                      # 돌고 있는 세션 목록
tmux attach -t agent         # 다시 들어가기
tmux kill-session -t agent   # 세션 종료
```

<details>
<summary>Ctrl+b는 왜 필요한가</summary>

tmux 안에서는 Claude Code나 vim 같은 프로그램도 키보드를 씁니다. 그러니 **"이 키는 tmux한테 하는 말"** 이라는 표시가 필요합니다. 그게 `Ctrl+b` 이고 **prefix key(접두 키)** 라고 부릅니다. 동시에 누르는 게 아니라 **순서대로** 누릅니다.

`d`는 **d**etach, `c`는 **c**reate. 대부분 영어 단어 첫 글자입니다.

조상 격인 GNU Screen이 `Ctrl+a` 를 썼는데, 그건 터미널에서 "줄 맨 앞으로 이동"과 충돌이 잦아서 tmux는 바로 옆 글자인 `b`를 골랐습니다.

| 키 | 하는 일 |
|---|---|
| `Ctrl+b` → `d` | 떼어놓기 (가장 많이 씀) |
| `Ctrl+b` → `[` | **스크롤 모드.** `q`로 나감 |
| `Ctrl+b` → `c` | 새 창 만들기 |
| `Ctrl+b` → `n` / `p` | 다음 / 이전 창 |
| `Ctrl+b` → `?` | 전체 단축키 목록 |

**`Ctrl+b` → `[` 는 꼭 기억하세요.** tmux 안에서는 마우스 휠 스크롤이 기본적으로 안 먹혀서, 이전 출력을 보려면 이 모드로 들어가야 합니다.

</details>

> **`/exit` 와 `Ctrl+b → d` 는 다릅니다**
>
> | 하는 일 | 결과 |
> |---|---|
> | `Ctrl+b` → `d` (detach) | 계속 실행됨 ✅ |
> | `/exit` (종료) | 대화 끝남 ❌ |
>
> tmux가 지키는 건 **떼어놓은** 프로그램이지 **종료한** 프로그램이 아닙니다. 앞으로 맥미니에서 나올 때는 `/exit` 대신 `Ctrl+b → d` 를 쓰세요.
>
> 종료한 대화를 되살리려면 Claude Code 자체 기능이 있습니다: `claude --continue`, `claude --resume`

**검증**: tmux 안에서 뭔가 실행 → `Ctrl+b` `d` → `exit` (SSH 종료) → `ssh mini` → `tmux attach -t agent` → 그대로 살아 있으면 성공.

### 방법 B — launchd (재부팅에도 살아남기)

tmux 세션은 **재부팅하면 사라집니다.** 맥의 공식 서비스 관리자인 launchd는 부팅 시 자동 실행하고 **프로세스가 죽으면 되살립니다.** tmux로 며칠 굴려보고 안정화된 뒤에 넘어오세요.

```bash
# ssh mini 안에서 — 실행 스크립트
mkdir -p ~/agent
cat > ~/agent/run.sh <<'EOF'
#!/bin/zsh
export PATH="/opt/homebrew/bin:$PATH"
cd "$HOME/agent"

# ↓ 실제로 돌릴 명령으로 바꾸세요
exec node index.js
EOF
chmod +x ~/agent/run.sh
```

`~/Library/LaunchAgents/com.dasepa.agent.plist` 를 만듭니다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.dasepa.agent</string>

  <key>ProgramArguments</key>
  <array>
    <string>/Users/dasepa/agent/run.sh</string>
  </array>

  <key>RunAtLoad</key>   <true/>
  <key>KeepAlive</key>   <true/>

  <key>WorkingDirectory</key>
  <string>/Users/dasepa/agent</string>

  <key>StandardOutPath</key>
  <string>/Users/dasepa/agent/out.log</string>
  <key>StandardErrorPath</key>
  <string>/Users/dasepa/agent/err.log</string>
</dict>
</plist>
```

`RunAtLoad` 는 부팅 시 시작, `KeepAlive` 는 죽으면 재시작. **이 두 줄이 "꺼지지 않는 에이전트"의 실체입니다.**

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.dasepa.agent.plist
launchctl print gui/$(id -u)/com.dasepa.agent   # 상태 확인
tail -f ~/agent/out.log                         # 로그 실시간

launchctl bootout gui/$(id -u)/com.dasepa.agent # 내리기
```

> ⚠️ `.plist` 를 수정했다면 **반드시 `bootout` 후 다시 `bootstrap`** 해야 반영됩니다. 파일만 고치면 아무 일도 일어나지 않습니다.
>
> ⚠️ LaunchAgent는 **사용자가 로그인된 상태**에서만 뜹니다. 재부팅 후에도 자동으로 뜨게 하려면 [8단계](#8-재부팅과-정전에서-스스로-돌아오기)의 자동 로그인이 필요합니다. (SSH 접속 자체에는 필요 없습니다.)

---

## 8. 재부팅과 정전에서 스스로 돌아오기

### 문제의 구조

맥미니가 정전으로 꺼졌다 켜졌다고 합시다. **FileVault(디스크 암호화)가 켜져 있으면 부팅 화면에서 사람이 암호를 칠 때까지 시스템이 멈춰 있습니다.** 네트워크도 안 올라오고, SSH도, Tailscale도 안 됩니다. 집에 돌아와 모니터를 연결하기 전까지 서버는 죽어 있는 셈입니다.

반대로 FileVault가 꺼져 있으면 로그인 화면까지 알아서 부팅되고, **sshd와 (brew로 설치한) Tailscale은 시스템 데몬이라 로그인 없이도 자동으로 뜹니다.**

### 선택

```bash
fdesetup status   # 현재 상태 확인
```

| 선택 | 내용 | 맞는 경우 |
|---|---|---|
| **FileVault 끄기** | 전원만 들어오면 사람 없이 완전 자동 복구. 대신 기기를 물리적으로 가져간 사람이 디스크를 읽을 수 있음 | 집 안에 고정 설치하고 민감한 데이터를 두지 않는 경우 — 대부분의 홈서버 |
| FileVault 유지 | 디스크는 안전하지만 정전·재부팅마다 사람이 직접 암호 입력 | 업무 자료나 고객 데이터를 다루는 경우 |

### FileVault 끄기 — SSH로 됩니다

모니터를 연결할 필요 없습니다.

```bash
# ssh mini 안에서
sudo fdesetup disable
#   Password:                      → 계정 암호
#   Enter the user name:           → dasepa
#   Enter the password for user …: → 계정 암호

fdesetup status                    # 진행률 확인
```

디스크 전체를 되돌리는 작업이 백그라운드로 돕니다. `Decryption in progress: Percent completed = 37` 처럼 진행률이 나오고, 용량에 따라 수십 분이 걸릴 수 있습니다. 그동안 맥미니는 평소처럼 써도 됩니다. `FileVault is Off.` 가 나오면 완료입니다.

### 자동 로그인은 필요 없습니다

흔한 오해인데, **SSH 접속에 자동 로그인은 필요하지 않습니다.** FileVault만 꺼져 있으면 아무도 로그인하지 않은 로그인 화면 상태에서도 sshd와 Tailscale이 뜹니다. 재부팅·정전 후 원격 접속이 살아나는 데는 이걸로 충분합니다.

자동 로그인이 필요해지는 건 **[7단계 방법 B](#방법-b--launchd-재부팅에도-살아남기)의 launchd로 에이전트를 자동 시작할 때**입니다. LaunchAgent는 사용자 로그인 세션 위에서만 뜨거든요. tmux를 쓰는 동안은 해당 없습니다.

필요해지면 `시스템 설정 › 사용자 및 그룹 › 자동으로 로그인` (FileVault가 꺼져 있어야 메뉴가 보임). 켜더라도 **화면 잠금은 따로 유지**할 수 있어서, 부팅은 사람 없이 끝나되 화면 앞에 앉은 사람은 암호를 요구받는 절충이 가능합니다.

### 검증

```bash
sudo shutdown -r now
```

SSH가 끊깁니다. **2~3분 기다렸다가** 맥북에서 `ssh mini`. 사람이 아무것도 안 했는데 접속되면 성공입니다.

그다음 **실제로 전원 코드를 뽑았다가 다시 꽂아보세요.** 1단계의 `autorestart 1` 덕분에 전기가 들어오면 저절로 켜집니다. 이 테스트를 반드시 해보세요.

> 재부팅하면 **tmux 세션은 사라집니다.** `tmux new -s agent` 로 다시 만드세요.

---

## 9. 보안 마무리

항상 켜져 있는 기계는 항상 노려지는 기계이기도 합니다. 기본만 해둬도 대부분 막힙니다.

### 방화벽

```bash
# ssh mini 안에서
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate on
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate   # enabled 확인
```

SSH와 화면 공유는 이미 허용된 서비스라 그대로 통과합니다.

### 암호 로그인 차단

3단계에서 키를 등록했으니 암호 인증은 더 이상 필요 없습니다. 무차별 대입을 원천 차단합니다.

> ⚠️ **반드시 `ssh mini` 가 암호 없이 되는 것을 확인한 다음** 실행하세요. 키가 제대로 등록되지 않은 상태에서 이걸 하면 원격으로 들어갈 방법이 사라집니다.

```bash
# ssh mini 안에서
printf 'PasswordAuthentication no\nKbdInteractiveAuthentication no\n' | sudo tee /etc/ssh/sshd_config.d/100-local.conf
sudo launchctl kickstart -k system/com.openssh.sshd
```

**현재 SSH 창을 닫지 말고** 맥북에서 새 터미널을 열어 `ssh mini` 가 되는지 먼저 확인하세요. 안 되면 열어둔 창에서 되돌립니다.

```bash
sudo rm /etc/ssh/sshd_config.d/100-local.conf
sudo launchctl kickstart -k system/com.openssh.sshd
```

### 자동 업데이트

`시스템 설정 › 일반 › 소프트웨어 업데이트 › ⓘ`

- **보안 응답 및 시스템 파일 설치** → 켜기
- **macOS 업데이트 자동 설치** → **끄기** 권장. 대형 업데이트는 재부팅을 동반하고, 하필 자리에 없을 때 로그인 대기 상태로 멈춰 있을 수 있습니다

### 백업

외장 SSD를 물려 **Time Machine** 을 켜두세요. 서버가 되는 순간 이 기계에는 다른 데 없는 것들이 쌓입니다. SSH 개인키도 여기 포함됩니다.

---

## 10. 최종 점검

| 테스트 | 방법 | 관련 단계 |
|---|---|---|
| 헤드리스 | 모니터·키보드·마우스를 전부 뽑고 `ssh mini` | 2, 4 |
| 방치 | 30분간 방치 후 다시 `ssh mini` | 1 |
| 재부팅 | `sudo shutdown -r now` 후 3분 대기 | 8 |
| 정전 | 전원 코드를 뽑았다 꽂기 | 1, 8 |
| 외부망 | 맥북을 휴대폰 핫스팟에 연결하고 `ssh mini` | 5 |
| 지속성 | 에이전트가 SSH 종료 후에도 살아 있는지 | 7 |

전부 통과하면 진짜 서버입니다.

---

## 막혔을 때

**접속이 안 될 때** — 화면 공유(2단계에서 켜둔 것)로 먼저 들어가 보세요.

| 화면 공유 | 원인 |
|---|---|
| 보임 | SSH 설정 문제 |
| 안 보임 | 네트워크 또는 전원 문제 |

이 구분만으로 원인의 절반이 좁혀집니다.

**그래도 안 되면** — 모니터와 키보드를 직접 연결하는 게 언제나 최후의 수단입니다. 그래서 케이블을 버리면 안 됩니다.

**진단에 쓸 명령**

```bash
pmset -g custom      # 절전 설정
tailscale status     # 네트워크
fdesetup status      # FileVault
uptime               # 며칠째 켜져 있나
tmux ls              # 돌고 있는 세션
```

---

## 삽질 기록

실제로 세팅하면서 막혔던 지점들. 대부분 문서에는 안 나오는 것들입니다.

| 증상 | 원인 | 해결 |
|---|---|---|
| `heredoc>` 에서 갇힘 | `EOF` 는 **직접 타이핑해야 끝나는 표시어**인데 저절로 나오는 줄 알았음 | `Ctrl+C` 로 취소 후 `printf` 한 줄로 대체 |
| 키를 만들었는데 계속 암호를 물음 | `ssh-copy-id` 단계를 건너뜀. 공개키가 맥미니에 없었음 | `exit` 후 맥북에서 `ssh-copy-id mini` |
| `zsh: command not found: homebrew` | 설치 전이었고, 설치 후에도 명령어 이름은 `brew` | 설치 스크립트 실행 |
| 맥미니 안에서 `ssh mini` 가 안 됨 | 이미 들어가 있는 상태. 별명 설정은 맥북에만 있음 | 프롬프트(`@mini` / `@macbook`)로 위치 확인 |
| tmux 붙었는데 대화가 없음 | `/exit` 로 종료했었음. tmux는 **떼어놓은** 것만 지킴 | `Ctrl+b` → `d` 로 나오기 |
| `open -e` 했는데 아무 일도 안 일어남 | TextEdit이 터미널 뒤에 열림 | 편집기 대신 `rm` + `printf` 로 파일 재작성 |
| 새 터미널마다 `(eval):1: no such file...` | `.zprofile` 의 brew 초기화 줄이 깨짐 (따옴표 문제) | 설치 프로그램이 알려주는 두 줄을 그대로 실행 |
| 키보드 설정 지원이 반복해서 뜸 | 비-애플 키보드 배열 판별. 허브를 거치면 키 입력이 안 먹힘 | 본체 USB에 직접 연결 후 `Z` → `/` → ANSI 선택 |

**PC 키보드를 맥에 연결했다면**

| PC 키 | macOS |
|---|---|
| Windows (⊞) | **Command (⌘)** |
| Alt | Option (⌥) |
| Caps Lock | **한/영 전환** (길게 누르면 진짜 Caps Lock) |

우측 Alt(한/영 키)는 macOS가 인식하지 않습니다. `Ctrl+Space` 도 입력 소스 전환에 쓸 수 있습니다.

---

## 참고

- [Tailscale 문서](https://tailscale.com/kb)
- [Homebrew](https://brew.sh)
- [tmux 치트시트](https://tmuxcheatsheet.com)
- [Claude Code 문서](https://docs.claude.com/en/docs/claude-code)

자주 쓰는 명령은 [CHEATSHEET.md](CHEATSHEET.md) 참고.
