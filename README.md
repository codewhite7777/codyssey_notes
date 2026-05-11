# Codyssey 학습 노트

> [Codyssey](https://www.codyssey.kr/) 부트캠프 과제를 진행하며 누적되는 학습 자산.
> Linux/OS, Python, Git, AI/SW 기초 개념을 정리·기록.

**현재 진행 단계**: B1-1 시스템 관제 자동화 스크립트 개발 (Linux/OS, 40h)

---

## 이 레포는 무엇인가

학생 입장에서 정답을 빠르게 만들기보다 **개념을 폭넓게 이해하고 사고 과정을 형성**하는 데 초점을 둔 학습 노트 모음입니다.

이 레포(**codyssey_notes**)는 **학습 자산만** 보관합니다. 실제 과제 산출물 코드는 과제별 별도 레포(`codyssey_b1_1`, `codyssey_b1_2`, ...)에 보관되며, 두 레포가 명확히 분리되어 있습니다.

## 디렉토리 구조

```
codyssey_notes/
├── README.md                          # 이 파일 (레포 랜딩)
├── INDEX.md                           # 15개 과제 진행 보드
├── CLAUDE.md                          # 학습 우선 원칙 + Claude Code 컨벤션
│
├── codyssey_b1_1_study/               # B1-1 학습 노트 묶음
│   ├── README.md                      # 노트 인덱스
│   ├── filesystem-tree.md
│   ├── users-and-groups.md
│   ├── file-permissions.md
│   ├── shell-environment.md
│   ├── process-and-signals.md
│   ├── ssh-deep-dive.md
│   ├── sshd-config.md
│   ├── ports-and-listening.md
│   ├── firewall-ufw-vs-firewalld.md
│   └── posix-acl.md
│
├── codyssey_b1_2_study/  (예정)       # B1-2 학습 노트 묶음
├── codyssey_b1_3_study/  (예정)
└── ...
│
├── retrospectives/                    # 주차별·과제별 회고
└── .claude/commands/                  # 학습 워크플로우 슬래시 명령
```

별도 레포로 분리된 과제 산출물:
```
github.com/codewhite7777/codyssey_b1_1   ← B1-1 실제 산출물 (코드·setup·docs)
github.com/codewhite7777/codyssey_b1_2   ← B1-2 (예정)
...
```

## 두 레포의 역할 분리

| 레포 | 무엇 들어가나 | 무엇이 안 들어가나 | 어디서 쓰나 |
|---|---|---|---|
| **codyssey_notes** (이 레포) | 학습 노트 (`_study` 폴더), INDEX, 회고, 학습 컨벤션 | 실행 코드·스크립트·산출물 | Mac에서 학습 참조, GitHub 웹에서 학습 기록 |
| **codyssey_b1_N** | monitor.sh·setup 스크립트·명세·수행내역서·증거 | 학습 노트 (해당 study 폴더 참조) | 클러스터에서 git clone → 실행 → 평가 |

→ 클러스터에는 **얇은 과제 레포만** 클론, 학습 자료는 codyssey_notes에 별도 보관.

## 학습 방식

### Learning-First 원칙 ([CLAUDE.md](./CLAUDE.md))

1. **코드 작성 전에 개념 먼저** — 정의 / 왜 / 원리 / 예시 / 오해
2. **정답보다 사고 과정** — 즉시 코드 덤프 X, 단계적 힌트
3. **"왜?"** 를 항상 동반 — 모든 결정에 이유
4. **작업 후 개념 추출** — `codyssey_b1_N_study/`에 정리
5. **과제 간 연결 명시** — 새 학습을 기존 노트와 잇기

### 학습 깊이 모드 (Learning Depth)

학습자의 CS 배경에 맞춰 노트의 깊이를 조정.

| 모드 | 적용 시점 | 특징 |
|---|---|---|
| **중급자 (기본)** | CS 기초 있음, 시스템/운영 학습 중 | 기술 용어 적극, 동작 원리·설계 의도 깊이, 인접 토픽 연결 |
| **초심자** | 특정 토픽이 완전히 새로움 | 일관된 비유, 용어 풀이 분리, 다이어그램 다수 |
| **고급/ODIN** | 본인 명시 요청 | 글로벌 ODIN 컨벤션 (압축, VS) |

### 노트 양식

각 노트는 다음 구조:
- **TLDR** (한 줄 핵심) + **GitHub alerts** (NOTE/TIP/WARNING)
- **Mermaid 다이어그램** (GitHub 자동 렌더링)
- **명령 + 실제 출력** 페어링
- **`<details>` 접기** 로 심화 내용 분리

## 진행 현황

전체 보드는 **[INDEX.md](./INDEX.md)** 참조.

| ID | 과제 | 상태 | 학습 노트 |
|---|---|---|---|
| B1-1 | 시스템 관제 자동화 스크립트 | 🟡 학습 중 | [codyssey_b1_1_study/](./codyssey_b1_1_study/) (10개) |
| B1-2 | 프로세스·리소스 트러블슈팅 | 🔵 시작 전 | — |
| B1-3 | 파일 기반 가계부 | 🔵 시작 전 | — |
| B1-4 | Git 협업 워크플로우 | 🔵 시작 전 | — |
| ... | (나머지 11개) | 🔵 | — |

## Claude Code 슬래시 명령

| 명령 | 용도 |
|---|---|
| `/learn <주제>` | 개념을 단계적으로 깊이 설명 |
| `/connect` | 현재 작업과 기존 학습 노트 연결 찾기 |
| `/extract` | 작업에서 학습 개념 추출 → 해당 `_study` 폴더에 정리 |
| `/start <id> <name>` | 새 과제 레포 부트스트랩 |
| `/recap` | 세션 회고 노트 초안 |

## 참고

이 레포는 **부트캠프 학습자 1인의 개인 노트**입니다. 정확성을 보장하지 않으며, 같은 학습 단계의 다른 분에게 참고가 되었으면 하는 마음으로 공개합니다.

오류·개선 제안은 issue로 환영합니다.

## 사용 도구

- **Claude Code** (Anthropic) — 학습 페어로 활용
- **GitHub** — 노트 백업·동기화
- **Markdown** + **Mermaid** — 노트 양식 + 시각화
