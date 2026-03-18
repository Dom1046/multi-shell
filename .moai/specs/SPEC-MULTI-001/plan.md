# SPEC-MULTI-001: 구현 계획

## 관련 SPEC

- SPEC ID: SPEC-MULTI-001
- 제목: MVP - 코어 엔진 및 단일 셸 실행

---

## 마일스톤

### Primary Goal: 프로젝트 스캐폴딩 및 빌드 체인 구축

- Cargo.toml 의존성 정의 (ratatui, crossterm, tokio, portable-pty, anyhow, log)
- 디렉토리 구조 생성 (src/main.rs, src/app.rs, src/ui/, src/shell/, src/input/)
- 기본 main.rs 엔트리 포인트 (터미널 초기화/복원)
- CI 파이프라인 기초 (cargo fmt --check, cargo clippy, cargo test)
- 난이도: 쉬움

### Secondary Goal: PTY 셸 생성 및 I/O 처리

- portable-pty를 래핑하는 PTY 추상화 레이어 구현 (src/shell/pty.rs)
- ShellManager 구현: 셸 생성, 출력 수신, 프로세스 종료 (src/shell/manager.rs)
- tokio 채널 기반 비동기 셸 출력 스트리밍
- Windows ConPTY 동작 검증
- 난이도: 어려움 (크로스 플랫폼 PTY 핸들링이 핵심 복잡도)

### Tertiary Goal: TUI 렌더링 및 이벤트 루프

- ratatui + crossterm 기반 터미널 초기화/복원 (raw mode, alternate screen)
- App 구조체 및 tokio::select! 기반 이벤트 루프 구현 (src/app.rs)
- 단일 패널 렌더링: 셸 출력을 Paragraph 위젯으로 표시 (src/ui/panel.rs)
- InputHandler: 키 입력을 활성 셸 PTY로 라우팅 (src/input/handler.rs)
- 종료 단축키(Ctrl+Q) 처리
- 난이도: 중간

### Final Goal: 통합 테스트 및 안정화

- 단위 테스트 작성 (ShellManager, InputHandler)
- 통합 테스트: 셸 생성 -> 명령 실행 -> 출력 확인 플로우
- 에러 처리 강화 (셸 비정상 종료, PTY 할당 실패)
- 성능 벤치마크 (시작 시간, 렌더링 지연)
- 난이도: 중간

---

## 기술 접근법

### PTY 통합 전략

portable-pty 크레이트를 직접 사용하되, 자체 추상화 레이어를 두어 향후 백엔드 교체 가능성을 확보한다.

```
PtyBackend trait
├── PortablePtyBackend (portable-pty 기반, 기본)
└── (향후) NativePtyBackend (OS 네이티브 직접 호출, 선택적)
```

핵심 고려사항:
- portable-pty의 `CommandBuilder`로 OS 기본 셸 실행
- `MasterPty`에서 비동기 읽기를 위해 tokio 스레드 활용
- 셸 출력은 `tokio::sync::mpsc` 채널로 이벤트 루프에 전달

### 이벤트 루프 아키텍처

tokio 기반 비동기 이벤트 루프를 채택한다:

1. **crossterm EventStream**: 키보드/마우스 입력을 async stream으로 수신
2. **셸 출력 채널**: mpsc 채널을 통해 셸 출력 비동기 수신
3. **틱 인터벌**: 100ms 간격으로 UI 강제 갱신 (idle 상태에서도 커서 깜빡임 등)

`tokio::select!`로 세 이벤트 소스를 동시 대기하며, 어느 것이든 먼저 도착하면 처리한다.

### 렌더링 전략

ratatui의 즉시 모드(immediate mode) 렌더링을 활용한다:

- 매 프레임마다 전체 UI를 다시 계산
- `terminal.draw()` 내에서 레이아웃 계산 및 위젯 렌더링
- crossterm 백엔드의 diff 기반 렌더링으로 실제 터미널 출력 최소화

---

## 아키텍처 설계 방향

### 레이어드 아키텍처

```
┌─────────────────────────────────────┐
│          main.rs (진입점)            │
├─────────────────────────────────────┤
│          app.rs (이벤트 루프)         │
├──────────┬──────────┬───────────────┤
│  ui/     │  shell/  │  input/       │
│  렌더링  │  PTY/프로 │  키 입력      │
│          │  세스     │  라우팅       │
└──────────┴──────────┴───────────────┘
```

- 각 레이어는 app.rs를 통해서만 간접 통신
- ui/와 shell/ 사이에 직접 의존성 없음
- 이벤트/메시지 기반 모듈 간 통신

### 에러 처리 전략

- `anyhow::Result`를 기본 에러 타입으로 사용
- 복구 가능한 에러: 로그 + 사용자 알림 (상태바 메시지)
- 복구 불가능한 에러: 터미널 복원 후 panic 메시지 출력
- 셸 프로세스 에러는 격리하여 다른 셸에 영향 없음

---

## 리스크 및 대응 계획

| 리스크 | 발생 가능성 | 영향도 | 대응 계획 |
|--------|------------|--------|-----------|
| Windows ConPTY 호환성 문제 | 중간 | 높음 | portable-pty 이슈 트래커 모니터링, 필요시 windows-rs 직접 사용 |
| PTY 비동기 읽기 블로킹 | 중간 | 높음 | tokio::task::spawn_blocking으로 별도 스레드 처리 |
| 터미널 복원 실패 | 낮음 | 높음 | panic hook에서 터미널 복원 보장 |
| 대량 출력 시 UI 버벅임 | 중간 | 중간 | 출력 버퍼링 + 렌더링 스로틀링 |

---

## 파일 구조 및 모듈 분할 계획

| 파일 | 예상 라인 수 | 핵심 책임 |
|------|-------------|-----------|
| src/main.rs | 30-50 | 진입점, 터미널 초기화/복원, App 실행 |
| src/app.rs | 100-150 | App 구조체, 이벤트 루프, 모듈 조율 |
| src/ui/mod.rs | 10-20 | UI 모듈 re-export |
| src/ui/panel.rs | 50-80 | 단일 패널 렌더링, 출력 표시 |
| src/shell/mod.rs | 10-20 | 셸 모듈 re-export |
| src/shell/manager.rs | 80-120 | ShellManager, 셸 생명주기 관리 |
| src/shell/pty.rs | 60-100 | PTY 추상화, portable-pty 래핑 |
| src/input/mod.rs | 10-20 | 입력 모듈 re-export |
| src/input/handler.rs | 40-60 | 키 입력 분류 및 라우팅 |

---

## 테스트 전략 개요

### 단위 테스트

- ShellManager: 셸 생성, 출력 수신, 종료 처리
- InputHandler: 단축키 인식, 일반 입력 분류
- PTY 래퍼: 모킹을 통한 인터페이스 검증

### 통합 테스트

- 엔드투엔드: 앱 시작 -> 셸 생성 -> 명령 입력 -> 출력 확인 -> 종료
- 에러 시나리오: PTY 할당 실패, 셸 비정상 종료

### 테스트 도구

- cargo test (내장 테스트)
- cargo-nextest (병렬 실행)
- cargo-tarpaulin (커버리지 85%+ 목표)
