# B1-1 학습 노트 — 시스템 관제 자동화 스크립트

> 과제 [codyssey_b1_1](https://github.com/codewhite7777/codyssey_b1_1) 의 학습 자산.
> 실제 산출물 코드는 별도 레포에, 여기는 **학습 노트만** 모임.

## Layer 1 — Linux Foundation

| 노트 | 핵심 주제 |
|---|---|
| [filesystem-tree.md](./filesystem-tree.md) | FHS, `/etc`·`/var`·`/proc`, pseudo-filesystem 트리오 |
| [users-and-groups.md](./users-and-groups.md) | UID/GID, primary vs supplementary group, NSS |
| [file-permissions.md](./file-permissions.md) | 12비트 mode, 권한 검사 알고리즘, setgid 디렉토리 |
| [shell-environment.md](./shell-environment.md) | 셸 시작 모드, startup 파일 로딩, cron 환경 함정 |
| [process-and-signals.md](./process-and-signals.md) | fork-exec, 상태 머신, SIGTERM vs SIGKILL, 좀비 reaping |

## Layer 2 — 보안 & 네트워킹

| 노트 | 핵심 주제 |
|---|---|
| [ssh-deep-dive.md](./ssh-deep-dive.md) | 3단계 핸드셰이크, 키 인증 흐름, 호스트 키 |
| [sshd-config.md](./sshd-config.md) | 옵션·우선순위·Match 블록·재로드 절차 |
| [ports-and-listening.md](./ports-and-listening.md) | TCP 상태 머신, 바인딩 주소, ss/lsof |
| [firewall-ufw-vs-firewalld.md](./firewall-ufw-vs-firewalld.md) | netfilter layer, ufw·firewalld 비교 |
| [posix-acl.md](./posix-acl.md) | 9비트 한계, ACL 항목·mask·default ACL |

## Layer 3-5 (예정)

- Layer 3 — 자원 관측 (top·ps·free·df·`/proc`)
- Layer 4 — Bash 스크립팅 (exit code, `set -euo pipefail`, trap)
- Layer 5 — 자동화 & 운영 (cron, logrotate)

## 노트 양식

각 노트는 다음 구조:
1. **TLDR** — 한 줄 핵심
2. **개요·왜 알아야 하나** — 동기·맥락
3. **핵심 개념** — 정의·동작 원리 (Mermaid 다이어그램)
4. **한 번 보자** — 실제 명령 + 출력 페어링
5. **흔한 함정** — 운영에서 부딪히는 미묘한 함정
6. **B1-1 매핑** — 과제 직접 적용
7. **인접 토픽** — `<details>` 접기로 확장 학습
8. **참고** — man page·표준 문서

## 출처
- B1-1 시스템 관제 자동화 스크립트 개발 (Linux/OS, 40h)
- 학습 기간: 2026-05-07 ~ 진행 중
