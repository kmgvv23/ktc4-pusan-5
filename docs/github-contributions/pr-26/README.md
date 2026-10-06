# #26 [AI] feature: 보고서 DB 저장 구현

원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/26

작성자: @hakyeoung
작성 시각: 2026-09-20T23:05:26Z
보관 시각 (UTC): 2026-10-06T15:19:25.015347+00:00

## PR 본문

## 이번에 한 일

LLM 이 생성한 종목 변동 요인 보고서 JSON 을 DB 에 넣는 계층을 만들었습니다. ai쪽 스키마가 아직 명확하지 않은 단계라 사람마다 출력물 스키마에 대해 생각하는 지점이 다를 것이라 생각해 db 저장 파트를 먼저 구현했습니다.

```
stock_move_reports                본체. 한 행 = 보고서 한 회차
stock_move_report_factors         원인 (순서 = 중요도)
stock_move_report_factor_sources  원인별 출처
stock_move_report_reviews         검수 판정. 이번엔 표만 만들고 쓰지 않습니다
```

`analyst_reports` 가 "증권사 애널리스트가 쓴 원문" 이고 이건 "우리가 생성한 설명" 이라 `stock_move_report` 접두사로 묶었습니다. backend 의 `report` / `report_block` / `report_citation` 과도 겹치지 않습니다 — 같은 Postgres 를 쓰므로 이름이 겹치면 서로의 표를 건드리게 됩니다.

AI 서버(배치)가 생성·검증·적재하고 backend 는 읽기만 하는 단방향입니다. backend 조회는 `verify_status = 'passed'` 를 전제합니다.

회의 결정대로 `ai/` 의 스키마는 **alembic** 으로 관리합니다(`ai/alembic/`). 리비전이 둘인데, `0001` 은 이미 `create_all` 로 만들어져 있던 `analyst_reports` 의 baseline 이고 `0002` 가 이번 표 네 개입니다. **쓰던 DB 라면 `uv run alembic stamp 0001` 을 한 번 찍고 `upgrade head` 하시면 됩니다.** 새 DB 는 `upgrade head` 만 하면 됩니다.

이번 범위 밖(후속 PR): 근거 검증기 연결, 재료 건수 게이트, 텔레그램 수집의 DB 화, backend 라우터·응답 스키마.

## 테이블 구조

```
stock_move_reports  1 ──< stock_move_report_factors  1 ──< stock_move_report_factor_sources
       │
       └──< stock_move_report_reviews        (이번엔 표만, 쓰지 않습니다)
```

자식 표는 전부 `ON DELETE CASCADE` 입니다.

### `stock_move_reports` — 보고서 한 회차

| 칼럼 | 타입 | | |
| --- | --- | --- | --- |
| `id` | BIGSERIAL | PK | |
| `run_id` | VARCHAR(200) | NULL | `runs/<ts>/parsed/<run_id>.json` 으로 되돌아가는 열쇠 |
| `ticker` | VARCHAR(6) | NULL | |
| `name` | VARCHAR(64) | NULL | 생성 시점의 종목명 |
| `target_date` | DATE | NULL | 출력 키는 `date` |
| `as_of` | TIMESTAMPTZ | NULL | 기준 시각. `date` + `"HH:MM"` 을 KST 로 합친 값 |
| `change_pct` | NUMERIC(8,2) | NULL | |
| `verdict` | VARCHAR(24) | NULL | `explained` / `partially_explained` / `no_clear_cause` |
| `summary_move` | TEXT | NULL | |
| `summary_main_cause` | TEXT | NULL | |
| `summary_counter` | TEXT | NULL | **NULL 이 정상값** |
| `summary_unexplained` | TEXT | NULL | |
| `terms` | JSONB | NULL | `[{plain, term}]` |
| `background` | JSONB | NULL | `{bullish[], bearish[], neutral[]}`, 항목은 `{text, watch}` 로 펴서 저장 |
| `not_found` | JSONB | NULL | 검수용. 화면에 안 나갑니다 |
| `raw_json` | JSONB | **NOT NULL** | 보고서 JSON 통째 |
| `parse_status` | VARCHAR(20) | NOT NULL | `ok` / `schema_violation` / `parse_failed`, 기본 `ok` |
| `parse_error` | TEXT | NULL | 무엇이 걸렸는지 줄 단위로 |
| `verify_status` | VARCHAR(20) | NOT NULL | `passed` / `failed` / `not_verified`, 기본 `not_verified` |
| `verify_error` | TEXT | NULL | |
| `prompt_version` | VARCHAR(32) | NULL | 후속 과제. 프롬프트에 버전 문자열이 아직 없습니다 |
| `generated_at` | TIMESTAMPTZ | NULL | LLM 생성 시각 |
| `loaded_at` | TIMESTAMPTZ | NOT NULL | `now()` |

인덱스 셋입니다. `(ticker, as_of)` / `(target_date)` / `(ticker, as_of) WHERE verify_status = 'passed'`.
마지막 것이 backend 조회용 부분 인덱스입니다 — 미검증·실패 행이 대부분일 수 있어서 갈랐습니다.

**자연키 `(ticker, target_date, as_of)` 에 UNIQUE 를 걸지 않았습니다.** 같은 기준시각을 다시 생성해도 덮어쓰지 않고 새 행으로 쌓습니다. 대신 멱등하지 않아서 같은 파일을 두 번 넣으면 행이 둘 생깁니다.

### `stock_move_report_factors` — 원인

| 칼럼 | 타입 | | |
| --- | --- | --- | --- |
| `id` | BIGSERIAL | PK | |
| `report_id` | BIGINT | FK → `stock_move_reports.id` | CASCADE |
| `order_index` | INTEGER | NOT NULL | 0-base. **배열 순서가 곧 중요도 순** |
| `claim` | TEXT | NULL | 화면에 나가는 원인 한 문장 |
| `detail` | TEXT | NULL | |
| `stance` | VARCHAR(16) | NULL | `bullish` / `bearish` / `neutral` |
| `direction_match` | BOOLEAN | NULL | `factors` 에는 `true` 만 옵니다. `false` 는 `summary_counter` 로 |
| `size_fit` | VARCHAR(16) | NULL | `sufficient` / `partial` / `insufficient` |
| `unconfirmed` | BOOLEAN | NULL | |

`UNIQUE (report_id, order_index)` + `report_id` 인덱스.

### `stock_move_report_factor_sources` — 원인별 출처

| 칼럼 | 타입 | | |
| --- | --- | --- | --- |
| `id` | BIGSERIAL | PK | |
| `factor_id` | BIGINT | FK → `stock_move_report_factors.id` | CASCADE |
| `order_index` | INTEGER | NOT NULL | |
| `source` | VARCHAR(16) | NOT NULL | **출처 종류.** 기본 `telegram`. 채널명이 아닙니다 |
| `channel` | VARCHAR(128) | NULL | 채널명 |
| `url` | TEXT | NULL | 텔레그램 메시지 URL |
| `datetime_kst` | TIMESTAMPTZ | NULL | |
| `quote` | TEXT | NULL | 원문 그대로 40자 이내. 화면에 안 나갑니다 |
| `match` | VARCHAR(16) | NULL | `direct` / `indirect`. 화면에 안 나갑니다 |
| `is_market_recap` | BOOLEAN | NULL | 화면에 안 나갑니다 |

`UNIQUE (factor_id, order_index)` + `factor_id` · `url` 인덱스.

### `stock_move_report_reviews` — 검수 판정 (이번엔 표만)

`id` / `report_id`(FK, CASCADE) / `decision`(NOT NULL) / `reviewer` / `note` / `created_at`.

보고서 표에 칼럼으로 붙이지 않은 이유는 AI 서버(적재)와 검수 쪽이 같은 행을 건드리면 단방향이 깨지기 때문입니다.

### 보고서 JSON 이 어디로 가는지

| JSON | 가는 곳 |
| --- | --- |
| `ticker` `name` `date` `as_of` `change_pct` `verdict` | `stock_move_reports` 동명 칼럼 (`date` → `target_date`) |
| `summary.move` / `.main_cause` / `.counter` / `.unexplained` | `summary_*` 네 칼럼으로 펴집니다 |
| `factors[]` | `stock_move_report_factors` 한 행씩 |
| `factors[].sources[]` | `stock_move_report_factor_sources` 한 행씩 |
| `terms` `background` `not_found` | JSONB 그대로 (`background` 만 `{text, watch}` 로 정규화) |
| 전체 | `raw_json` 에 한 번 더 통째로 |

**모든 정규화 칼럼이 nullable 입니다.** 검증 실패·스키마 위반 건도 버리지 않고 상태만 달아 적재하기 때문입니다. `ticker` 조차 못 읽는 회차가 있을 수 있어서, NOT NULL 을 걸면 정작 원인 분석에 제일 필요한 행이 통째로 사라집니다. NOT NULL 인 건 `raw_json` · `parse_status` · `verify_status` · `loaded_at` 뿐입니다.

## 리뷰에서 봐주셨으면 하는 곳

1. 에이전트가 내놓을 출력물, db에 저장하는 형태가 본인의 생각과 다르다면 코멘트 남겨주시고 노션에도 기록해주시면 감사하겠습니다. 다 같이 회의하고 납득해야 이후 진행이 수월하기 때문입니다.

2. 리뷰하게 되면 수정한 부분이 많다고 뜰텐데 클로드 코드로 구현하면서 저장 형식이 꼬이지 않게 하기 위해 실험 결과 파일을 이번 pr에 같이 업로드 했습니다. 그래서 test/fixtures/reports 안에 있는 md 파일을 따로 읽으실 필요는 없을 것 같고, stock_report_system도 실험 때 썼던 시스템 프롬프트를 그대로 들고 온 것이라 이번 리뷰에서 확인 할 필요는 없을 것 같습니다. 즉, 위에 언급된 파일들만 읽으면 되니 분량이 많다고 너무 부담 가지시지 않아도 될 것 같습니다..

## 고민했던 점 · 다른 선택지

### 입력을 DB 가 아니라 파일로 읽습니다

수집 원본(`analyst_reports`, `news`)은 DB 에 쌓는데 보고서 생성 입력만 파일입니다. 일부러 그렇게 했습니다. 저도 헷갈렸던 지점인데, 읽는건 에이전트가 생성한 보고서입니다. 생성된 보고서는 검증을 거쳐야 하는데, 검증되지 않은 보고서까지 DB에 저장했다가 불러오는 형태보단 일단 파일로 만들어 놓고, 검증에 성공한 보고서만 DB에 저장하는 것이 더 효율적이라고 판단했기 때문입니다.

- **보고서 생성 결과는 일회성 산출물입니다.** `runs/<타임스탬프>/parsed/*.json` 은 매 실행마다 새로 만들어지고, 다음 실행이 같은 것을 다시 만듭니다. 수집 원본처럼 "한 번 받아두면 원천이 사라져도 남아야 하는 것" 이 아닙니다. 입력 테이블을 두면 쓰고 나서 지울 표가 하나 더 생깁니다.
- **원본이 이미 보존됩니다.** 적재된 행의 `raw_json` 에 보고서 JSON 이 통째로 들어갑니다. 입력을 따로 쌓으면 같은 내용이 DB 안에 두 벌이 됩니다.
- **생성 쪽과 적재 쪽의 수명 주기가 다릅니다.** 보고서 생성(LLM 호출)은 실험 레포에서 돌고 이 레포로 옮겨오지 않았습니다. 중간에 표를 끼우면 두 레포가 스키마를 공유하게 돼서, 실험 쪽 출력이 바뀔 때마다 이쪽 마이그레이션이 필요해집니다. 파일이면 경계가 파일 포맷 하나입니다.
- **지금은 사람이 `runs/` 에서 파일을 가져다 둡니다.** 파이프라인 직결은 후속 PR 입니다. 그 사이에도 적재를 손으로 돌려볼 수 있어야 해서 파일 입력이 오히려 편합니다.

대안으로 `raw_report_runs` 같은 입력 표를 두고 거기서 읽는 것도 봤는데, 위 네 가지 중 어느 것도 해결하지 못하면서 표만 하나 늘어서 접었습니다.

### 입력 파일이 2단이라 파싱도 2단입니다

이 구조가 된 이유는 환각 예방을 위해서입니다. 회의 중에는 툴 트레이스까지 db에 넣자는 의견이 나왔으나, 코드를 통해 에이전트가 정리한 내용이 에이전트가 근거로 제시한 링크에 있는지 확인 가능하기 때문에 그것이 다르다면 저장하지 않으면 환각이 예방되기 때문입니다.

바깥은 실험 파이프라인의 실행 기록 래퍼고(보고서 저장 앞단계에서 처리할 부분), 보고서 JSON 은 `final_text` 안에 **문자열로** 들어 있습니다. 래퍼에서 읽는 건 다섯 개뿐입니다 — `final_text` · `json_status` · `schema_problems` · `is_error` · `asked_question`.

나머지는 의도적으로 흘려보냅니다. 툴 지표(`n_tool_calls` 등)는 툴을 막아둬서 언제나 0 이고, `pubdate_*` 는 당일 생성으로 바뀌면서 의미가 없어졌습니다. 래퍼는 실험 환경 산출물이라 필드가 늘거나 주는데, 모르는 필드에 걸려 죽으면 적재가 통째로 멈춥니다.

**래퍼의 `verdict` / `n_factors` / `n_sources` 는 저장하지 않습니다.** `final_text` 안의 값을 다시 센 것이라 두 군데 두면 어긋났을 때 무엇을 믿을지 문제가 됩니다. 단일 출처는 `final_text` 입니다.

### 시스템 프롬프트가 아직 완전하지 않습니다

테스트 때에는 현재 시스템 프롬프트로 잘 동작했고, 특별한 불편함을 느끼지 못해서 그대로 들고 왔습니다. 하지만 실제 서비스를 해보면 실험 환경과 다른 결과가 나올 수 있기에 시스템 프롬프트를 어기더라도 우리가 인지하고, 시스템 프롬프트를 수정할지, 이건 확실하게 차단하는게 맞는지 지금 정하는 건 이르다고 판단했기 때문입니다.

- **`background` 항목이 문자열이기도 `{"text":..., "watch":true}` 객체이기도 합니다.** 둘 다 정상값입니다. 모델에게 일관성을 요구하는 대신 적재가 `{text, watch}` 로 펴서 화면이 매번 타입을 확인하지 않게 했습니다. 원본 혼용은 `raw_json` 에 그대로 남습니다.
- **상한 초과를 잘라내지 않습니다.** `factors` 4 / `terms` 8 / `background` 전체 6·칸당 3 을 넘겨도 그대로 넣고 로그에만 남깁니다. 적재가 말없이 자르면 실행 점검 스크립트의 상한 위반 검사가 영원히 통과합니다. (실제로 픽스처의 SK하이닉스 건이 `background` 12개입니다.)
- **`summary.counter` 의 `null` 은 정상값입니다.** 방향이 어긋나는 재료가 없는 날과 `no_clear_cause` 인 날이 그렇습니다. 빈 문자열로 바꾸지 않습니다 — 화면이 "없음" 과 "못 채움" 을 구분해야 합니다.

### 임의로 정한 것

- **`change_pct` → `Numeric(8,2)`.** `analyst_reports.upside_pct` 와 같은 정밀도입니다. float 로 받으면 `-2.01` 이 `-2.0100000000000002` 로 굳는데, 루트 CLAUDE.md 가 "가공하지 않은 값을 준다" 를 요구하므로 DB 가 값을 흔들면 안 됩니다.
- **`as_of` → `date` + `"HH:MM"` 을 합쳐 KST aware datetime.** 루트 CLAUDE.md 가 naive datetime 을 금지하고 응답은 ISO 8601 이어야 합니다. 원본 `"15:30"` 은 `raw_json` 에 남아 진실의 출처가 둘로 갈리지 않습니다.
- **적재 시각은 `func.now()`(DB 시계).** `analyst_reports.collected_at` 관례를 따랐습니다. 배치가 여러 대가 되어도 적재 순서가 한 시계로 보존됩니다.
- **종목명(`name`)을 보고서마다 저장합니다.** 종목 마스터가 생기면 중복이지만, 프롬프트가 회차 간 참조를 금지해 각 회차가 독립 문서이므로 생성 시점의 이름이 그 문서의 일부입니다. 종목명은 실제로 바뀝니다.
- **출처 종류 칼럼(`source`)을 지금 만들었습니다.** 지금은 전부 `telegram` 이지만 뉴스·리포트 PDF 가 붙을 때 소스별 오탐률을 갈라 봐야 하고, 그때 추가하면 이미 쌓인 데이터가 전부 미상이 됩니다. 이름은 `analyst_reports.source` 관례를 따랐습니다.
- **`prompt_version` 은 nullable 입니다.** 지금 시스템 프롬프트에 버전 문자열이 없습니다. **후속 과제**로 칸만 만들어 뒀습니다.
- **`target_date` 라는 칼럼명.** 출력 키는 `date` 지만 칼럼명으로 쓰면 날짜 타입명과 겹쳐 읽기 나빠서, 프롬프트 **입력**이 같은 값을 부르는 이름을 썼습니다.

### 검증하지 않는 것

적재 모듈이 근거 검증을 직접 수행하지 않습니다. `verify_status` 는 인자로 받고 기본값은 `not_verified` 입니다. `failed` 를 기본으로 두면 "검증해서 떨어진 것" 과 구분이 안 되고, `passed` 를 기본으로 두면 미검증 보고서가 backend 조회로 샙니다. 검증기는 실험 레포에 있고 연결은 후속 PR 입니다.

`quote` · `match` · `is_market_recap` · `not_found` 는 화면에 안 나가지만 DB 에는 전부 넣었습니다. 사후 환각 추적 수단이 이것뿐이고, 노출 제어는 backend 응답 단계에서 합니다.

---

`cd ai && uv run pytest` 31 passed / `uv run ruff check .` 통과.

마이그레이션이 모델과 어긋나지 않았는지는 `tests/test_migrations.py` 가 봅니다. `alembic upgrade head --sql` 로 DDL 을 받아 모델을 직접 컴파일한 DDL 과 칼럼 단위로 맞추는 방식이라 **DB 없이 돕니다.** 다만 이미 돌고 있는 DB 의 실제 스키마와 모델의 차이는 못 잡습니다 — 그건 DB 를 띄우고 `uv run alembic check` 을 돌려야 하고, CI 에 넣을 자리입니다.



## 리뷰와 대화

### PR 대화 댓글: @kmgvv23

시각: 2026-09-21T02:05:58Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/26#issuecomment-5754473089

지원님께서 BE에 저희 종목 분석 관련 구조만 올려둘 때, reports 라는 표현을 쓰셔서 증권사 애널리스트 리포트 관련은 analyst_report로,
저희가 생성하는 종목분석은 브리핑으로 보고 breif 관련으로 정리하자고 제안드렸는데, report라서 학영님께서 잡은 구조 그대로 가도 큰 상관없을지, 아님 크게 어려운게 아니라면 breif 관련으로 바꾸는게 나을지 의견 여쭙고자 합니다.

세부 코드는 아직 확인 못하여 추후에 확인 후 리뷰하겠습니다.

### PR 대화 댓글: @hakyeoung

시각: 2026-09-21T05:21:32Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/26#issuecomment-5755778631

저도 이름이 겹치는건 생각하지 못했습니다. 좋은 피드백 감사합니다! 민규님은 brief 형태로 바꾸는 걸 제시해 주셨는데,
추후에 간단 요약에 대한 수요가 생겼을 때, brief를 이미 선점하고 있으면 지금과 같은 문제가 다시 발생할 수 있을 것이라 생각됩니다.
그래서 저는 analysis 형태로 바꿀까 하는데 이 부분에 대해 어떻게 생각하시는지 의견 남겨주시면 감사하겠습니다!!

### PR 대화 댓글: @kmgvv23

시각: 2026-09-22T04:16:35Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/26#issuecomment-5771128225

brief가 또 그렇게 겹칠건 생각 못했네요 ㅎㅎ 학영님께서 정해주시면 그거에 맞춰서 진행하겠습니다! 그대로 가도 될 것 같아요

### PR 대화 댓글: @hakyeoung

시각: 2026-09-22T11:04:02Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/26#issuecomment-5775326539

@kmgvv23 말씀해주신 부분 수정 완료했는데 확인 부탁드립니다!

### PR 대화 댓글: @swimny

시각: 2026-09-22T11:39:59Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/26#issuecomment-5775783906

현재 뉴스 발췌해온 부분에 대한 db가 없는데, 확인해보니 이게 이미 백엔드에 구현이 되어 있었습니다. 그런데 이게 지금 백엔드에 있어야 하는 게 아니라 실질적으로 AI 파트에서 관리를 해야 할 것 같아서, 이걸 어떻게 할지 얘기를 해봐야겠다는 생각이 들었습니다.

### 리뷰: @swimny / APPROVED

시각: 2026-09-22T12:40:15Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/26#pullrequestreview-5278148400

말씀 드린 부분에 대해서는 나중에 협의를 봐야 할 부분인 것 같고, 현재 작성된 부분에 대해서는 이대로 진행해도 괜찮을 것 같습니다.

### 리뷰: @kmgvv23 / COMMENTED

시각: 2026-09-22T16:34:45Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/26#pullrequestreview-5280895055

내용 파악중이어서 다시 리뷰 드리겠습니다.

### PR 대화 댓글: @hakyeoung

시각: 2026-09-22T23:16:39Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/26#issuecomment-5785894317

일단 병합 가능한 상태로 변경 완료 했습니다.
민규님이 먼저 작성해주신 부분 따라가는 형태로 수정했는데 다른 이슈가 있다면 말씀해주시면 감사하겠습니다!

## 변경 파일

- `ai/README.md` (+45 / -20)
- `ai/alembic/env.py` (+9 / -1)
- `ai/alembic/versions/0019_stock_move_analysis.py` (+160 / -0)
- `ai/app/llm/prompts/stock_analysis_system.md` (+444 / -0)
- `ai/app/models/__init__.py` (+13 / -1)
- `ai/app/models/stock_move_analysis.py` (+253 / -0)
- `ai/app/repositories/stock_move_analysis.py` (+626 / -0)
- `ai/tests/fixtures/reports/SK이노베이션-2026-09-18-with-indirect__B__r1.json` (+41 / -0)
- `ai/tests/fixtures/reports/SK이노베이션-2026-09-18-with-indirect__B__r2.json` (+41 / -0)
- `ai/tests/fixtures/reports/SK하이닉스-2026-09-18-with-indirect__B__r1.json` (+41 / -0)
- `ai/tests/fixtures/reports/삼성전자-2026-09-18-with-indirect__B__r1.json` (+41 / -0)
- `ai/tests/test_migration_schema_sync.py` (+176 / -0)
- `ai/tests/test_migrations.py` (+2 / -1)
- `ai/tests/test_stock_move_analysis_model.py` (+120 / -0)
- `ai/tests/test_stock_move_analysis_parse.py` (+186 / -0)
