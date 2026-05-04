---
title: Resolume 개발 문서
version: Resolume Arena / Avenue 7.20+
last_updated: 2026-04-30
---

# Resolume 7 개발 문서

**Resolume Arena와 Avenue 7.x**의 OSC 제어, FFGL 플러그인 개발, ISF 셰이더 작성을 다루는 개발자용 문서. Resolume 7.20+ 현재 상태 기준, 공식 문서와 활발히 유지보수되는 서드파티 레퍼런스로 검증.

> **이 문서가 필요한 이유.** 인터넷의 Resolume 튜토리얼 대부분이 아직 Resolume 4/5 시절을 참조함 — OSC 네임스페이스가 틀리고, SDK 경로는 deprecated이고, 셰이더 패턴은 구식. 이 문서는 현행 자료와 FFGL master 브랜치 기준으로 작성됨.

## 언어

- **한국어** — 이 인덱스 → 아래 문서 참조
- **English** — [README.en.md](README.en.md)

## 문서

### OSC 제어

OSC 가능한 어떤 애플리케이션(TouchOSC, Companion, Max/MSP, 직접 짠 코드)에서든 Resolume 제어. 현재의 `/composition/...` 네임스페이스, 값 수정자, 상태 조회, 그리고 관련 REST/WebSocket API 다룸.

- [osc/osc-reference.ko.md](osc/osc-reference.ko.md)

### FFGL 플러그인 개발 (C++)

공식 FFGL SDK로 네이티브 이펙트/소스/믹서를 C++로 빌드. SDK 셋업, 플러그인 라이프사이클, 파라미터 타입, 컬러 컴포넌트, 오디오 리액티브, 자주 하는 실수까지.

- [ffgl/ffgl-guide.ko.md](ffgl/ffgl-guide.ko.md)

### ISF 셰이더 작성

JSON 프리픽스가 붙은 GLSL 프래그먼트 셰이더로 Resolume 이펙트/소스/트랜지션 작성. 빌드 시스템도, SDK도 없음 — `.fs` 파일을 떨어뜨리면 Resolume에 나타남.

- [isf/isf-guide.ko.md](isf/isf-guide.ko.md)

### Wire 패치 개발

Resolume Wire의 노드 기반 패치로 Source / Effect / Mixer 빌드. 절차적 비주얼, 오디오 리액티브, ISF 셰이더, OSC/MIDI 로직을 코드 없이 결합. `.wired` / `.cwired`로 컴파일해 Arena/Avenue에 로드.

- [wire/wire-guide.ko.md](wire/wire-guide.ko.md)

## 디렉토리 구조

```
resolume-dev-docs/
├── index.html                    ← GitHub Pages 진입점 (= README.en.html)
├── README.{en,ko}.{md,html}      ← 인덱스, MD/HTML 양쪽
├── osc/
│   └── osc-reference.{en,ko}.{md,html}
├── ffgl/
│   └── ffgl-guide.{en,ko}.{md,html}
├── isf/
│   └── isf-guide.{en,ko}.{md,html}
├── wire/
│   └── wire-guide.{en,ko}.{md,html}
└── assets/
    └── style.css                 ← HTML 사이트 공통 스타일
```

Markdown이 정본. `.html`은 미리 빌드되어 커밋되며 **GitHub Pages**로 빌드 단계 없이 그대로 서비스됨. Markdown을 수정하면 커밋 전에 매칭되는 HTML도 같이 갱신해야 함.

## 어떤 도구를 쓸지

| 하고 싶은 일 | 사용 |
|---|---|
| 컨트롤러에서 클립 트리거 / 파라미터 변경 | OSC |
| 복잡한 투어 제어 표면, 상태 피드백, 구조 조작 | REST/WebSocket API (OSC 문서 §15) |
| 새 비주얼 이펙트나 제너레이터를 빠르게, 프래그먼트 셰이더 로직만 | ISF |
| 호스트 오디오 데이터 접근, 풀 GL 제어의 피드백 버퍼 | ISF (audio 입력 + PASSES) |
| 여러 소스 + 수학 + 오디오 리액티브 + OSC/MIDI 로직을 시각적으로 구성 | **Wire** |
| 고성능 네이티브 이펙트, 커스텀 GL 파이프라인, 단일 프래그먼트 셰이더 그 이상 | FFGL |
| 비기술 사용자에게 매끈한 UX로 배포 | FFGL (컴파일된 바이너리) 또는 Juicebar 통한 Wire `.cwired` |

## 최신성 메모

- **OSC**: 여기 정리된 네임스페이스는 Resolume 7.20+, 활발히 유지보수되는 Bitfocus Companion 모듈 소스, Resolume의 정적 OSC 리스트로 검증됨. `/composition/...` 경로는 Resolume 7.0 이래 안정적 — 포인트 릴리스에서 바뀐 건 주로 추가된 경로(groups, layer groups, transport 하위)이고 breaking change는 없음.
- **FFGL**: master SDK 브랜치 기준이며, 클래스명·FF_TYPE_* 상수·라이프사이클 메서드를 `github.com/resolume/ffgl`의 헤더와 대조 검증. 구버전 Resolume이 타깃이면 `v2.2` 태그 사용.
- **ISF**: 스펙은 수년째 안 움직임. ISF v2.0이 현행이며, 함수명은 VVISF-GL 레퍼런스 파서와 대조 검증.
- **Wire**: Wire 7.26과 Resolume 공식 지원 페이지 기준. 패치 파일 형식(`.wired` / `.cwired`), 노드 카테고리, flow 타입, 그리고 최근 추가(7.20 Color 타입, 7.21 파라미터 그룹화, 7.24 Patch Navigator)가 현행 릴리스 반영.

> **검증.** 이 문서는 Resolume 공식 지원 페이지, FFGL SDK 소스, ISF 스펙 저장소, Bitfocus Companion 모듈, 그리고 여러 커뮤니티 OSC 구현체와 교차 검증됨. 각 문서의 출처 섹션 참고.

## 기여 / 업데이트

Resolume이 새 릴리스에서 OSC 경로나 FFGL 기능을 추가했을 때:

1. 해당 `.md` 파일 편집 (영어 먼저, 한국어 미러링).
2. 프론트매터의 version 라인 업데이트.
3. 커밋.
