# Judge0 서버 구성

> 작성 2026-09-14. 서버 조사는 전부 읽기 명령(`uname`, `cat /proc/*`, `stat`, `docker --version`)만 사용. 설치·기동은 PM(서버 관리자, sudo) 몫

## 1. 목표 구성

- Judge0 CE **1.13.1** 자체 호스팅: `server`(API :2358) + `workers` + `db`(postgres 13) + `redis`(6) - 공식 compose 그대로, 설정만 우리 값
- 접근: **Ondal BE 만** - 인터넷·nginx 에 노출하지 않음. `AUTHN_TOKEN` 필수(X-Auth-Token), 제출 코드의 네트워크 차단(`ENABLE_NETWORK=false`, `ALLOW_ENABLE_NETWORK=false`)
- 파일 단일 원천 = GitHub: `ondal-BE/infra/judge0/`(compose + `judge0.conf.example` + 절차 README) - 서버 안내 가이드 규칙("서버에서 혼자 수정 금지")
- BE 연결: `.env` 에 `JUDGE0_URL`, `JUDGE0_TOKEN`, `ONDAL_JUDGE_ENGINE=judge0`(없으면 prod 기동 거부 - design.md 결정 6)

## 2. 해달 서버 현황 (2026-09-14)

| 항목 | 값 | 채점 엔진에 미치는 영향 |
|---|---|---|
| OS · 커널 | Ubuntu 26.04.1 LTS · `7.0.0-31-generic` | 최신 - cgroup v2 전용 |
| systemd | 259 | **cgroup v1 지원 제거됨(v258 부터)** |
| cgroup | `/sys/fs/cgroup` = cgroup2fs, 커널 `CONFIG_MEMCG_V1 is not set`, `/proc/cgroups` 에 `memory` 없음 | **legacy memory cgroup 자체가 없음** → GRUB 로 v1 전환해도 memory 컨트롤러가 생기지 않음 |
| 하드웨어 | 8 vCPU(bare metal, `/dev/kvm` 있음, vmx), RAM 7.2GB(가용 4.7GB), 디스크 76GB 여유 | VM 1대(2 vCPU · 2GB) 여유 있음. Judge0 이미지 약 20GB 디스크 |
| Docker | 29.8.0 · Compose v5.5.1 | OK |
| 기존 서비스 | 0-infra(nginx·cloudflared) · 1-auth(Keycloak) · 2-home · 3-ondal(ondal-be·ondal-db) · 4-rental | 재부팅·커널 파라미터 변경은 전부에 영향 |

## 3. 결정 대기 - Judge0 1.13.1 은 이 서버 커널에서 그대로 돌지 않는다

- 사실
  - Judge0 1.13.1(2024-04, 마지막 릴리스)의 샌드박스 isolate 1.8.1 은 **cgroup v1 memory 컨트롤러**(`/sys/fs/cgroup/memory/box-N`)를 요구. 공식 배포 문서도 Ubuntu 22.04 에 `systemd.unified_cgroup_hierarchy=0` GRUB 설정을 요구
  - 해달 서버는 (1) systemd 259 - v1 모드 없음 (2) 커널에 MEMCG_V1 미탑재 → **GRUB 우회 불가**
  - cgroup v2 대응은 upstream PR [judge0/judge0#599](https://github.com/judge0/judge0/pull/599)(2026-05, isolate 2.6 + `judge0/compilers#29`) - **미머지·미리뷰**, 작성자 운영 환경(AKS, Debian bookworm)에서 스모크 9/9 통과 보고. 관련 이슈 #543·#536·#603 열림
- 선택지

| | A. Judge0 1.13.1 을 **VM 안에서** | B. Judge0 cgroup v2 포크(PR #599) 직접 빌드 | C. 자체 Docker 러너 (`JudgeEngine` 구현 추가) |
|---|---|---|---|
| Judge0 결정 유지 | ✅ 릴리스 그대로 | ✅ (비공식 빌드) | ❌ 엔진 교체 |
| 서버 변경 | multipass(snap) + VM 1대. 호스트 커널·GRUB 무변경 | 없음(privileged 컨테이너) | 없음 |
| 위험 | VM 운영 1개 추가(자동 시작·메모리 2GB) | 미리뷰 코드·이미지 자체 빌드(수 GB)·업데이트 경로 불확실 | 샌드박스를 우리가 책임(보안 검증 부담), 개발 1~2일 |
| 격리 | VM 경계 + isolate - 가장 강함 | isolate 2.6 | Docker 격리만(`--network none`, 자원 제한) |
| 착수 소요 | PM 1~2시간(4절) | PM+Claude 반나절~, 실패 가능 | Claude 1~2일 |

- **권고 = A** - 이번 주 목표와 "Judge0 확정" 결정을 동시에 지키는 유일한 길. B 는 A 가 막힐 때(예: snap·KVM 사용 불가) 파일럿, C 는 최후 대안. BE·FE 는 어느 쪽이든 `JudgeEngine` 뒤에서 같다(A·B 는 같은 Judge0 API → `Judge0Engine` 그대로)
- 이 결정은 PM 몫 - 서버 관리자 판단(VM 운영 의사·자원)이 필요. 결정 전까지 BE·FE 는 `FakeJudgeEngine` 으로 개발·검증 진행

## 4. 권고안 A 절차 (PM 실행, 전부 sudo)

1. 호스트에 multipass: `sudo snap install multipass` → `multipass launch 22.04 --name judge0 --cpus 2 --memory 2G --disk 25G` → `multipass shell judge0`
2. VM 안 (Ubuntu 22.04 - cgroup v1 memory 컨트롤러 있음)
   - Docker: `curl -fsSL https://get.docker.com | sudo sh`, `sudo usermod -aG docker ubuntu`
   - cgroup v1: `/etc/default/grub` 의 `GRUB_CMDLINE_LINUX="systemd.unified_cgroup_hierarchy=0"` → `sudo update-grub` → `sudo reboot` → 확인 `stat -fc %T /sys/fs/cgroup/` = `tmpfs`, `ls /sys/fs/cgroup/memory` 존재
   - 파일 배치: `ondal-BE/infra/judge0/` 의 `docker-compose.yml`·`judge0.conf.example` 을 VM `~/judge0/` 로(`multipass transfer` 또는 git clone) → `cp judge0.conf.example judge0.conf` → `REDIS_PASSWORD`·`POSTGRES_PASSWORD`·`AUTHN_TOKEN` 채움(`openssl rand -hex 32`)
   - 기동: `docker compose up -d db redis` → 10초 → `docker compose up -d` (공식 절차 - db 초기화 뒤 서버)
3. 확인
   - VM IP: 호스트에서 `multipass info judge0 | grep IPv4` (10.x.x.x - multipass 내부 브리지, 외부 노출 없음)
   - 헬스: `curl -s -H "X-Auth-Token: <토큰>" http://<VM IP>:2358/system_info` → JSON
   - 스모크: 6절 명령 - C "hello" 가 status 3(Accepted) 로 끝나면 성공. 여기서 `Failed to create control group` 이 나오면 VM 의 cgroup v1 전환이 안 된 것
   - ondal-be 컨테이너에서 도달 확인: `sudo docker exec ondal-be wget -qO- --header="X-Auth-Token: <토큰>" http://<VM IP>:2358/languages | head -c 200` (Docker bridge → 호스트 → multipass 브리지 라우팅. 막히면 `sudo iptables -I DOCKER-USER -d <VM IP> -j ACCEPT`)
4. BE 연결: `/opt/haedal/3-haedal-ondal/ondal-BE/.env` 에 `JUDGE0_URL=http://<VM IP>:2358`, `JUDGE0_TOKEN=<토큰>`, `ONDAL_JUDGE_ENGINE=judge0` → `sudo docker compose up -d --build`
5. 자동 시작: multipass 는 호스트 부팅 시 VM 을 자동 시작(기본), compose 는 `restart: always`. 호스트 재부팅 후 3절 헬스 한 번

## 5. Judge0 설정값 (judge0.conf - 기본값과 다른 것만)

| 변수 | 값 | 이유 |
|---|---|---|
| `AUTHN_TOKEN` | 랜덤 64자 | BE 외 호출 차단 |
| `ENABLE_WAIT_RESULT` | `false` | BE 는 폴링만(동기 대기로 서버 스레드 점유 방지) |
| `ENABLE_CALLBACKS` | `false` | 콜백 미사용(design.md 결정 5) |
| `ENABLE_NETWORK` / `ALLOW_ENABLE_NETWORK` | `false` / `false` | 제출 코드 네트워크 완전 차단 |
| `ENABLE_COMPILER_OPTIONS` / `ENABLE_COMMAND_LINE_ARGUMENTS` / `ENABLE_ADDITIONAL_FILES` | `false` | 쓰지 않는 입력면 닫기(CVE-2024-28185 계열 완화) |
| `MAX_QUEUE_SIZE` | `500` | 분반 30명 × 케이스 10개 동시 제출 대비 |
| `COUNT`(workers) | `2` | VM 2 vCPU - 채점 시간 측정 안정성(코어 수 이상은 시간 튐) |
| `MAX_CPU_TIME_LIMIT` / `MAX_WALL_TIME_LIMIT` | `15` / `30` | 출제 폼 상한과 일치 |
| `MAX_MEMORY_LIMIT` | `524288` | 512MB - 출제 폼 상한 |
| `MAX_SUBMISSION_BATCH_SIZE` | `50` | 케이스 최대 50개를 배치 1회로 |
| `JUDGE0_TELEMETRY_ENABLE` | `false` | 외부 전송 없음 |
| `MAINTENANCE_MODE` | `false` | 점검 시 `true` 로 - BE 는 503 을 `PENDING` 유지·재시도로 흡수 |

## 6. 스모크 테스트 명령 (VM 또는 호스트에서)

```bash
TOKEN=<AUTHN_TOKEN>; J=http://<VM IP>:2358
# 1) 제출 (C, language_id 50) - base64 로 소스·입력 전달
SRC=$(printf '#include <stdio.h>\nint main(){int a,b;scanf("%%d %%d",&a,&b);printf("%%d\\n",a+b);return 0;}' | base64 -w0)
IN=$(printf '1 2\n' | base64 -w0)
TOKEN_ID=$(curl -s -X POST "$J/submissions?base64_encoded=true&wait=false" -H "X-Auth-Token: $TOKEN" -H 'Content-Type: application/json' \
  -d "{\"language_id\":50,\"source_code\":\"$SRC\",\"stdin\":\"$IN\",\"cpu_time_limit\":2,\"memory_limit\":262144}" | sed -E 's/.*"token":"([^"]+)".*/\1/')
sleep 3
# 2) 결과 - status.id 3 = Accepted, stdout(base64) "Mwo=" = "3\n"
curl -s "$J/submissions/$TOKEN_ID?base64_encoded=true&fields=status,stdout,time,memory,compile_output,message" -H "X-Auth-Token: $TOKEN"
```

## 7. 운영 메모

- 로그: VM 안 `docker compose logs -f workers` - `Failed to create control group` = cgroup, `Internal Error`(13) 빈발 = 디스크·메모리 확인
- 업데이트: Judge0 릴리스가 나오면 compose 의 이미지 태그만 - cgroup v2 정식 지원 릴리스가 나오면 4절 VM 을 걷고 호스트 compose 로 이전(파일은 같은 `infra/judge0/`)
- 백업 불필요: Judge0 DB 는 실행 캐시일 뿐, 판정·케이스는 Ondal DB(`test_cases`·`judge_results`)에 있음. 재설치 = 5분
- 용량: Judge0 이미지 약 20GB(컴파일러 전부 포함) - VM 디스크 25GB 로 시작, `docker system df` 로 확인
- 보안: 2358 은 VM 내부 브리지에만 - 호스트 방화벽에서 외부 → 10.x 차단 확인. 토큰은 `.env`·`judge0.conf` 두 곳, 레포에 올리지 않음(`judge0.conf` gitignore)
