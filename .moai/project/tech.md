# Multi-Shell - 기술 문서

## 기술 스택 명세

### 핵심 기술

| 기술 | 버전 | 용도 |
|------|------|------|
| Rust | 1.85.0 (stable, 2025-02) | 주 개발 언어 |
| Cargo | 1.85.0 | 빌드 시스템 및 패키지 관리자 |

### 주요 라이브러리

| 라이브러리 | 버전 | 용도 |
|------------|------|------|
| ratatui | 0.29.0 | TUI 프레임워크 (터미널 UI 렌더링) |
| crossterm | 0.28.1 | 크로스 플랫폼 터미널 제어 |
| tokio | 1.43.0 | 비동기 런타임 (셸 I/O 처리) |
| portable-pty | 0.8.1 | 크로스 플랫폼 PTY 관리 |
| serde | 1.0.217 | 설정 직렬화/역직렬화 |
| toml | 0.8.19 | TOML 설정 파일 파싱 |
| clap | 4.5.27 | CLI 인자 파싱 |
| log | 0.4.25 | 로깅 인터페이스 |
| env_logger | 0.11.6 | 환경 변수 기반 로그 출력 |
| anyhow | 1.0.95 | 에러 처리 |
| dirs | 6.0.0 | 크로스 플랫폼 디렉토리 경로 |

---

## 기술 선택 근거

### Rust를 선택한 이유

1. **크로스 플랫폼**: 단일 코드베이스로 Windows, macOS, Linux 바이너리 생성
2. **메모리 안전성**: 소유권 시스템으로 메모리 누수 및 레이스 컨디션 방지
3. **높은 성능**: C/C++ 수준의 성능, TUI 렌더링에 최적
4. **단일 바이너리 배포**: 외부 런타임 불필요, 설치 간편
5. **풍부한 TUI 생태계**: ratatui, crossterm 등 성숙한 라이브러리

### ratatui + crossterm 조합

- ratatui: Rust TUI 생태계에서 가장 활발하게 유지보수되는 프레임워크
- crossterm: Windows ConPTY 지원이 뛰어난 크로스 플랫폼 터미널 백엔드
- 두 라이브러리의 조합은 Rust TUI 개발의 사실상 표준

### tokio 비동기 런타임

- 여러 셸의 I/O를 동시에 비차단(non-blocking)으로 처리
- 이벤트 루프 기반 설계와 자연스럽게 통합
- Rust 비동기 생태계의 표준 런타임

---

## 개발 환경

### 필수 도구

| 도구 | 버전 | 용도 |
|------|------|------|
| rustup | latest | Rust 툴체인 관리 |
| cargo | 1.85.0+ | 빌드, 테스트, 의존성 관리 |
| git | 2.40+ | 버전 관리 |

### 권장 도구

| 도구 | 용도 |
|------|------|
| rust-analyzer | IDE 언어 서버 (자동완성, 타입 검사) |
| cargo-watch | 파일 변경 시 자동 빌드/테스트 |
| cargo-nextest | 빠른 테스트 실행기 |
| cargo-clippy | Rust 린터 |
| cargo-fmt | 코드 포매터 (rustfmt) |

### 개발 명령어

```bash
# 빌드
cargo build                    # 디버그 빌드
cargo build --release          # 릴리스 빌드

# 실행
cargo run                      # 디버그 모드 실행
cargo run --release            # 릴리스 모드 실행

# 테스트
cargo test                     # 전체 테스트
cargo test --lib               # 단위 테스트만
cargo test --test integration  # 통합 테스트만
cargo nextest run              # nextest로 실행 (병렬화)

# 품질 검사
cargo clippy -- -D warnings    # 린트 (경고를 에러로)
cargo fmt --check              # 포매팅 검사
cargo fmt                      # 자동 포매팅
```

---

## 테스트 전략

### 테스트 프레임워크

| 도구 | 용도 |
|------|------|
| cargo test (built-in) | 단위 테스트 및 통합 테스트 |
| cargo-nextest | 병렬 테스트 실행, 더 나은 출력 |
| cargo-tarpaulin | 커버리지 측정 |

### 테스트 레벨

| 레벨 | 위치 | 범위 | 목표 커버리지 |
|------|------|------|---------------|
| 단위 테스트 | 각 모듈 내 `#[cfg(test)]` | 개별 함수/구조체 | 85%+ |
| 통합 테스트 | `tests/integration/` | 모듈 간 상호작용 | 주요 시나리오 |
| 수동 테스트 | - | UI/UX 검증 | 모든 단축키, 레이아웃 |

### 테스트 원칙

- 모든 공개 API에 대해 단위 테스트 작성
- PTY/프로세스 관련 테스트는 모킹 활용
- UI 렌더링은 스냅샷 테스트 고려
- CI에서 `cargo clippy`와 `cargo fmt --check` 필수 실행

---

## CI/CD 및 배포

### CI 파이프라인 (GitHub Actions)

```yaml
# 주요 단계
- cargo fmt --check          # 포매팅 검증
- cargo clippy -- -D warnings # 린트 검증
- cargo test                  # 테스트 실행
- cargo build --release       # 릴리스 빌드
```

### 배포 대상

| 플랫폼 | 대상 | 형식 |
|---------|------|------|
| Windows | x86_64-pc-windows-msvc | .exe 바이너리 |
| macOS | x86_64-apple-darwin, aarch64-apple-darwin | 바이너리 |
| Linux | x86_64-unknown-linux-gnu | 바이너리 |

### 배포 방식

- GitHub Releases에 바이너리 첨부
- 향후 고려: cargo install 지원, Homebrew formula, Scoop bucket

---

## 성능 요구사항

| 항목 | 목표치 | 측정 방법 |
|------|--------|-----------|
| UI 프레임 렌더링 | 16ms 이하 (60fps) | 프레임 타이밍 로그 |
| 키 입력 응답 | 10ms 이하 | 입력-렌더링 지연 측정 |
| 셸 출력 처리량 | 10MB/s 이상 | 대량 출력 벤치마크 |
| 메모리 (기본) | 30MB 이하 | 셸 1개 기준 |
| 메모리 (셸당 추가) | 20MB 이하 | 스크롤백 버퍼 포함 |
| 시작 시간 | 500ms 이하 | cold start 기준 |

---

## 보안 요구사항

- 셸 프로세스는 사용자 권한으로만 실행
- 설정 파일에 민감 정보 저장 금지
- 외부 네트워크 통신 없음 (순수 로컬 애플리케이션)
- 입력 데이터 유효성 검증 (설정 파일 파싱 시)

---

## 기술적 제약 및 고려사항

### 알려진 제약

| 제약 | 영향 | 대응 방안 |
|------|------|-----------|
| Windows ConPTY 제한 | 일부 ANSI 이스케이프 시퀀스 미지원 가능 | portable-pty로 추상화, 폴백 처리 |
| 터미널 크기 제한 | 작은 터미널에서 패널 분할 제한 | 최소 크기 검증, 자동 레이아웃 조정 |
| UTF-8 와이드 문자 | CJK 문자 폭 계산 복잡 | unicode-width 크레이트 활용 |

### TRUST 5 원칙 적용

| 원칙 | 현재 상태 | 적용 계획 |
|------|-----------|-----------|
| Tested | 미도입 | cargo test + tarpaulin 85%+ 커버리지 |
| Readable | 미도입 | clippy 경고 0, 명확한 네이밍 규칙 |
| Unified | 미도입 | rustfmt 자동 포매팅, 일관된 에러 처리 |
| Secured | 미도입 | cargo-audit 의존성 취약점 검사 |
| Trackable | 미도입 | Conventional Commits, SPEC 연동 |

---

## 히스토리

| 날짜 | 내용 |
|------|------|
| 2026-03-18 | 기술 문서 최초 작성 |
