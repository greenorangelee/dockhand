# Dockhand 설치 기록

Proxmox LXC 위에 Docker를 사용해서 Dockhand를 구축한 과정을 기록한 레포입니다.
Dockhand는 여러 Docker 호스트를 웹 UI에서 통합 관리할 수 있는 컨테이너 관리 도구입니다.

---

## 환경

| 항목 | 내용 |
|------|------|
| Proxmox 버전 | 9.x |
| CPU | AMD Ryzen 7 5700X |
| Dockhand LXC IP | 10.100.100.111 |
| 네트워크 | UniFi UCG Fiber + 관리형 스위치 + VLAN 구성 |

---

## 레포 구조

```
dockhand-setup/
├── dockhand/
│   ├── docker-compose.yml   # Dockhand + PostgreSQL
│   └── .env                 # 환경변수
└── README.md
```

---

## Dockhand 버전 선택

Dockhand는 두 가지 버전이 있습니다.

| | SQLite 버전 | PostgreSQL 버전 |
|---|---|---|
| DB | 내장 SQLite | 외부 PostgreSQL 컨테이너 |
| 컨테이너 수 | 1개 | 2개 |
| 적합한 경우 | 소규모 홈랩 | 수십 개 이상 컨테이너 운영 |
| 마이그레이션 | PostgreSQL로 전환 시 재설치 필요 | - |

수십 개 컨테이너 운영 예정이므로 **PostgreSQL 버전**을 선택했습니다.
나중에 SQLite → PostgreSQL 전환 시 재설치가 필요하므로 처음부터 PostgreSQL로 시작하는 것을 권장합니다.

---

## 1단계 — Proxmox LXC 생성

### Dockhand 전용 LXC 생성

Proxmox 웹 UI에서 새 LXC 컨테이너 생성:

| 항목 | 값 |
|------|------|
| 템플릿 | ubuntu-24.04-standard |
| vCPU | 2코어 |
| RAM | 2048 MB |
| 디스크 | 20 GB |
| 네트워크 | 고정 IP (10.100.100.111/24) |

### Nesting 활성화 (필수)

Docker를 LXC 안에서 실행하려면 반드시 필요합니다.

**방법 1 — 웹 UI:**
LXC 선택 → `Options` → `Features` → `Nesting` 체크

**방법 2 — Proxmox 호스트 쉘:**
```bash
pct set <CTID> --features nesting=1
```

> **참고:** Ubuntu 24.04 LXC 생성 시 아래 경고가 뜰 수 있습니다.
> `WARN: Systemd 255 detected. You may need to enable nesting.`
> Nesting을 활성화하면 해결됩니다.

---

## 2단계 — Docker 설치

Dockhand LXC 콘솔에서 실행:

```bash
apt update && apt upgrade -y
curl -fsSL https://get.docker.com | sh
docker --version  # 설치 확인
```

---

## 3단계 — Dockhand 배포

### 디렉토리 생성

```bash
mkdir -p /home/docker/dockhand/postgres
mkdir -p /home/docker/dockhand/data
cd /home/docker/dockhand
```

### 파일 작성

`dockhand/docker-compose.yml` 내용 복사:

```bash
nano docker-compose.yml
```

`dockhand/.env` 내용 참고해서 작성:

```bash
nano .env
```

```env
POSTGRES_USER=dockhand
POSTGRES_PASSWORD=비밀번호_변경필수
POSTGRES_DB=dockhand
DOCKHAND_PORT=3000
```

### 컨테이너 실행

```bash
docker compose up -d
docker compose logs -f  # 정상 실행 확인
```

### 접속 확인

브라우저에서 `http://10.100.100.111:3000` 접속 후 초기 계정 생성

---

## 4단계 — 로컬 환경 연결

Dockhand 웹 UI에서 로컬 Docker를 관리하려면 환경을 추가해야 합니다.

1. 우측 상단 `+` 클릭 → `Add environment`
2. 아래처럼 입력:

| 항목 | 값 |
|------|------|
| Name | local (원하는 이름) |
| Connection type | Unix socket |
| Socket path | /var/run/docker.sock |
| Public IP | 10.100.100.111 |

3. `Test connection` 클릭 → 초록 체크 확인
4. `+ Add` 클릭

---

## 5단계 — 원격 환경 연결 (Hawser 에이전트)

다른 LXC/VM의 Docker를 Dockhand에서 관리하려면 해당 호스트에 Hawser 에이전트를 설치해야 합니다.

### Dockhand에서 토큰 발급

1. `Environments` → `+` 클릭
2. Connection type: `Hawser agent (edge)` 선택
3. 표시된 토큰 복사
4. Public IP: 원격 호스트 IP 입력
5. `+ Add` 클릭 (토큰 저장)

### 원격 호스트에 Hawser 설치

원격 LXC 콘솔에서:

```bash
mkdir -p /home/docker/hawser/stacks
cd /home/docker/hawser
```

`docker-compose.yml` 작성:

```yaml
services:
  hawser:
    image: ghcr.io/finsys/hawser:latest
    container_name: hawser
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /home/docker/hawser/stacks:/data/stacks
    environment:
      - DOCKHAND_SERVER_URL=${DOCKHAND_SERVER_URL}
      - TOKEN=${HAWSER_TOKEN}
```

`.env` 작성:

```env
DOCKHAND_SERVER_URL=ws://10.100.100.111:3000/api/hawser/connect
HAWSER_TOKEN=Dockhand에서_발급받은_토큰
```

```bash
docker compose up -d
docker compose logs -f  # Connected to Dockhand 메시지 확인
```

### Dockhand에서 연결 확인

환경 목록에서 해당 환경이 `Online` 상태로 바뀌면 성공입니다.

---

## 추후 계획

- [ ] Traefik 리버스 프록시 연동
- [ ] 각 LXC에 Hawser 에이전트 배포
- [ ] Proxmox 정기 백업 설정

---

## 참고

- [Dockhand 공식 GitHub](https://github.com/Finsys/dockhand)
- [Hawser 에이전트 공식 GitHub](https://github.com/Finsys/hawser)
