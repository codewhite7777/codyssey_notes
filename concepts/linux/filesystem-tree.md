# Linux 파일시스템 계층 (FHS)

## 개요
모든 파일·디렉토리가 단일 루트(`/`)에서 시작하는 트리로 조직되며, 그 배치는 **Filesystem Hierarchy Standard (FHS)**로 표준화되어 있다. Windows의 드라이브 분리 모델(`C:`, `D:`)과 달리, 외부 저장장치도 트리의 어떤 노드에 mount되어 단일 네임스페이스에 통합된다.

## 왜 알아야 하나
- 시스템 자동화·운영의 모든 경로 가정이 FHS에 기반
- 패키지 매니저·컨테이너·로깅 시스템이 표준 위치를 전제
- 보안 사고 다수가 잘못된 디렉토리 권한·위치에서 발생
- 모니터링·디버깅 시 "어디 봐야 하나"의 첫 단계

## 표준 디렉토리

```
/                    루트 (단일 네임스페이스의 진입점)
├── etc/             정적 시스템 설정 (텍스트 파일 위주)
├── home/            일반 사용자 홈 (멀티유저 일관 위치)
├── var/             가변 데이터 (logs, mail, spool, lib, cache)
│   ├── log/         시스템 + 애플리케이션 로그
│   └── lib/         애플리케이션 영구 상태 (DB, package state)
├── usr/             사용자 영역 프로그램 (대부분의 설치된 바이너리)
│   ├── bin/         일반 사용자 실행 파일
│   ├── sbin/        시스템 관리자용
│   └── local/       사이트 로컬 설치 (패키지 매니저 외)
├── bin, sbin/       기본 명령 (modern Linux는 /usr/bin·/usr/sbin 심볼릭 링크)
├── proc/            ★ pseudo-filesystem — 커널·프로세스 상태를 파일 인터페이스로 노출
├── sys/             ★ sysfs — 디바이스·드라이버·커널 객체 인터페이스
├── dev/             ★ devfs — 장치 파일 (블록·캐릭터 디바이스, udev 동적 관리)
├── tmp/             임시 (보통 tmpfs, 재부팅 시 휘발)
├── opt/             3rd-party 수동 설치 패키지
├── mnt, media/      mount 대상 (외부 저장 장치)
└── root/            root 사용자 홈 (/home과 분리 — 단일 사용자 부트 보장)
```

## /proc, /sys, /dev — pseudo-filesystem 트리오

전통적 디스크 파일이 아니라 **커널이 동적으로 생성하는 가상 파일**:

```
/proc       프로세스·커널 런타임 상태 (POSIX 스타일)
            예: /proc/PID/status, /proc/cpuinfo, /proc/meminfo, /proc/stat

/sys        커널 객체 모델 (sysfs) — 디바이스·드라이버·전원·cgroup
            예: /sys/class/net/eth0/, /sys/fs/cgroup/

/dev        디바이스 노드 (현대는 udev가 동적 관리)
            예: /dev/sda, /dev/null, /dev/random
```

**핵심 메커니즘**: `cat /proc/cpuinfo` 같은 read는 실제로는 **VFS가 커널 핸들러를 호출**해 그때그때 텍스트를 합성·반환. 디스크 I/O 없음. monitor.sh의 정보 수집 핵심 인터페이스.

## 한 번 보자

```bash
ls -F /                       # 디렉토리(/), 실행파일(*), 링크(@) 표시
stat /etc                     # inode 정보, 권한, 접근/수정 시간
mount | head                  # 현재 마운트된 파일시스템들
df -hT                        # 파일시스템 종류 포함 사용량 (-T 핵심)
findmnt                       # mount 트리 시각화
cat /proc/self/status         # 현재 프로세스(=cat) 상태
ls -l /proc/$$/fd/            # 현재 셸의 열린 파일 디스크립터들
cat /proc/mounts              # mount 정보 raw (=findmnt 데이터 소스)
```

## 흔한 함정

- **`/proc` write 가능한 항목** — 단순 read-only가 아니라 커널 파라미터 조정 인터페이스 (`echo 1 > /proc/sys/net/ipv4/ip_forward`). sysctl이 이걸 추상화.
- **`/etc/init.d` vs systemd unit** — `init.d`는 레거시 backwards compat. 현대는 `/etc/systemd/system/`.
- **`/tmp` ≠ `/var/tmp`** — `/tmp`는 보통 tmpfs로 휘발, `/var/tmp`는 디스크에 영구 (재부팅 후에도).
- **`/usr/local/bin` 우선순위** — `$PATH`에서 `/usr/bin`보다 먼저 와야 사이트 설치가 시스템 설치를 override.
- **`/root`가 별도인 이유** — `/home`이 별도 파티션이고 마운트 실패해도 root는 부팅 가능해야 함.
- **`/dev/null`도 디스크에 없음** — character device. write는 항상 성공, read는 즉시 EOF.

## B1-1 매핑

| 요구 | 경로 | 설계 의도 |
|---|---|---|
| `sshd_config` 수정 | `/etc/ssh/sshd_config` | 시스템 데몬 설정은 `/etc` |
| `AGENT_HOME` | `/home/agent-admin/agent-app` | 사용자 소유 앱은 홈 아래 |
| `monitor.log` | `/var/log/agent-app/` | 가변 로그는 `/var/log` |
| 리소스 측정 | `/proc/stat`, `/proc/meminfo` | 커널 상태 노출은 procfs |
| 임시 작업 | `/tmp` (tmpfs 권장) | 휘발성 OK |

## 인접 토픽 (확장 학습)

- **mount namespace** — 컨테이너가 격리된 `/`를 갖는 메커니즘
- **bind mount** (`mount --bind`) — 같은 디렉토리를 트리의 다른 위치에 노출
- **overlayfs** — 컨테이너 이미지의 레이어드 파일시스템 (lower + upper + work)
- **VFS layer** — inode·dentry·superblock 추상화로 ext4·XFS·btrfs·tmpfs 등 통합
- **inode 고갈** — 파일 수가 많으면 디스크 용량 남아도 새 파일 생성 실패 (`df -i`)

## 참고
- `man 7 hier` — 표준 매뉴얼
- `man 7 file-hierarchy` (systemd 진영)
- [FHS 3.0](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html)

---
출처: B1-1 (Layer 1.1) · 학습일: 2026-05-07
