# #12 [BE] feature: AI 리포트 저장용 DB 스키마 초안 추가 (channel/source_card/report/report_citation)

원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/12

작성자: @jiw0nahn
작성 시각: 2026-09-15T13:24:33Z
보관 시각 (UTC): 2026-10-06T15:19:25.015347+00:00

## PR 본문

## 이번에 한 일

AI가 생성하는 리포트를 저장할 DB 스키마 초안.
AI 쪽 산출물 형태가 아직 확정 전이라, 나중에 수정하기 쉽게 자유 문자열/JSONB로 열어둔 곳이 많습니다.

각 테이블
- **channel**: 텔레그램 채널 메타데이터. 신뢰도 등급(A/B/C/D)을 갖고 있고, 사람이 검수하기 전까지는 NULL
- **source_card**: 텔레그램/뉴스/공시 등 AI가 근거로 쓰는 원문 카드를 담는 단일 테이블. 타입은 `card_type`으로 구분하고, 타입별로 달라지는 필드는 `payload`(JSONB)에 유연하게 저장
- **report**: 리포트 한 건
- **report_block**: 리포트 본문을 "이유(FACTOR)/참고(NOTE)/확인 못한 것(UNCONFIRMED)" 블록 단위로 쪼갠 것
- **report_citation**: 어느 블록이 어느 source_card를 인용했는지 연결하는 근거 추적용 테이블

## 리뷰에서 봐주셨으면 하는 곳

- `source_card` 하나에 텔레그램/뉴스/공시를 전부 같이 저장하는 게 맞는 방향인지. 종류마다 테이블을 따로 만드는 대신, 종류를 구분하는 값(`card_type`)만 다르게 주고 한 테이블에 몰아넣었습니다. 나중에 종류가 많이 늘어나도 이 방식이 괜찮을지 봐주시면 좋겠습니다.
- 지금 설계는 학영님이 올려준 예시 리포트 하나만 보고 만든 거라, AI쪽 개발 방향에 따라 다시 수정해나갈 예정이라, 수정해야할 사항이 있으면 알려주셨으면 좋겠습니다.

## 고민했던 점 · 다른 선택지

<!-- 두 방법 사이에서 고른 게 있다면 왜 이걸 골랐는지. 없으면 지워도 됩니다. -->
- 리포트 본문을 처음엔 그냥 JSON 하나에 통째로 저장하려고 했는데, 실제 리포트 예시를 보고 "이유별로 하나씩 행을 나누는 게" 나중에 다루기(검색, 통계 등) 편할 것 같아서 지금처럼 `report_block`을 따로 만들었습니다.
- 데이터베이스 스키마 관리를 위해 Alembic을 도입해야 할지..
## 지난 리뷰 반영 (재리뷰 요청일 때만)

<!--
2차 리뷰 요청 시 채웁니다. 어떤 피드백을 어디서 반영했는지 적어주세요.
- [피드백 요약] → `refactor/...` 브랜치, 파일/함수
반영하지 않기로 한 게 있으면 그 이유도 함께 적어주세요. 그것도 좋은 리뷰 소재입니다.
-->

---

<!--
🚨 이 PR 이 멘토 리뷰용(`develop` → `main`)이라면, 받는 쪽(base)이 `main` 인지 확인해 주세요.
   `develop` 이 기본 브랜치라 GitHub 이 base 를 `develop` 으로 미리 채워 둡니다.
   `main` 이 아니면 멘토가 자동 지정되지 않고 리뷰가 시작되지 않습니다.
-->


## 리뷰와 대화

### 코드 리뷰 댓글: @kmgvv23

시각: 2026-09-16T02:18:49Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/12#discussion_r4021919983
파일: `backend/app/models/report.py` / 줄: 1
커밋: `ec3273ef85f852db61c3844d1cc6dad98b86a090`

애널리스트 리포트를 가져오는 것과 종목 분석 리포트 관련 코드에서 모두 report라는 키워드를 써서 헷갈릴 수도 있을 것 같습니다.
애널리스트 리포트 관련은 제가 analyst_report로 표기하고, 종목 분석 관련해서는 "브리핑", brief 같은 파일명이나 키워드를 쓰는게 어떤지 제안드립니다.

### 리뷰: @kmgvv23 / COMMENTED

시각: 2026-09-16T02:18:49Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/12#pullrequestreview-5217799846

(본문 없음)

## 변경 파일

- `backend/app/models/__init__.py` (+4 / -1)
- `backend/app/models/channel.py` (+37 / -0)
- `backend/app/models/report.py` (+83 / -0)
- `backend/app/models/source_card.py` (+59 / -0)
- `backend/pyproject.toml` (+1 / -0)
- `backend/uv.lock` (+91 / -0)
