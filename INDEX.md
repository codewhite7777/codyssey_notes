# Codyssey 진행 현황

> codyssey_notes 레포 — 학습 자산 통합. 각 과제 산출물은 별도 레포 (`codyssey_b1_N`).

## 상태 표기
🔵 시작 전 | 🟡 진행 중 | 🟢 완료 | 🔴 막힘 | ⚪ 보류

## 기본 단계 (Basic)

| ID | 주제 | 분야 | 시간 | 상태 | 학습 노트 | 과제 레포 |
|---|---|---|---|---|---|---|
| B1-1 | 시스템 관제 자동화 스크립트 | Linux/OS | 40h | 🟢 | [`codyssey_b1_1_study/`](./codyssey_b1_1_study/) (21개) | [`codyssey_b1_1`](https://github.com/codewhite7777/codyssey_b1_1) — 코드 작성 완료, 테스트 대기 |
| B1-2 | 프로세스·리소스 트러블슈팅 | Linux/OS | 40h | 🔵 | (예정) | `codyssey_b1_2` |
| B1-3 | 파일 기반 가계부 | Python/Git | 60h | 🔵 | (예정) | `codyssey_b1_3` |
| B1-4 | Git 협업 워크플로우 | Python/Git | 20h | 🔵 | (예정) | `codyssey_b1_4` (팀) |
| B1-5 | (미정) |  |  | 🔵 |  |  |
| B1-6 | (미정) |  |  | 🔵 |  |  |
| B1-7 | (미정) |  |  | 🔵 |  |  |
| B1-8 | (미정) |  |  | 🔵 |  |  |
| B1-9 | (미정) |  |  | 🔵 |  |  |
| B1-10 | (미정) |  |  | 🔵 |  |  |
| B1-11 | (미정) |  |  | 🔵 |  |  |
| B1-12 | (미정) |  |  | 🔵 |  |  |
| B1-13 | (미정) |  |  | 🔵 |  |  |
| B1-14 | (미정) |  |  | 🔵 |  |  |
| B1-15 | (미정) |  |  | 🔵 |  |  |

> 나머지 11개 과제 정보 받으면 채워넣기.

## 핵심 개념 (B1-1 — 21개 노트 모두 학습자 친화 패턴 적용)

| Layer | 주제 | 노트 |
|---|---|---|
| 1. Linux Foundation | 파일·사용자·환경·프로세스 | [filesystem-tree](./codyssey_b1_1_study/filesystem-tree.md), [users-and-groups](./codyssey_b1_1_study/users-and-groups.md), [file-permissions](./codyssey_b1_1_study/file-permissions.md), [shell-environment](./codyssey_b1_1_study/shell-environment.md), [process-and-signals](./codyssey_b1_1_study/process-and-signals.md) |
| 2. 보안 & 네트워킹 | SSH·방화벽·포트·ACL | [ssh-deep-dive](./codyssey_b1_1_study/ssh-deep-dive.md), [sshd-config](./codyssey_b1_1_study/sshd-config.md), [ports-and-listening](./codyssey_b1_1_study/ports-and-listening.md), [firewall-ufw-vs-firewalld](./codyssey_b1_1_study/firewall-ufw-vs-firewalld.md), [posix-acl](./codyssey_b1_1_study/posix-acl.md) |
| 3. 자원 측정 | CPU·MEM·DISK 모니터링 | [cpu-measurement](./codyssey_b1_1_study/cpu-measurement.md), [memory-measurement](./codyssey_b1_1_study/memory-measurement.md), [disk-usage-df-vs-du](./codyssey_b1_1_study/disk-usage-df-vs-du.md) |
| 4. Bash 스크립팅 | 기초·안전·흐름·치환·trap | [bash-fundamentals](./codyssey_b1_1_study/bash-fundamentals.md), [bash-set-safe](./codyssey_b1_1_study/bash-set-safe.md), [bash-control-flow](./codyssey_b1_1_study/bash-control-flow.md), [bash-substitution](./codyssey_b1_1_study/bash-substitution.md), [bash-trap](./codyssey_b1_1_study/bash-trap.md) |
| 5. 자동화 & 로그 | cron·로그 회전 | [cron-fundamentals](./codyssey_b1_1_study/cron-fundamentals.md), [cron-environment-gotchas](./codyssey_b1_1_study/cron-environment-gotchas.md), [log-rotation](./codyssey_b1_1_study/log-rotation.md) |

**공통 패턴**: 과제 요구사항 → 구현 방법 → 개념 / 명세 원문 정확 인용 / 회사 비유 + 이모지 / 4노드 이하 Mermaid / 일반인 친화 언어.

## 학습 자산
- 학습 컨벤션: [`CLAUDE.md`](./CLAUDE.md)
- B1-1 학습: [`codyssey_b1_1_study/`](./codyssey_b1_1_study/) — 21개 노트
- **회고 노트**: [`retrospectives/`](./retrospectives/)
  - [2026-05-12 B1-1 트러블슈팅 5건](./retrospectives/2026-05-12-b1-1-troubleshooting.md) — mktemp 권한·logrotate 보안·SIGPIPE×pipefail·파일명 불일치·GLIBC 미스매치

## 누적 통계
- 완료: 0 / 15 (B1-1 코드 작성 완료, 평가 대기)
- 학습 시간 누적: ~40h / ~600h (예상)
- 정리한 노트 수: **21** (B1-1 Layer 1~5 전체)
