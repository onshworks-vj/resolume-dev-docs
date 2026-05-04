---
title: Resolume Arena / Avenue 7 — OSC 레퍼런스
version: Resolume 7.20+ (7.26까지 검증)
last_updated: 2026-04-30 (검증완료)
---

# Resolume OSC 레퍼런스 (Arena & Avenue 7)

**Resolume Arena와 Avenue 7.x**를 OSC로 제어하기 위한 최신 실전 레퍼런스. 아래의 모든 경로는 7.20 이상에서 유효하며, 공식 `resolume.com/support/en/osc`와 활발히 유지보수되고 있는 Bitfocus Companion 모듈(`bitfocus/companion-module-resolume-arena`) 기준으로 검증됨.

> **이 문서가 필요한 이유.** 인터넷에 떠도는 OSC 튜토리얼 대부분이 아직 Resolume 4/5 시절의 `/layer1/clip3/...` 같은 네임스페이스를 쓰고 있음. 그 네임스페이스는 **Resolume 7에서 제거됨**. 현재는 `/composition`으로 시작하는 계층 구조.

---

## 1. 설정

**Preferences → OSC**에서 활성화:

| 설정 | 설명 | 기본값 |
|---|---|---|
| Incoming OSC port | Resolume이 듣는 UDP 포트 | **7000** |
| Outgoing OSC host | 피드백 송신 대상 | `127.0.0.1` |
| Outgoing OSC port | 피드백용 UDP 포트 | **7001** |
| OSC output preset | 어떤 메시지를 내보낼지 결정 | "Output All OSC Messages" 권장 |

**양방향 규칙.** Resolume은 쿼리 응답을 **요청을 보낸 포트로** 되돌려 보냄 — 설정의 outgoing port가 아님. `?` 쿼리에 대한 응답을 받으려면, 듣고 있는 같은 UDP 소켓에서 보내야 함.

---

## 2. 주소 규칙

Resolume 7의 OSC 주소는 이런 형태:

```
/composition/layers/1/clips/3/connect
```

기억할 세 가지:

1. **항상 `/composition`으로 시작.** 단축 네임스페이스는 없음.
2. **인덱스는 1부터.** Layer 1, clip 1, deck 1 — layer 0은 없음.
3. **트리거는 값이 필요.** 정수 `1`을 보내면 트리거되고, `0`으로 풀어줌. Resolume은 상승 엣지를 트리거 이벤트로 처리.

### 값 수정자(prefix)

설정 가능한 파라미터는 첫 번째 OSC 인자(string)로 다음 prefix를 받고, 그 다음에 값을 받음:

| Prefix | 의미 | 예시 |
|---|---|---|
| (없음) | 절대값 설정 | `0.5` |
| `+` | 더함 | `"+", 1` |
| `-` | 뺌 | `"-", 1` |
| `*` | 곱함 | `"*", 2` |
| `?` | 현재 값 조회 (값 인자 없음) | `"?"` |

> **주의 — float에 대한 상대 수정자.** Resolume 7.x에서 `+` / `-` / `*` 상대 수정자는 **float** 파라미터에서 안정적으로 동작하지 않음. **integer** 인자만 일관되게 동작. 불투명도 같은 부동소수 파라미터를 점진적으로 변경하려면 클라이언트에서 절대값을 계산해 보낼 것. ([Companion 이슈 #9](https://github.com/bitfocus/companion-module-resolume-arena/issues/9), Resolume 포럼 t=21959.)

> **팁.** 주소를 가장 빠르게 알아내는 방법: Resolume에서 **Shortcuts → Edit OSC** 모드 진입 후, 원하는 컨트롤을 클릭. Shortcuts 패널에 정확한 주소가 표시됨.

---

## 3. Composition (전체)

| 주소 | 타입 | 설명 |
|---|---|---|
| `/composition/master` | <span class="tag float">float 0–1</span> | 전체 마스터 출력 레벨 |
| `/composition/video/opacity` | <span class="tag float">float 0–1</span> | 컴포지션 비디오 불투명도 |
| `/composition/audio/volume` | <span class="tag float">float 0–1</span> | 컴포지션 오디오 볼륨 |
| `/composition/speed` | <span class="tag float">float 0–2</span> | 마스터 재생 속도 (1.0 = 정상) |
| `/composition/direction` | <span class="tag int">int</span> | 재생 방향 (`0` 역방향, `1` 정방향, `2` 정지, `3` 랜덤) |
| `/composition/disconnectall` | <span class="tag trigger">trigger</span> | 재생 중인 모든 클립 정지 |
| `/composition/connectnextcolumn` | <span class="tag trigger">trigger</span> | 다음 컬럼 트리거 |
| `/composition/connectprevcolumn` | <span class="tag trigger">trigger</span> | 이전 컬럼 트리거 |
| `/composition/connectspecificcolumn` | <span class="tag int">int</span> | 인덱스로 컬럼 트리거 (1부터) |
| `/composition/selectnextdeck` | <span class="tag trigger">trigger</span> | 다음 데크 선택 |
| `/composition/selectprevdeck` | <span class="tag trigger">trigger</span> | 이전 데크 선택 |
| `/composition/selectspecificdeck` | <span class="tag int">int</span> | 인덱스로 데크 선택 |
| `/composition/name` | <span class="tag string">string</span> | 컴포지션 이름 (읽기/쓰기) |
| `/composition/recorder/record` | <span class="tag bool">bool</span> | 녹화 토글 (Arena) |

### 템포

| 주소 | 타입 | 설명 |
|---|---|---|
| `/composition/tempocontroller/tempo` | <span class="tag float">float</span> | BPM (범위 2.0–500.0) |
| `/composition/tempocontroller/tempotap` | <span class="tag trigger">trigger</span> | 탭 템포 |
| `/composition/tempocontroller/resync` | <span class="tag trigger">trigger</span> | 플레이헤드 재동기화 |
| `/composition/tempocontroller/tempopush` | <span class="tag trigger">trigger</span> | 페이즈를 앞으로 미세조정 |
| `/composition/tempocontroller/tempopull` | <span class="tag trigger">trigger</span> | 페이즈를 뒤로 미세조정 |
| `/composition/tempocontroller/tempoinc` | <span class="tag trigger">trigger</span> | BPM 1 증가 |
| `/composition/tempocontroller/tempodec` | <span class="tag trigger">trigger</span> | BPM 1 감소 |
| `/composition/tempocontroller/tempodividetwo` | <span class="tag trigger">trigger</span> | BPM 절반 |
| `/composition/tempocontroller/tempomultiplytwo` | <span class="tag trigger">trigger</span> | BPM 두 배 |
| `/composition/tempocontroller/metronome` | <span class="tag bool">bool</span> | 메트로놈 사운드 토글 |

### 크로스페이더 (컴포지션 레벨 전용)

Resolume의 크로스페이더는 컴포지션 레벨 — **레이어별 크로스페이더 주소는 없음**. 레이어는 `crossfadergroup` 파라미터로 A 또는 B에 참여.

| 주소 | 타입 | 설명 |
|---|---|---|
| `/composition/crossfader/phase` | <span class="tag float">float -1..1</span> | A/B 크로스페이더 위치 |
| `/composition/crossfader/sidea` | <span class="tag trigger">trigger</span> | A 사이드로 스냅 |
| `/composition/crossfader/sideb` | <span class="tag trigger">trigger</span> | B 사이드로 스냅 |
| `/composition/crossfader/curve` | <span class="tag int">int</span> | 커브 모양 |
| `/composition/crossfader/behaviour` | <span class="tag int">int</span> | 동작 모드 |
| `/composition/crossfader/video/mixer/blendmode` | <span class="tag int">int</span> | 크로스페이더용 블렌드 모드 |

### 대시보드 링크

Resolume은 8개의 사용자 매핑 가능한 컴포지션 단축:

```
/composition/dashboard/link1   …   /composition/dashboard/link8
```

레이어/클립별로도 존재 (`/composition/layers/{L}/dashboard/link{N}`, `…/clips/{C}/dashboard/link{N}`).

---

## 4. Layers (레이어)

`{L}` = 레이어 인덱스 (1부터).

| 주소 | 타입 | 설명 |
|---|---|---|
| `/composition/layers/{L}/select` | <span class="tag trigger">trigger</span> | 레이어 선택 |
| `/composition/layers/{L}/clear` | <span class="tag trigger">trigger</span> | 이 레이어의 활성 클립 정지 |
| `/composition/layers/{L}/bypassed` | <span class="tag bool">bool</span> | 레이어 바이패스 |
| `/composition/layers/{L}/solo` | <span class="tag bool">bool</span> | 레이어 솔로 |
| `/composition/layers/{L}/master` | <span class="tag float">float 0–1</span> | 레이어 마스터 |
| `/composition/layers/{L}/name` | <span class="tag string">string</span> | 레이어 이름 (읽기/쓰기) |
| `/composition/layers/{L}/video/opacity` | <span class="tag float">float 0–1</span> | 레이어 불투명도 |
| `/composition/layers/{L}/audio/volume` | <span class="tag float">float 0–1</span> | 레이어 볼륨 |
| `/composition/layers/{L}/audio/pan` | <span class="tag float">float</span> | 레이어 오디오 팬 |
| `/composition/layers/{L}/speed` | <span class="tag float">float</span> | 레이어 재생 속도 |
| `/composition/layers/{L}/direction` | <span class="tag int">int</span> | `0` 역방향 / `1` 정방향 / `2` 정지 / `3` 랜덤 |
| `/composition/layers/{L}/transition/duration` | <span class="tag float">float</span> | 크로스페이드 시간(초) |
| `/composition/layers/{L}/connectnextclip` | <span class="tag trigger">trigger</span> | 레이어의 다음 클립 트리거 |
| `/composition/layers/{L}/connectprevclip` | <span class="tag trigger">trigger</span> | 레이어의 이전 클립 트리거 |
| `/composition/layers/{L}/connectspecificclip` | <span class="tag int">int</span> | 이 레이어에서 인덱스로 클립 트리거 |
| `/composition/layers/{L}/autopilot` | <span class="tag int">int</span> | 오토파일럿 모드 |
| `/composition/layers/{L}/faderstart` | <span class="tag bool">bool</span> | 페이더 시작 활성 |
| `/composition/layers/{L}/ignorecolumntrigger` | <span class="tag bool">bool</span> | 컬럼 트리거 무시 |
| `/composition/layers/{L}/playmode` | <span class="tag int">int</span> | 레이어 재생 모드 |
| `/composition/layers/{L}/crossfadergroup` | <span class="tag int">int</span> | 레이어를 크로스페이더 사이드(Off/A/B)에 할당 |
| `/composition/layers/{L}/maskmode` | <span class="tag int">int</span> | 레이어 마스크 모드 |

### 비디오 믹서 (레이어별)

| 주소 | 타입 | 설명 |
|---|---|---|
| `/composition/layers/{L}/video/mixer/blendmode` | <span class="tag int">int</span> | 클립 레이어의 블렌드 모드 (Add, Lighten 등) |
| `/composition/layers/{L}/video/transition/mixer/blendmode` | <span class="tag int">int</span> | 클립 트랜지션 시 사용할 블렌드 모드 |

---

## 5. Clips (클립)

`{L}` = 레이어, `{C}` = 컬럼 인덱스.

| 주소 | 타입 | 설명 |
|---|---|---|
| `/composition/layers/{L}/clips/{C}/connect` | <span class="tag trigger">trigger</span> | 클립 트리거 (가장 자주 쓰는 동작) |
| `/composition/layers/{L}/clips/{C}/select` | <span class="tag trigger">trigger</span> | 트리거 없이 선택만 |
| `/composition/layers/{L}/clips/{C}/name` | <span class="tag string">string</span> | 클립 이름 (읽기/쓰기) |
| `/composition/layers/{L}/clips/{C}/video/opacity` | <span class="tag float">float</span> | 클립별 불투명도 오버라이드 |
| `/composition/layers/{L}/clips/{C}/audio/volume` | <span class="tag float">float</span> | 클립별 볼륨 |

### 트랜스포트 (플레이헤드)

| 주소 | 타입 | 설명 |
|---|---|---|
| `/composition/layers/{L}/clips/{C}/transport/position` | <span class="tag float">float 0–1</span> | 정규화된 플레이헤드 위치 |
| `/composition/layers/{L}/clips/{C}/transport/position/behaviour/speed` | <span class="tag float">float</span> | 클립별 속도 |
| `/composition/layers/{L}/clips/{C}/transport/position/behaviour/playdirection` | <span class="tag int">int</span> | `0` 역방향 / `1` 정방향 / `2` 정지 / `3` 랜덤 |
| `/composition/layers/{L}/clips/{C}/transport/position/behaviour/playmode` | <span class="tag int">int</span> | 재생 모드 (loop / once / bounce / random) |
| `/composition/layers/{L}/clips/{C}/transport/position/behaviour/playmodeaway` | <span class="tag int">int</span> | 클립 비연결 시 재생 모드 |
| `/composition/layers/{L}/clips/{C}/transport/position/behaviour/syncmode` | <span class="tag int">int</span> | BPM 동기화 모드 |
| `/composition/layers/{L}/clips/{C}/transport/position/behaviour/dividetwo` | <span class="tag trigger">trigger</span> | 재생 속도 절반 |
| `/composition/layers/{L}/clips/{C}/transport/position/behaviour/multiplytwo` | <span class="tag trigger">trigger</span> | 재생 속도 두 배 |

### 큐포인트 (클립별)

| 주소 | 타입 | 설명 |
|---|---|---|
| `/composition/layers/{L}/clips/{C}/transport/cuepoints/setparams/set{N}` | <span class="tag trigger">trigger</span> | 현재 플레이헤드를 큐포인트 N에 저장 |
| `/composition/layers/{L}/clips/{C}/transport/cuepoints/jumpparams/jump{N}` | <span class="tag trigger">trigger</span> | 저장된 큐포인트 N으로 점프 |

### 기타 클립 파라미터

| 주소 | 타입 | 설명 |
|---|---|---|
| `/composition/layers/{L}/clips/{C}/autopilot` | <span class="tag int">int</span> | 클립별 오토파일럿 |
| `/composition/layers/{L}/clips/{C}/beatsnap` | <span class="tag int">int</span> | 비트 스냅 값 |
| `/composition/layers/{L}/clips/{C}/cliptarget` | <span class="tag int">int</span> | 클립 타겟 |
| `/composition/layers/{L}/clips/{C}/cliptriggerstyle` | <span class="tag int">int</span> | 트리거 스타일 |
| `/composition/layers/{L}/clips/{C}/faderstart` | <span class="tag bool">bool</span> | 페이더 시작 |
| `/composition/layers/{L}/clips/{C}/ignorecolumntrigger` | <span class="tag bool">bool</span> | 컬럼 트리거 무시 |
| `/composition/layers/{L}/clips/{C}/transporttype` | <span class="tag int">int</span> | 트랜스포트 타입 |

> **클립별 오디오 볼륨**은 빌드별로 다름. 일부 Resolume 빌드에서는 `/composition/layers/{L}/clips/{C}/audio/volume`이 노출되고, 다른 빌드에선 오디오 볼륨이 레이어 전용. 사용 전에 **Shortcuts → Edit OSC**로 확인.

---

## 6. Columns (컬럼)

`{C}` = 컬럼 인덱스.

| 주소 | 타입 | 설명 |
|---|---|---|
| `/composition/columns/{C}/connect` | <span class="tag trigger">trigger</span> | 전체 컬럼 발사 (모든 레이어) |
| `/composition/columns/{C}/name` | <span class="tag string">string</span> | 컬럼 이름 (읽기/쓰기) |
| `/composition/columns/{C}/selected` | <span class="tag bool">bool</span> | 읽기 전용 피드백: 이 컬럼이 선택됐는지 |

> **"컬럼 선택" 액션은 없음.** Resolume은 컬럼을 연결(connect) 없이 선택하는 OSC 주소를 제공하지 않음. 컬럼을 발사하려면 `connect`(자동으로 선택됨), 현재 선택을 읽으려면 `selected`에 `?`를 보낼 것.

---

## 7. Decks (데크)

`{D}` = 데크 인덱스.

| 주소 | 타입 | 설명 |
|---|---|---|
| `/composition/decks/{D}/select` | <span class="tag trigger">trigger</span> | 데크 선택 |
| `/composition/decks/{D}/name` | <span class="tag string">string</span> | 데크 이름 (읽기/쓰기) |

---

## 8. Layer Groups (레이어 그룹)

Resolume 7부터 **레이어 그룹**이 도입됨. 이름이 프로토콜에 따라 다름 — 별칭이 아니라 두 개의 별개 네임스페이스:

| 프로토콜 | 네임스페이스 |
|---|---|
| **OSC** | `/composition/groups/{G}/...` |
| **REST / WebSocket** | `/composition/layergroups/{G}/...` |

프로토콜을 섞어 쓰면(예: OSC로 보내고 WebSocket으로 듣기) 두 형태 모두 보일 수 있음. Bitfocus Companion 모듈도 OSC 송신엔 `groups`, REST API 호출엔 `layergroups`를 씀.

`{G}` = 그룹 인덱스.

| 주소 (OSC) | 타입 | 설명 |
|---|---|---|
| `/composition/groups/{G}/select` | <span class="tag trigger">trigger</span> | 그룹 선택 |
| `/composition/groups/{G}/bypassed` | <span class="tag bool">bool</span> | 그룹 전체 바이패스 |
| `/composition/groups/{G}/solo` | <span class="tag bool">bool</span> | 그룹 솔로 |
| `/composition/groups/{G}/master` | <span class="tag float">float</span> | 그룹 마스터 |
| `/composition/groups/{G}/speed` | <span class="tag float">float</span> | 그룹 속도 |
| `/composition/groups/{G}/direction` | <span class="tag int">int</span> | `0` 역방향 / `1` 정방향 / `2` 정지 / `3` 랜덤 |
| `/composition/groups/{G}/video/opacity` | <span class="tag float">float</span> | 그룹 불투명도 |
| `/composition/groups/{G}/audio/volume` | <span class="tag float">float</span> | 그룹 볼륨 |
| `/composition/groups/{G}/columns/{C}/connect` | <span class="tag trigger">trigger</span> | 그룹 안에서 컬럼 트리거 |
| `/composition/groups/{G}/columns/{C}/select` | <span class="tag trigger">trigger</span> | 그룹 안에서 컬럼 선택 |
| `/composition/groups/{G}/disconnectlayers` | <span class="tag trigger">trigger</span> | 그룹의 모든 클립 정지 |

---

## 9. Effects (이펙트)

이펙트는 파라미터 트리를 따라 접근. 이펙트 파라미터는 그것이 붙은 엔티티(컴포지션/그룹/레이어/클립) 아래에 위치:

```
/composition/video/effects/{N}/...
/composition/layers/{L}/video/effects/{N}/...
/composition/layers/{L}/clips/{C}/video/effects/{N}/...
/composition/groups/{G}/video/effects/{N}/...
```

`{N}` = 이펙트 체인 내 슬롯 인덱스 (1부터). 각 이펙트 공통 노출:

| 하위 주소 | 타입 | 설명 |
|---|---|---|
| `/bypassed` | <span class="tag bool">bool</span> | 이펙트 바이패스 |
| `/mix` | <span class="tag float">float 0–1</span> | Wet/dry 믹스 |
| `/{paramName}` | <span class="tag float">float</span> | 이펙트 고유 파라미터 (camelCase) |

> **이펙트 주소 알아내기.** Resolume에서 **이펙트 파라미터 우클릭 → Shortcuts → Edit OSC → 파라미터 클릭**. 표시되는 주소가 정확하며, 현재 이펙트 순서를 반영.

### 자주 만나는 파라미터 타입

- `float` (대부분) — 0–1 정규화
- `int` (옵션 선택) — 이산 enum
- `bool` (토글)
- `color` — 보통 `r`, `g`, `b`, `a` float로 분리되거나 hex int 단일

---

## 10. Source / Generator 파라미터

Resolume 내장 소스(Solid, Gradients 등)와 FFGL/ISF 제너레이터의 파라미터는 클립의 `video/source/` 하위에 노출:

```
/composition/layers/{L}/clips/{C}/video/source/{paramName}
```

파라미터 이름은 Resolume UI에 보이는 이름과 동일 (camelCase, 공백 없음).

---

## 11. 상태 조회 (`?`)

문자열 인자 `"?"`만 담아 보내면 현재 값을 받음:

```
/composition/layers/1/video/opacity   "?"
```

Resolume이 같은 UDP 소켓(요청 보낸 포트)으로 현재 값을 적절한 타입으로 회신.

---

## 12. 업데이트 수신

OSC output preset을 **Output All OSC Messages**로 설정하면 Resolume이 지속적으로 브로드캐스트:

- 모든 파라미터 변경 (수동/자동 무관)
- 템포, 비트, 페이즈
- 활동 피드백 (클립 트리거, 선택)

설정된 outgoing 포트(기본 7001)에서 수신 — 송신과 동일한 주소 체계.

---

## 13. 예시

### 레이어 2의 컬럼 3 클립 트리거

```
/composition/layers/2/clips/3/connect   1
```

### 컴포지션 마스터 75% 설정

```
/composition/master   0.75
```

### 레이어 1 불투명도 10% 증가

상대 수정자는 Resolume 7에서 float 값에 안정적이지 않음(§2 참조). 클라이언트에서 절대값을 계산해 보낼 것:

```
# 현재 값을 0.4로 조회한 뒤
/composition/layers/1/video/opacity   0.5
```

### 탭 템포, 그리고 재동기화

```
/composition/tempocontroller/tempotap    1
/composition/tempocontroller/resync      1
```

### 레이어 2 현재 불투명도 조회

```
/composition/layers/2/video/opacity   "?"
```

### 모두 정지

```
/composition/disconnectall   1
```

---

## 14. 자주 하는 실수

1. **트리거 패턴은 `1` 다음 `0`.** "안 발사된다" 이슈의 대부분은 `1`만 보냈기 때문. Companion은 `1` 보내고 ~50ms 대기 후 `0`을 보냄. 직접 컨트롤러 만들 때도 똑같이.
2. **`/layer1/...` 형식은 안 됨.** 그건 Resolume 4/5 네임스페이스. `/composition/layers/1/...`로 마이그레이션할 것.
3. **인덱스는 1부터.** layer `0`으로 보내면 무시됨.
4. **컨텍스트 상대 주소는 `selectedlayer`.** Resolume이 `/composition/selectedlayer/...`와 `/composition/selectedclip/...`을 노출. 사용자가 선택한 대상에 작용시키고 싶을 때.
5. **OSC output preset 확인.** "Output All OSC Messages"가 선택돼 있지 않으면 대부분의 피드백 메시지를 못 봄.
6. **이펙트 슬롯 인덱스는 재배열 시 바뀜.** 쇼 도중 오퍼레이터가 이펙트 순서를 바꿀 수 있다면 슬롯 번호를 하드코딩하지 말 것. REST/WebSocket API의 `/by-id/...`로 안정적인 ID 사용.

---

## 15. OSC 너머: REST와 WebSocket API

상태가 풍부한 제어(썸네일, 구조 변경, 안정적 ID)가 필요하면 REST/WebSocket API 사용. 동일 포트.

- REST: `http://<host>:<port>/api/v1/composition`
- WebSocket: `ws://<host>:<port>/api/v1`
- 기본 포트: Arena/Avenue **8080**, Wire **8081** (Preferences → Webserver)
- API 문서: `http://<host>:<port>/api/v1/` (Swagger UI 라이브 호스팅)
- 도입: **Resolume 7.8**.

### 파라미터 액션 (WebSocket)

소문자 동사: `subscribe`, `unsubscribe`, `set`, `get`, `reset`, `trigger`.

```json
{ "action": "set", "parameter": "/composition/layers/1/video/opacity", "value": 0.5 }
```

`value`는 `set`에만 필요. 파라미터 경로는 `/parameter/by-id/{id}`(재배열에 견디는 안정 ID) 또는 OSC와 동일한 논리 경로 사용.

### 구조 액션 (WebSocket)

엔티티 추가/제거를 위한 별도 액션 패밀리, `path` + 옵션 `body`:

```json
{ "action": "post",   "id": "my_layer", "path": "/composition/layers/add" }
{ "action": "remove", "path": "/composition/layers/3" }
```

응답에 `id`가 그대로 돌아오고 성공 시 `error: null`.

> 재배열을 견디는 안정적인 파라미터 ID, 클립 썸네일, 구조 편집(레이어/컬럼/이펙트 추가·삭제), 변경 구독이 필요할 때는 REST/WebSocket API 사용.

---

## 출처

- [resolume.com/support/en/osc](https://resolume.com/support/en/osc) — 공식 OSC 지원 페이지
- [resolume.com/support/en/restapi](https://resolume.com/support/en/restapi) — REST API
- [resolume.com/support/en/websocket-api](https://resolume.com/support/en/websocket-api) — WebSocket API
- [resolume.com/docs/restapi/](https://resolume.com/docs/restapi/) — Swagger 레퍼런스
- [github.com/bitfocus/companion-module-resolume-arena](https://github.com/bitfocus/companion-module-resolume-arena) — 가장 활발하게 유지보수되는 서드파티 레퍼런스
