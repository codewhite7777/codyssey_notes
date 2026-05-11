# 메모리 사용률 측정

> **TLDR** · `free` 출력의 `used`는 직관과 다름 — buff/cache를 포함하지만 이 메모리는 회수 가능. **`available` 컬럼이 실제 "여유" 메모리 신호**. 프로세스 단위 메모리는 RSS(실제 점유) vs VSZ(매핑 전체) 구분이 중요. JVM 같은 큰 VSZ도 RSS는 훨씬 작을 수 있음.

## 개요

Linux 메모리는 단순한 "총 메모리 / 사용 메모리" 모델로는 정확하게 표현되지 않는다. 커널은 free memory를 buff/cache로 적극 활용하므로 "사용 중인 메모리"가 무엇을 의미하는지가 미묘하다. `free` 명령의 출력을 처음 보면 "왜 메모리가 거의 다 차 있지?" 하고 놀라기 쉽지만, 그게 정상이다.

메모리 측정은 두 관점이 있다 — 시스템 전체 (`free`, `/proc/meminfo`)와 프로세스 단위 (`ps`, `/proc/PID/status`). monitor.sh는 보통 전체 사용률을 기준으로 하지만, 어떤 메트릭이 진짜 "사용량"인지를 정확히 알아야 한다.

## 왜 알아야 하나

운영 관점에서 가장 흔한 오해가 "free 메모리가 작다 = 시스템이 위험하다"는 직관이다. Linux는 unused memory를 buff/cache로 적극 활용하는데, 이 메모리는 필요시 즉시 application에 반환된다. 따라서 진짜 위험 신호는 `free` 컬럼이 아니라 `available` 컬럼이 적은 경우다. 이 차이를 모르고 false positive 알람을 받는 운영자가 많다.

이번 과제는 MEM > 10% 경고를 요구하는데, 어떤 측정값을 기준으로 할지가 monitor.sh 설계 결정이다. `used` 비율을 그대로 쓰면 buff/cache 포함이라 항상 경고가 나올 가능성이 높고, 실제 부하 신호로 활용하려면 `available` 기준이 적절하다.

## free 출력의 구조

```
$ free -h
               total        used        free      shared  buff/cache   available
Mem:           7.7Gi       3.4Gi       2.3Gi       100Mi       2.0Gi       4.0Gi
Swap:          2.0Gi          0B       2.0Gi
```

각 컬럼의 정확한 의미는 다음과 같다.

| 컬럼 | 의미 |
|---|---|
| **total** | 시스템 총 메모리 |
| **used** | total - free - buff/cache - shared (좁은 의미의 사용량) |
| **free** | 어디에도 안 쓰이는 진짜 free (보통 작음 — 정상) |
| **shared** | tmpfs, shared memory |
| **buff/cache** | 페이지 캐시·버퍼 — 회수 가능 |
| **available** | ★ application이 swap 없이 즉시 사용 가능한 메모리 |

`available`이 가장 운영적으로 의미 있는 지표다. `used + available ≈ total`이 대략 맞지만 정확히는 다양한 변수가 영향을 준다.

## /proc/meminfo와 측정 원리

`free`의 데이터 소스는 `/proc/meminfo`다.

```
$ cat /proc/meminfo | head -10
MemTotal:        8001234 kB
MemFree:         2345678 kB
MemAvailable:    4123456 kB
Buffers:          234567 kB
Cached:          1834567 kB
SwapCached:            0 kB
Active:          3456789 kB
Inactive:        2345678 kB
SwapTotal:       2097148 kB
SwapFree:        2097148 kB
```

`free` 명령은 이 파일을 파싱해서 보기 좋게 재포맷할 뿐이다. 직접 파싱하면 더 정밀한 측정 가능.

```mermaid
flowchart LR
    A[free 명령] --> B[/proc/meminfo 파싱]
    B --> C["Total = MemTotal"]
    B --> D["Free = MemFree<br/>(보통 작음 — 정상)"]
    B --> E["Available = MemAvailable<br/>★ 실제 여유"]
    B --> F["Buff/Cache = Buffers + Cached<br/>(회수 가능)"]
    B --> G["Used = Total - Free - Buff/Cache - Shared"]

    style E fill:#ccffcc
    style D fill:#ffe6cc
```

핵심은 `MemAvailable`이다. 이 값이 application 입장에서 진짜 사용 가능한 메모리이며, kernel이 buff/cache 회수 가능성·watermark 등을 종합해 계산해준다.

## 프로세스 메모리: RSS vs VSZ

`ps`나 `top`에서 보이는 프로세스 메모리는 두 종류다.

`VSZ`(Virtual Size)는 프로세스가 매핑한 모든 가상 메모리의 크기다. 코드·데이터·힙·스택·매핑된 라이브러리·mmap된 파일 등 모두 포함. 실제로 물리 메모리에 로드되지 않은 영역도 포함되므로 종종 매우 크게 보인다.

`RSS`(Resident Set Size)는 실제로 물리 메모리에 올라와 있는 부분만이다. 디스크에 swap된 페이지나 아직 page fault로 로드 안 된 영역은 제외. 운영에서 의미 있는 "프로세스가 차지하는 메모리"는 RSS다.

```
$ ps -p 1234 -o pid,vsz,rss,pmem,cmd
  PID    VSZ   RSS %MEM CMD
 1234 567890 145632  1.8 python agent_app.py
```

VSZ가 567MB, RSS가 145MB라는 의미 — 가상 주소공간은 크지만 실제 사용은 145MB. `%MEM`은 RSS/MemTotal × 100%다.

더 정확한 측정은 `/proc/PID/status` 또는 `/proc/PID/smaps`에서 가능하다. `VmRSS`, `VmData`, `RssAnon`, `RssFile`, `RssShmem` 등의 구분이 있다.

## 한 번 보자

```bash
free -h                        # 사람 읽기 좋은 형식
free -m                        # MB 단위
free -k                        # KB 단위 (기본)
free -s 2 -c 5                 # 2초 간격으로 5번 측정

# 사용률 계산 (used 기준 — 부정확)
free | awk '/^Mem:/ {printf "%.1f\n", $3/$2 * 100}'

# 사용률 계산 (available 기준 — 정확)
free | awk '/^Mem:/ {printf "%.1f\n", (1 - $7/$2) * 100}'

# /proc/meminfo 직접 파싱
total=$(awk '/^MemTotal/{print $2}' /proc/meminfo)
available=$(awk '/^MemAvailable/{print $2}' /proc/meminfo)
echo "scale=1; (1 - $available/$total) * 100" | bc
```

특정 프로세스의 메모리:

```bash
ps -p 1234 -o rss=             # KB 단위 RSS만
ps -p 1234 -o pid,vsz,rss,pmem,cmd
cat /proc/1234/status | grep -E 'Vm|Rss'

# 메모리 많이 쓰는 프로세스 top 10
ps aux --sort=-rss | head -11
```

## 흔한 함정

> [!WARNING]
> **`used`는 진짜 사용이 아니다**: `free` 출력의 `used` 컬럼은 buff/cache를 일부 포함하므로 실제 application 사용량보다 크다. 진짜 여유는 `available` 컬럼을 봐야 함. `used/total * 100`을 사용률로 쓰면 80%+ 항상 나오는 게 정상.

메모리 측정의 함정은 대부분 buff/cache 해석에서 비롯된다. 페이지 캐시는 디스크 I/O 가속을 위해 커널이 활용하는 영역으로, application이 메모리를 요청하면 즉시 해제된다. 따라서 `free`가 작아도 `available`이 크면 시스템은 건강하다. "메모리가 다 찼다" 알람을 보내려면 항상 available 기준이어야 한다.

OOM 진단에서도 같은 함정이 있다. OOM killer가 발동하기 직전 메모리 상태를 보면 `free`는 매우 작지만 그건 정상이다. OOM은 swap까지 다 차고 available도 거의 0일 때 발생한다. `dmesg | grep -i "killed process"`로 사후 확인 가능.

VSZ를 메모리 사용량으로 착각하는 경우도 흔하다. JVM은 종종 GB 단위 VSZ를 가지지만 실제 RSS는 훨씬 작다 — heap을 미리 큰 가상 영역으로 reserve하는 패턴이다. 프로세스 메모리 모니터링은 항상 RSS를 기준으로.

cgroup 환경의 메모리 측정은 또 다르다. 컨테이너 안에서 `free`를 실행하면 호스트의 메모리 정보가 보이는 경우가 많다 (`/proc/meminfo`가 namespace를 인지 못함). 정확한 컨테이너 메모리는 `cat /sys/fs/cgroup/memory.current`(v2) 또는 `cat /sys/fs/cgroup/memory/memory.usage_in_bytes`(v1)를 봐야 한다.

## B1-1 매핑

monitor.sh의 메모리 측정 패턴은 두 가지 접근이 가능하다.

```bash
# 접근 1: free 기반 used 사용 (간단, 명세 따라가는 형태)
MEM_USED=$(free | awk '/^Mem:/ {printf "%.0f\n", $3/$2 * 100}')

# 접근 2: available 기반 사용률 (운영적으로 정확)
MEM_USED=$(free | awk '/^Mem:/ {printf "%.0f\n", (1 - $7/$2) * 100}')

# 임계값 비교
if [ "$MEM_USED" -gt 10 ]; then
    echo "[WARNING] MEM 사용률 ${MEM_USED}% > 10%"
fi
```

명세의 임계값 10%는 매우 낮아서 거의 항상 경고가 나올 가능성이 높다. 실 운영에서는 80-90%가 일반적이지만, 이번 과제는 "경고 출력 동작 확인"이 목적이라 낮게 설정한 듯하다.

agent-app의 특정 프로세스 메모리만 추적하려면 다음 패턴을 쓴다.

```bash
PID=$(pgrep -f agent_app.py)
PROC_MEM=$(ps -p $PID -o pmem= | tr -d ' ')
echo "agent_app.py 메모리: ${PROC_MEM}%"
```

monitor.log 출력 포맷에 맞추려면 소수점 1자리까지 보존:

```bash
MEM_RAW=$(free | awk '/^Mem:/ {printf "%.1f", $3/$2 * 100}')
printf "MEM:%s%%\n" "$MEM_RAW"
```

## 인접 토픽

<details>
<summary><b>응용 토픽 — OOM killer·cgroup memory·swap·NUMA (펼치기)</b></summary>

OOM (Out Of Memory) killer는 메모리 압박 시 시스템이 hang되지 않도록 프로세스를 강제 종료하는 메커니즘이다. `/proc/PID/oom_score`로 OOM 우선순위를 보고, `/proc/PID/oom_score_adj`로 가중치 조정. 중요한 프로세스는 `-1000`(절대 안 죽임), 희생양 후보는 `1000` 같은 값을 준다. B1-2 트러블슈팅에서 자세히.

cgroup memory는 컨테이너·systemd 서비스의 메모리 제한을 위한 메커니즘이다. `MemoryLimit=512M` 같은 설정이 cgroup으로 구현되며, 한계를 넘으면 OOM killer가 컨테이너 안에서 발동한다. cgroup v2는 더 정밀한 제어(MemoryHigh, MemoryMax 분리)를 제공한다.

swap은 메모리 부족 시 디스크로 페이지를 임시 옮기는 메커니즘이다. swap을 켜면 OOM이 늦게 발동하지만 swap-in 시점에 성능이 급격히 저하된다. 데이터베이스 서버에서는 swap을 끄거나 `vm.swappiness=1`로 매우 낮게 설정하는 게 표준이다. 컨테이너 시대에 들어 swap의 가치가 재평가되고 있다 (zram, swap on SSD 등).

NUMA(Non-Uniform Memory Access)는 멀티소켓 서버에서 각 CPU 소켓이 자기 가까운 메모리를 더 빠르게 접근하는 아키텍처다. `numactl`로 메모리 affinity 설정 가능. 대규모 시스템에서 성능 튜닝의 영역으로, 일반 개발에서는 깊이 들어갈 일 적지만 Redis·DB 운영자는 알아야 한다.

`/proc/PID/smaps`는 프로세스 메모리 매핑의 세밀한 정보를 제공한다. 어느 라이브러리가 얼마나 차지하는지, anonymous vs file-backed 등 깊은 디버깅에 활용. `pmap PID`가 wrapper.

</details>

## 참고

- `man free`, `man ps`, `man proc` — meminfo 섹션
- `/proc/meminfo` — raw 데이터 소스
- [Linux Memory Management documentation](https://www.kernel.org/doc/html/latest/admin-guide/mm/)
- [Why Linux Ate My RAM?](https://www.linuxatemyram.com/) — buff/cache 오해 해소

---
출처: B1-1 (Layer 3.2) · 학습일: 2026-05-11
