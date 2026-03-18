# SPEC-MULTI-001: MVP - 코어 엔진 및 단일 셸 실행

## 메타데이터

| 항목 | 값 |
|------|-----|
| SPEC ID | SPEC-MULTI-001 |
| 제목 | MVP: 코어 엔진 및 단일 셸 실행 |
| 상태 | Planned |
| 우선순위 | High |
| 생성일 | 2026-03-18 |
| 관련 SPEC | 없음 (기반 SPEC) |

---

## Environment (환경)

### 기술 환경

- **언어**: Rust 1.85.0 (stable)
- **빌드 시스템**: Cargo 1.85.0
- **핵심 라이브러리**:
  - ratatui 0.29.0 (TUI 프레임워크)
  - crossterm 0.28.1 (터미널 제어)
  - tokio 1.43.0 (비동기 런타임)
  - portable-pty 0.8.1 (PTY 관리)
  - anyhow 1.0.95 (에러 처리)
  - log 0.4.25 + env_logger 0.11.6 (로깅)

### 플랫폼 환경

- **우선 지원**: Windows 11 (ConPTY)
- **추후 지원**: macOS, Linux (POSIX PTY)
- **터미널**: 터미널 에뮬레이터 독립적 동작

### 참고 프로젝트

- **zellij**: Rust 기반 터미널 멀티플렉서, 레이아웃/플러그인 시스템 참고
- **alacritty**: Rust 기반 GPU 가속 터미널, PTY 처리 및 성능 최적화 참고
- **wezterm**: Rust 기반 터미널, portable-pty 크레이트 원작자 프로젝트

---

## Assumptions (가정)

### 기술 가정

1. portable-pty 0.8.1이 Windows 11 ConPTY를 안정적으로 지원한다
   - 신뢰도: 높음 (wezterm에서 검증된 크레이트)
   - 검증 방법: Windows 11에서 기본 셸(PowerShell, cmd) 생성 테스트
   - 위험: 일부 ANSI 이스케이프 시퀀스 미지원 시 폴백 처리 필요

2. ratatui + crossterm 조합이 Windows에서 60fps 렌더링이 가능하다
   - 신뢰도: 높음 (Rust TUI 사실상 표준 조합)
   - 검증 방법: 단일 패널 렌더링 벤치마크

3. tokio 이벤트 루프와 crossterm 이벤트 스트림이 충돌 없이 통합된다
   - 신뢰도: 중간 (crossterm의 EventStream이 tokio 호환)
   - 검증 방법: 입력 이벤트 + PTY 출력 동시 처리 테스트

### 사용자 가정

1. 사용자는 터미널 환경에 익숙한 개발자이다
2. 기본 셸은 OS 기본값 사용 (Windows: PowerShell, macOS/Linux: $SHELL)

---

## Requirements (요구사항)

### 유비쿼터스 요구사항 (항상 충족)

- **[REQ-001]** 시스템은 **항상** 16ms 이하의 UI 프레임 렌더링 시간을 유지해야 한다
- **[REQ-002]** 시스템은 **항상** 키 입력에 대해 10ms 이하의 응답 시간을 보장해야 한다
- **[REQ-003]** 시스템은 **항상** 메모리 안전성을 보장해야 한다 (unsafe 블록 최소화)

### 이벤트 기반 요구사항

- **[REQ-010]** **WHEN** 사용자가 애플리케이션을 실행하면 **THEN** 500ms 이내에 기본 셸이 포함된 TUI 화면을 표시해야 한다
- **[REQ-011]** **WHEN** 사용자가 키보드 입력을 하면 **THEN** 활성 셸의 PTY로 해당 입력을 전달해야 한다
- **[REQ-012]** **WHEN** 셸 프로세스가 출력을 생성하면 **THEN** 해당 출력을 TUI 패널에 렌더링해야 한다
- **[REQ-013]** **WHEN** 사용자가 종료 단축키(Ctrl+Q)를 누르면 **THEN** 모든 셸 프로세스를 정리하고 애플리케이션을 종료해야 한다
- **[REQ-014]** **WHEN** 셸 프로세스가 비정상 종료되면 **THEN** 사용자에게 상태를 표시하고 새 셸 생성을 안내해야 한다

### 상태 기반 요구사항

- **[REQ-020]** **IF** 터미널 크기가 최소 요구사항(80x24) 미만이면 **THEN** 경고 메시지를 표시하고 축소 레이아웃을 적용해야 한다
- **[REQ-021]** **IF** 셸이 활성 상태이면 **THEN** 사용자 입력을 해당 셸의 PTY에 전달해야 한다

### 금지 요구사항

- **[REQ-030]** 시스템은 사용자 권한을 초과하는 프로세스를 실행**하지 않아야 한다**
- **[REQ-031]** 시스템은 외부 네트워크 통신을 수행**하지 않아야 한다**

---

## Specifications (사양)

### 모듈 구조

```
src/
├── main.rs                 # 엔트리 포인트, CLI 인자 파싱
├── app.rs                  # App 구조체, 이벤트 루프
├── ui/
│   ├── mod.rs              # UI 모듈 진입점
│   └── panel.rs            # 단일 패널 렌더링
├── shell/
│   ├── mod.rs              # 셸 모듈 진입점
│   ├── manager.rs          # ShellManager (셸 생명주기)
│   └── pty.rs              # PTY 추상화 (portable-pty 래핑)
└── input/
    ├── mod.rs              # 입력 모듈 진입점
    └── handler.rs          # InputHandler (키 분류 및 라우팅)
```

### 핵심 데이터 구조

#### App (애플리케이션 상태)

- `shells`: 셸 인스턴스 목록
- `active_shell`: 현재 활성 셸 인덱스
- `should_quit`: 종료 플래그
- `terminal`: ratatui Terminal 인스턴스

#### ShellInstance

- `id`: 고유 식별자
- `pty_pair`: PTY 마스터/슬레이브 쌍
- `child`: 자식 프로세스 핸들
- `output_buffer`: 출력 버퍼 (Vec<String>)
- `title`: 셸 제목

### 이벤트 루프 설계

```
loop {
    1. terminal.draw(|frame| ui::render(frame, &app))
    2. tokio::select! {
         _ = tick_interval.tick() => { /* UI 갱신 */ }
         event = crossterm_events.next() => { /* 입력 처리 */ }
         output = shell_output.recv() => { /* 셸 출력 수신 */ }
       }
    3. if app.should_quit { break; }
}
```

### 성능 목표

| 항목 | 목표치 |
|------|--------|
| 시작 시간 | 500ms 이하 |
| UI 렌더링 | 16ms 이하 (60fps) |
| 키 입력 응답 | 10ms 이하 |
| 메모리 (1셸) | 30MB 이하 |

---

## Traceability (추적성)

| 태그 | 관련 파일 |
|------|-----------|
| SPEC-MULTI-001 | spec.md, plan.md, acceptance.md |
| REQ-001 ~ REQ-003 | src/app.rs (이벤트 루프 성능) |
| REQ-010 ~ REQ-014 | src/shell/manager.rs, src/input/handler.rs |
| REQ-020 ~ REQ-021 | src/ui/panel.rs |
| REQ-030 ~ REQ-031 | src/shell/pty.rs |
