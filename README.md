# AQUABIT 회사 홈페이지

클라우드 컴퓨팅 수업 팀 프로젝트용 **가상 게임 회사 AQUABIT**의 회사 홈페이지(`www.aquabit.lab`)입니다.
게임 서비스 Pixel Aquarium(`game.aquabit.lab`)과는 별도의 앱이며, "게임 시작" 버튼으로만 연결됩니다.

> AQUABIT은 실제 회사가 아닙니다.

- Node.js + Express + EJS
- 외부 CDN·폰트·이미지 없이 동작 (학교 내부망용)
- 공지·채용·FAQ·팀 정보는 `data/*.json`만 고치면 바로 반영 (서버 재시작 불필요)

## 페이지

| 주소 | 내용 |
|---|---|
| `/` | Canvas 픽셀 아쿠아리움 배너, 최신 공지 3개, 서비스 특징 |
| `/about` | 비전, 개발 연혁, 팀 역할, 서비스 구성 (ESXi/vCenter, DNS, 메일, 웹) |
| `/game` | Pixel Aquarium 단계별 소개 |
| `/notices`, `/notices/:id` | 공지 목록(분류 필터, 페이지 나눔), 상세, 이전·다음 글 |
| `/careers` | 가상 채용 공고 |
| `/support` | FAQ, 1:1 문의 폼 |
| 그 외 | 404 페이지 |

## 폴더 구조

```
server.js            앱 시작 (HOST:PORT에서 실행)
app.js               Express 설정 (helmet, 라우터, 404/500)
config/index.js      .env 값 읽기
routes/              pages.js, notices.js, support.js
lib/                 data.js(JSON 읽기), validate.js, mailer.js, inquiryStore.js, rateLimit.js
data/                notices / careers / faq / team / history .json  (+ inquiries.json 자동 생성)
views/               EJS 화면 (partials/에 head, header, footer)
public/              css, js(nav, aquarium, faq), img(SVG 로고·아이콘)
```

## 환경 변수 (.env)

`.env.example`을 복사해서 `.env`를 만듭니다. `.env`는 git에 올라가지 않습니다.

| 이름 | 기본값 | 설명 |
|---|---|---|
| `HOST` | `127.0.0.1` | 앱이 열릴 주소 (Apache 프록시를 쓰면 그대로 두기) |
| `PORT` | `3001` | 앱 포트 |
| `GAME_URL` | `http://game.aquabit.lab` | "게임 시작" 버튼 주소 |
| `SMTP_HOST` | `mail.aquabit.lab` | 사내 메일 서버(hMailServer) |
| `SMTP_PORT` | `25` | SMTP 포트 |
| `SMTP_USER` / `SMTP_PASS` | (없음) | 메일 서버 로그인 계정. **비밀번호는 .env에만** |
| `MAIL_FROM` | `SMTP_USER` | 보내는 사람 주소 |
| `MAIL_TO` | `support@aquabit.lab` | 문의 메일을 받을 주소 |

메일은 25번 포트에서 암호화(STARTTLS) 없이 `AUTH LOGIN`으로 로그인해 보냅니다. (실습용 메일 서버 설정에 맞춤)

---

## 1) 내 PC에서 실행하기

Node.js 20 이상이 필요합니다. (`node -v`로 확인)

```bash
git clone <저장소 주소> aquabit-homepage
cd aquabit-homepage
npm install
cp .env.example .env        # Windows PowerShell: Copy-Item .env.example .env
# .env를 열어 SMTP_PASS 등을 채움 (메일 서버가 없어도 실행은 됨)
npm start
```

브라우저에서 <http://127.0.0.1:3001> 을 엽니다.

- `npm run dev`: 파일을 고치면 서버가 자동으로 다시 켜집니다.
- 메일 서버에 연결할 수 없으면 문의는 `data/inquiries.json`에 저장되고 화면에 "접수됨(메일 전달 실패)"이 표시됩니다.

---

## 2) Rocky Linux 10 web VM에서 실행하기

```bash
# 패키지 설치
sudo dnf install -y git nodejs npm
node -v        # v20 이상인지 확인

# 앱 전용 사용자와 폴더
sudo useradd --system --home-dir /opt/aquabit-homepage --shell /sbin/nologin aquabit
sudo git clone <저장소 주소> /opt/aquabit-homepage
cd /opt/aquabit-homepage
sudo npm ci --omit=dev

# 환경 변수
sudo cp .env.example .env
sudo vi .env                      # SMTP_USER, SMTP_PASS 등 입력
sudo chmod 600 .env

# 권한: 앱 사용자가 data 폴더(문의 저장)에 쓸 수 있어야 함
sudo chown -R aquabit:aquabit /opt/aquabit-homepage

# 시험 실행 (Ctrl+C로 종료)
sudo -u aquabit node server.js
curl -I http://127.0.0.1:3001     # 다른 터미널에서 200 OK 확인
```

나중에 코드를 업데이트할 때:

```bash
cd /opt/aquabit-homepage
sudo -u aquabit git pull
sudo -u aquabit npm ci --omit=dev
sudo systemctl restart aquabit-homepage
```

> `git pull`을 `aquabit` 사용자로 하려면 저장소가 공개(public)이거나 접근 권한이 있어야 합니다.
> 안 되면 `sudo git pull` 후 `sudo chown -R aquabit:aquabit /opt/aquabit-homepage`를 다시 실행하세요.

---

## 3) systemd 서비스로 등록 (재부팅 후 자동 실행)

`/etc/systemd/system/aquabit-homepage.service` 파일을 만듭니다.

```ini
[Unit]
Description=AQUABIT company homepage (Node.js)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=aquabit
Group=aquabit
WorkingDirectory=/opt/aquabit-homepage
ExecStart=/usr/bin/node server.js
Environment=NODE_ENV=production
Restart=on-failure
RestartSec=3

# 보안 설정: data 폴더만 쓰기 허용
NoNewPrivileges=true
ProtectSystem=full
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/opt/aquabit-homepage/data

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now aquabit-homepage
sudo systemctl status aquabit-homepage
journalctl -u aquabit-homepage -f        # 로그 보기 (메일 전달 실패 기록도 여기에 남음)
```

`which node` 결과가 `/usr/bin/node`가 아니면 `ExecStart` 경로를 맞게 바꿔 주세요.

---

## 4) Apache VirtualHost로 www.aquabit.lab 프록시

같은 web VM의 Apache가 `www.aquabit.lab` 요청을 `127.0.0.1:3001`로 넘깁니다.
(`game.aquabit.lab`용 VirtualHost는 게임 서비스 쪽 설정을 그대로 둡니다.)

```bash
sudo dnf install -y httpd          # 이미 설치되어 있으면 생략
```

`/etc/httpd/conf.d/www.aquabit.lab.conf`:

```apache
<VirtualHost *:80>
    ServerName www.aquabit.lab
    ServerAlias aquabit.lab

    ProxyRequests Off
    ProxyPreserveHost On
    ProxyPass        / http://127.0.0.1:3001/
    ProxyPassReverse / http://127.0.0.1:3001/
    RequestHeader set X-Forwarded-Proto "http"

    ErrorLog  /var/log/httpd/www.aquabit.lab-error.log
    CustomLog /var/log/httpd/www.aquabit.lab-access.log combined
</VirtualHost>
```

```bash
# SELinux: Apache(httpd)가 다른 포트(3001)로 접속하도록 허용 (-P: 재부팅 후에도 유지)
sudo setsebool -P httpd_can_network_connect 1
getsebool httpd_can_network_connect          # → on

# 설정 검사 후 적용
sudo apachectl configtest                    # Syntax OK
sudo systemctl enable --now httpd
sudo systemctl reload httpd

# 방화벽에서 HTTP 열기
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

확인:

```bash
curl -I -H "Host: www.aquabit.lab" http://127.0.0.1/
```

- DNS(dns1·dns2)에 `www` 레코드가 web VM IP를 가리켜야 다른 PC에서 접속됩니다.
- SELinux 설정을 빠뜨리면 브라우저에 **503 Service Unavailable**이 뜨고, `/var/log/audit/audit.log`에 `httpd` 거부 기록이 남습니다.
- 앱은 `127.0.0.1`에서 온 프록시 요청에 한해 `X-Forwarded-For`를 믿으므로, 문의 횟수 제한(10분에 5건)이 실제 접속자 IP 기준으로 동작합니다.

---

## 문의 폼 동작

1. 서버에서 입력 검사: 이름, 이메일 형식, 분류, 제목, 내용 10자 이상, 개인정보 동의
2. 통과하면 `data/inquiries.json`에 저장 → nodemailer로 `MAIL_TO`에 메일 발송
3. 메일 발송이 실패해도 저장은 유지하고 "접수됨(메일 전달 실패)"으로 안내 (`mailStatus: "failed"` 기록)
4. 중복 접수 방지: 접수 후 결과 화면으로 이동(POST → 303 → GET)하고, 폼마다 한 번만 쓰는 토큰을 사용
5. 스팸 방지: 보이지 않는 honeypot 칸, 같은 IP는 10분에 5건까지

접수된 문의 보기: `cat data/inquiries.json`

## 보안

- helmet 보안 헤더 적용. HTTP 사이트라서 `upgrade-insecure-requests`와 HSTS는 끔
- 인라인 스크립트 없이 CSP를 엄격하게 적용 (`script-src 'self'`)
- 화면 출력은 EJS `<%= %>` 자동 이스케이프
- 비밀번호는 `.env`에만 두고, `.env`와 `data/inquiries.json`은 `.gitignore`로 제외

## 내용 수정

| 바꿀 것 | 파일 |
|---|---|
| 공지 | `data/notices.json` (`category`: `notice` / `update` / `maintenance` / `event`) |
| 채용 공고 | `data/careers.json` |
| FAQ | `data/faq.json` |
| 팀 역할 | `data/team.json` (`icon`: server, globe, code, mail / `color`: coral, yellow, mint, violet) |
| 연혁 | `data/history.json` |

JSON 문법이 틀리면 그 목록이 비어 보이고 서버 로그에 `읽기 실패`가 남습니다.

## GitHub에 올리기

```bash
git init
git add .
git status            # .env, node_modules, data/inquiries.json이 목록에 없는지 확인
git commit -m "AQUABIT 회사 홈페이지"
git branch -M main
git remote add origin https://github.com/<계정>/aquabit-homepage.git
git push -u origin main
```
