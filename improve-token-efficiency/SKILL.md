---
name: improve-token-efficiency
description: Claude Code 세션 JSONL 로그를 파싱하여 토큰/컨텍스트 효율 리포트(HTML 대시보드 + $ 절감안)를 생성하는 스킬. '토큰 효율 분석', '세션 효율', '비용 분석', 'Claude Code usage report', 'analyze token efficiency', 'show session cost', 'improve token efficiency', 'efficiency score', 'how much did I spend on Claude', 'session report' 등의 요청에 반드시 트리거할 것. 사용자가 레포의 Claude Code 사용 패턴/비용/효율을 알고 싶어하거나 어떻게 줄일지 문의할 때 사용. 단일 세션이 아니라 '여러 세션 전체'에 대한 집계·점수화·시각화가 필요한 모든 경우에 적용.
---

# Improve Token Efficiency

Claude Code 가 `~/.claude/projects/<encoded-repo-path>/*.jsonl` 에 남기는 세션 로그를 파싱해서, 레포 단위로 **토큰 사용·캐시 효율·비용·점수**를 집계하고 개선안을 제시하는 HTML 대시보드를 만든다.

왜 JSONL을 직접 파싱하는가: Anthropic CLI가 세션당 모든 assistant 메시지의 `usage` 필드를 기록해 둔다. 여기에 `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`, 그리고 `cache_creation.ephemeral_5m_input_tokens` / `ephemeral_1h_input_tokens` 이 다 들어있다. 그래서 별도 API 호출 없이 레포 하나의 전체 비용 구조를 재구성할 수 있다.

## 동작 흐름

### 1. 대상 레포 결정

사용자가 경로를 명시했으면 그것을 쓴다. 아니면 현재 작업 디렉터리(`pwd`)를 기준으로 삼는다. 경로의 영숫자가 아닌 모든 문자(`/`, `\`, `:`, `_`, `.` 등)를 `-`로 바꾼 이름으로 `~/.claude/projects/<encoded>/` 를 구성한다. 예:

- `/Users/jaeha/Projects/GrowthNote` → `-Users-jaeha-Projects-GrowthNote`
- `C:\Users\User\Project\Stock_analyst` → `C--Users-User-Project-Stock-analyst`

스크립트가 이 변환을 자동으로 한다. 사용자에게 확인할 필요 없다.

### 2. 세션 분석 실행

```bash
python3 ~/.claude/skills/improve-token-efficiency/scripts/analyze_sessions.py \
    --repo "$(pwd)" \
    --out /tmp/session_analysis.json
```

- `--repo` 없으면 cwd. `--sessions-dir` 로 직접 지정도 가능.
- 출력: `/tmp/session_analysis.json` (totals + per-session stats + scores).
- 빈 세션(usage 기록 0)은 자동 제외한다.
- 응답 하나가 내용 블록(thinking, text, tool_use)마다 한 줄씩, 같은 `usage`를 반복해 기록된다. 그래서 usage는 `message.id`당 한 번만 센다(줄마다 더하면 약 2.4배 과대 집계).
- 로그는 UTF-8로 읽는다. 인코딩을 지정하지 않으면 한국어 Windows(cp949)에서 모든 줄이 디코딩에 실패해 전 세션이 usage 0이 된다.

스크립트가 세션별로 다음을 계산한다:

| 필드 | 설명 |
|---|---|
| `input_tokens` | 캐시 미스 입력 토큰 |
| `cache_create_5m / 1h` | 5분/1시간 에페메럴 캐시 쓰기 |
| `cache_read` | 캐시 읽기 (재사용) |
| `output_tokens` | 실제 생성 토큰 |
| `cache_hit_ratio` | `cache_read / total_input` |
| `redundant_reads` | 동일 파일을 Read 툴로 반복 호출한 횟수 합 |
| `tool_use_counter` | 툴별 호출 수 |
| `cost_usd` | 모델별 가격 × 토큰 |

### 3. 점수화 (Rubric)

`scripts/analyze_sessions.py` 가 각 세션을 4개 지표로 0–100 점화하고 가중 합산한다. 아래 기준을 바꾸고 싶으면 `score_session()` 안의 가중치 상수(`0.40` 등)를 수정한다. 플래그로 바꾸는 기능은 없다.

| 지표 | 가중치 | 측정 |
|---|---|---|
| **Cache utilization** | 40% | `cache_read / total_input`. 0.85 이상이 만점. 캐시 재사용률이 높을수록 매 turn 토큰 재전송 비용이 줄어든다. |
| **Output density** | 20% | `output / total_input`. ~2%가 sweet spot. 너무 낮으면 읽기만 많고 산출 부족 (thrash), 너무 높으면 긴 독백. |
| **Read redundancy** | 20% | `redundant_reads / total_reads`. 같은 파일을 여러 번 Read하면 감점. Grep/Glob으로 위치 먼저 좁히지 않은 패턴. |
| **Tool economy** | 20% | 출력 1k 토큰당 툴 호출 수. 2–10 사이가 건강한 범위. >20은 탐색 thrash 신호. |

최종 등급: A+ ≥90, A ≥85, A- ≥80, B+ ≥75, B ≥70, B- ≥65, C+ ≥60, C ≥55, C- ≥50, D ≥40, F <40.

### 4. 대시보드 생성

```bash
python3 ~/.claude/skills/improve-token-efficiency/scripts/build_dashboard.py \
    --input /tmp/session_analysis.json \
    --out /tmp/efficiency_report.html
open /tmp/efficiency_report.html
```

Chart.js 기반 단일 HTML 파일이며 네트워크가 되는 브라우저면 CDN만으로 즉시 열린다. 포함 요소:

- **KPI 카드**: 누적 비용, 총 토큰, 캐시 적중률, 평균 점수, 툴 호출 수
- **비용 내역 doughnut**: cache write 1h/5m, cache read, output, input
- **Pareto 차트**: 세션별 비용 내림차순 + 누적 %
- **등급 히스토그램**: A+ ~ F 분포
- **Cost vs Score 버블 scatter**: x=비용(log), y=점수, 버블=툴 호출 수
- **Rubric 표**: 각 기준별 평균 점수 및 설명
- **Top 20 세션 표**: 비용·툴·캐시 적중률·중복 read·점수·등급
- **개선 카드 6종**: 각 개선안마다 예상 절감 $ 표시

### 5. 사용자에게 요약 보고

HTML을 연 뒤 Korean 으로 최상위 수치와 가장 큰 개선 3가지를 2–3줄로 요약. 예:

> 106개 세션 / $1,248 / 평균 B+. 상위 14개가 전체 비용의 63% 차지. **Opus→Sonnet 라우팅, 장시간 세션 /compact, 이미지 세션 관리** 3가지만 해도 ~40% 절감 기대.

## 가격 기준 (per 1M tokens, USD)

스크립트에 하드코딩된 기본값. 새 모델이 나오면 `scripts/analyze_sessions.py` 상단 `PRICING` dict 수정.

| Model | Input | Output | Cache write 5m | Cache write 1h | Cache read |
|---|---|---|---|---|---|
| Opus 5.5 (`claude-opus-5-5`) | 4.0 | 20.0 | 5.0 | 8.0 | 0.20 |
| Sonnet 5.5 (`claude-sonnet-5-5`) | 2.0 | 10.0 | 2.5 | 4.0 | 0.20 |
| Opus 4.x (`claude-opus-4-6`, `claude-opus-4-7`) | 15.0 | 75.0 | 18.75 | 30.0 | 1.50 |
| Sonnet 4.x (`claude-sonnet-4-6`) | 3.0 | 15.0 | 3.75 | 6.0 | 0.30 |
| Haiku 4.x (`claude-haiku-4-5-*`) | 0.80 | 4.0 | 1.0 | 1.6 | 0.08 |
| `<synthetic>` | 0 | 0 | 0 | 0 | 0 |

캐시 쓰기는 입력 단가의 1.25배(5m), 2배(1h)다. `build_dashboard.py` 상단의 `OPUS_*`·`SONNET` 상수(비용 내역 차트와 개선안 산정용)도 Opus 5.5 / Sonnet 5.5 단가이므로 기본 모델이 바뀌면 함께 고친다.

## 개선안 산정 로직

`build_dashboard.py` 가 6가지 휴리스틱으로 예상 절감을 계산한다. 보수적으로 단순 합산이 전체 비용의 40–50%를 넘으면 단일 카드에 "중복 고려 시 실제로는 30–40% 수준" 주석을 붙인다.

1. **Opus → Sonnet 라우팅**: 같은 토큰의 30%를 Sonnet 5.5로 돌린다고 가정 → `0.3 × (실제 비용 − 같은 토큰을 Sonnet 5.5 단가로 계산한 비용)` 절감. 입력·출력·캐시 쓰기는 2배 차이지만 캐시 읽기는 둘 다 $0.20라, 토큰 종류별로 따로 계산한다. (`/fast`는 Sonnet 전환이 아니라 Opus 고속 모드로 더 비싸다.)
2. **장시간 세션 `/compact`**: 상위 14개 세션(Pareto 기준)의 `cache_read` 의 30% 감소 가정.
3. **이미지 세션 관리**: 이미지 포함 세션에서 장당 ~40k 토큰이 세션 내내 유지된다고 보고, 이의 50%가 컴팩션으로 사라지는 경우.
4. **Cache TTL 1h → 5m**: 전체 1h 캐시 쓰기의 40%가 5m 로 충분하다고 가정 → `cache_create_1h × 0.4 × ($8 − $5) / 1M` 절감. TTL 설정 방법은 Claude Code 문서로 확인한다(검증되지 않은 환경변수를 안내하지 않는다).
5. **세션 scope 축소**: 상위 14개 외 세션의 `cache_read` 15% 감소 가정.
6. **중복 Read 제거**: `redundant_reads × 3000토큰 × (cache write + 10 re-read)` 비용.

이 숫자는 참고용이다. 사용자가 "실제로 얼만큼 절감되나?" 물으면 "휴리스틱 추정치, 실행 후 재측정 필요" 라고 답한다.

## 에지 케이스 / 주의사항

- **세션 디렉터리가 없다**: 레포가 Claude Code 로 한 번도 열린 적 없는 경우. "분석할 세션 없음" 안내하고 종료.
- **모든 세션이 빈 usage**: 아주 오래된 CLI 버전일 수 있음. 스크립트가 자동으로 걸러내고 남은 게 없으면 종료.
- **가격 미등록 모델**: 기본값으로 Opus 5.5 가격 적용하고, 콘솔에 `[warn] unknown model: <id>, applying Opus default` 출력.
- **Python 3.9+**: dataclass / walrus 사용 없음, 순수 stdlib 만 필요. pip install 불필요.
- **Chart.js CDN 의존**: HTML이 `cdn.jsdelivr.net`에서 Chart.js 4.4.0을 불러온다. 오프라인 환경이면 차트가 그려지지 않는다(로컬 복사본을 넣는 옵션은 없다).

## 품질 기준 (skill 자체에 대한)

이 스킬이 유용하려면:

1. **동일 결과의 재현성**: 같은 세션 디렉터리를 두 번 돌리면 동일 JSON / 동일 점수 나와야 함.
2. **cwd 외 레포 지원**: `--repo /path/to/other-repo` 로도 동작.
3. **가격 업데이트 용이**: 새 모델이 나와도 `PRICING` dict 한 줄 추가로 대응.
4. **개선안 수치 방어 가능**: 각 항목마다 계산 로직을 주석으로 명기. 사용자가 "왜 이 숫자?" 물으면 답할 수 있어야 함.
5. **보고 언어**: 사용자가 Korean 사용자임이 명확하면 보고는 Korean, 용어는 영어 병기 (예: "캐시 적중률(cache hit ratio)").
