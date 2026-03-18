# SPEC-MULTI-003: 출력 스크롤백 및 설정 관리

## 메타데이터

| 항목 | 값 |
|------|-----|
| SPEC ID | SPEC-MULTI-003 |
| 제목 | 출력 스크롤백 및 설정 관리 |
| 상태 | Planned |
| 우선순위 | Medium |
| 생성일 | 2026-03-18 |
| 관련 SPEC | SPEC-MULTI-001, SPEC-MULTI-002 (선행 필수) |

---

## Environment (환경)

### 기술 환경

- SPEC-MULTI-001/002의 멀티패널 TUI 위에 구축
- 추가 의존성:
  - serde 1.0.217 + toml 0.8.19 (설정 파일)
  - clap 4.5.27 (CLI 인자)
  - dirs 6.0.0 (사용자 디렉토리 경로)

### 참고 구현

- **alacritty**: 스크롤백 버퍼 구현, 링 버퍼 기반 메모리 효율적 설계 참고
- **zellij**: 설정 파일 구조 및 기본값 시스템 참고

---

## Assumptions (가정)

### 기술 가정

1. 링 버퍼 기반 스크롤백이 메모리 효율적이다
   - 신뢰도: 높음 (alacritty에서 검증된 패턴)
   - 검증 방법: 10,000줄 스크롤백에서 메모리 사용량 측정

2. TOML 설정 파일이 사용자에게 충분히 직관적이다
   - 신뢰도: 높음 (Rust 생태계 표준, Cargo.toml과 동일 형식)

3. 사용자별 설정 디렉토리 위치를 dirs 크레이트로 안정적으로 찾을 수 있다
   - 신뢰도: 높음
   - Windows: %APPDATA%\multi-shell\, macOS: ~/Library/Application Support/multi-shell/

---

## Requirements (요구사항)

### 유비쿼터스 요구사항

- **[REQ-201]** 시스템은 **항상** 각 셸의 출력 히스토리를 최소 10,000줄까지 보존해야 한다
- **[REQ-202]** 시스템은 **항상** 설정 파일이 없을 경우 합리적인 기본값을 적용해야 한다

### 이벤트 기반 요구사항

- **[REQ-210]** **WHEN** 사용자가 Normal 모드에서 스크롤 단축키(PageUp/PageDown 또는 Ctrl+U/D)를 누르면 **THEN** 활성 패널의 출력이 스크롤되어야 한다
- **[REQ-211]** **WHEN** 사용자가 Normal 모드에서 검색 단축키(/)를 누르면 **THEN** 검색 모드로 전환되어 출력 내 텍스트 검색이 가능해야 한다
- **[REQ-212]** **WHEN** 사용자가 검색어를 입력하고 Enter를 누르면 **THEN** 일치하는 텍스트가 하이라이트되고 n/N으로 다음/이전 결과로 이동해야 한다
- **[REQ-213]** **WHEN** 애플리케이션이 시작되면 **THEN** 설정 파일을 로드하고 적용해야 한다
- **[REQ-214]** **WHEN** CLI 인자로 설정값이 제공되면 **THEN** 설정 파일의 값을 오버라이드해야 한다

### 상태 기반 요구사항

- **[REQ-220]** **IF** 스크롤 모드(출력이 맨 아래가 아닌 위치)이면 **THEN** 새 출력이 도착해도 자동 스크롤하지 않아야 한다
- **[REQ-221]** **IF** 스크롤을 맨 아래로 되돌리면 **THEN** 자동 스크롤을 재개해야 한다
- **[REQ-222]** **IF** 설정 파일에 잘못된 값이 있으면 **THEN** 경고를 로그에 기록하고 기본값을 사용해야 한다

### 선택 요구사항

- **[REQ-230]** **가능하면** 검색 결과를 실시간 하이라이트(incremental search) 제공

---

## Specifications (사양)

### 스크롤백 버퍼 설계

```rust
struct ScrollbackBuffer {
    lines: VecDeque<String>,  // 링 버퍼 (FIFO)
    max_lines: usize,         // 기본: 10,000
    scroll_offset: usize,     // 현재 스크롤 위치 (0 = 맨 아래)
}
```

- VecDeque로 FIFO 링 버퍼 구현
- max_lines 초과 시 가장 오래된 줄 자동 제거
- scroll_offset으로 현재 뷰 위치 추적

### 검색 기능 설계

```
Search Mode (Normal 모드에서 / 입력)
├── 검색어 입력 (하단 입력바 표시)
├── Enter: 검색 실행, 첫 번째 결과로 이동
├── n: 다음 결과
├── N: 이전 결과
├── Escape: 검색 종료, Normal 모드 복귀
└── (선택) 실시간 하이라이트
```

### 설정 파일 구조

파일 위치: `{config_dir}/multi-shell/config.toml`

```toml
# Multi-Shell 설정

[general]
default_shell = ""          # 빈 문자열이면 OS 기본 셸 사용
max_shells = 8              # 최대 동시 셸 수

[display]
scrollback_lines = 10000    # 스크롤백 버퍼 크기
show_statusbar = true       # 상태바 표시 여부

[keybindings]
# 향후 사용자 정의 단축키 지원 예약
```

### CLI 인자

```
multi-shell [OPTIONS]

Options:
  -c, --config <PATH>     설정 파일 경로 (기본: 자동 감지)
  -s, --shell <SHELL>     사용할 셸 (기본: OS 기본값)
  -n, --num <N>           시작 시 생성할 셸 수 (기본: 1)
  --scrollback <LINES>    스크롤백 버퍼 크기 (기본: 10000)
  -v, --verbose           디버그 로그 활성화
  -V, --version           버전 표시
  -h, --help              도움말 표시
```

### 추가/수정 모듈

```
src/
├── config/
│   ├── mod.rs              # Config 모듈 진입점
│   └── settings.rs         # Settings 구조체, TOML 파싱, CLI 오버라이드
├── ui/
│   └── panel.rs (수정)     # 스크롤백 렌더링, 검색 하이라이트 추가
└── shell/
    └── manager.rs (수정)   # ScrollbackBuffer 통합
```

---

## Traceability (추적성)

| 태그 | 관련 파일 |
|------|-----------|
| SPEC-MULTI-003 | spec.md, plan.md, acceptance.md |
| REQ-201 | src/shell/manager.rs (ScrollbackBuffer) |
| REQ-202, REQ-213, REQ-214, REQ-222 | src/config/settings.rs |
| REQ-210 ~ REQ-212 | src/ui/panel.rs, src/input/handler.rs |
| REQ-220, REQ-221 | src/ui/panel.rs |
| REQ-230 | src/ui/panel.rs (선택 구현) |
