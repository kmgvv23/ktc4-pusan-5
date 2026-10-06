# #18 [AI] feature: 네이버 증권 리서치 수집기

원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/18

작성자: @kmgvv23
작성 시각: 2026-09-16T08:09:46Z
보관 시각 (UTC): 2026-10-06T15:19:25.015347+00:00

## PR 본문

## 이번에 한 일

네이버 증권 리서치에서 리포트를 받아 `analyst_reports` 에 넣는 수집기입니다. #16 에서 만든 테이블을 씁니다.

- `services/analyst/schema.py` 소스 공통 `AnalystReportItem`, `PdfText` 모델
- `services/analyst/pdf.py` PDF → 텍스트
- `services/analyst/naver.py` 목록, 상세, PDF 받기. 산업 리포트 업종의견 추출
- `repositories/analyst_report.py` `upsert_analyst_reports`, `known_ids`
- `collectors/analyst_report.py` 수집 배치
- `alembic/` 테이블 생성과 `source_category`·자연키 변경 이력
- `tests/` 파싱, 증분 수집, AI 자료 제외, PDF 추출과 폴백, PostgreSQL 마이그레이션 검증. 총 33개

실행 방법입니다.

```bash
cd ai
uv sync
brew install poppler          # pdftotext. Ubuntu 는 apt install poppler-utils
uv run alembic upgrade head   # 빈 DB 기준. 기존 DB는 아래 전환 안내 참고
uv run python -m app.collectors.analyst_report --days 7
```

옵션입니다. `--limit 5` 로 하면 금방 확인됩니다.

```
--days 7              오늘로부터 며칠 전까지
--category company    특정 카테고리만. 여러 번 줄 수 있습니다
--limit 5             카테고리당 최대 건수
```

실제로 돌린 결과입니다.

```
 source_id | category |   broker   |    item_name     | opinion | goal_price | upside_pct | body_chars
-----------+----------+------------+------------------+---------+------------+------------+-----------
 96183     | company  | DS투자증권 | 셀트리온         | 매수    |     320000 |      80.28 |      26662
 96182     | company  | 교보증권   | 디케이티         | 없음    |            |            |       9599
 96181     | company  | 교보증권   | 펌텍코리아       | 없음    |            |            |       6305
```

`pdftotext` 는 파이썬 패키지가 아니라 시스템 프로그램이라 `uv sync` 로 안 깔립니다. 위 `brew install poppler` 를 꼭 해주세요. 없어도 pypdf 로 넘어가서 돌아가긴 하는데 품질이 떨어집니다. 나중에 이걸 서버에서 돌리게 되면 그 이미지에도 poppler-utils 를 넣어야 합니다.

## 리뷰에서 봐주셨으면 하는 곳

- 클라이언트를 함수가 아니라 클래스로 뒀습니다. 일주일치를 받으면 HTTP 요청이 1,000번 넘게 나갑니다. 리포트 한 건마다 상세 1번, PDF 1번이라 500건이면 그렇게 됩니다. 요청마다 접속을 새로 열면 서버와 연결하는 절차를 1,000번 반복하게 됩니다. 클래스로 두면 접속 하나를 계속 쓸 수 있습니다. `services/news/naver.py` 는 요청이 몇 번 안 되니 함수로 충분한데 여기는 양이 달라서 다르게 갔습니다.
- PDF 원본은 저장하지 않습니다. 임시 폴더에 받아 텍스트만 뽑고 파일은 지웁니다. `attach_url` 이 계속 살아 있어서 파서를 고치면 다시 받을 수 있습니다.
- 이미지로만 된 PDF 가 1% 정도 있습니다. 글자가 아예 안 뽑히는데 OCR 까지 붙일 양은 아니라고 봐서 본문은 버립니다. 실패와 구분하려고 `body_status='empty'` 로 표시만 해둡니다.

## 고민했던 점 · 다른 선택지

### PDF 추출 옵션은 `-raw` 로

pdftotext 에 `-layout` 과 `-raw` 가 있습니다. `-layout` 은 화면 위치를 공백으로 재현하느라 사이드바가 본문 문장 중간에 끼어듭니다. 실제로 `영업이익 1,514억` 다음에 사이드바 두 줄이 들어가고 그 뒤에 `원(+10% YoY)` 이 붙었습니다.

리포트 46건을 둘 다 돌려 비교했고 `-raw` 가 전부 낫거나 같았습니다.

### 추출 성공 판정은 종료코드가 아니라 글자수로

poppler 26.07 은 `Catalog dictionary does not contain a valid "Pages" entry` 로 아무것도 못 읽고도 종료코드 0 을 줍니다. 받아온 PDF 55건 중 21건이 그랬고, 그중 203쪽짜리가 본문 0자로 들어갔습니다.

pypdf 는 같은 파일을 정상으로 읽어서 폴백으로 뒀습니다.

### 한국IR협의회가 AI 로 만든 자료는 제외

제목이 `[AI] ` 로 시작하는 것들인데 494건 중 20건입니다. 투자의견과 목표주가가 전부 비어 있고 제목이 종목과 맞지 않습니다. 아이디스홀딩스(054800)와 씨에스베어링(297090)에 같은 제목이 붙어 있습니다. 네이버 API 원본이 그렇습니다.

### 산업 리포트에는 종목별 목표주가를 안 넣음

리포트 하나에 종목이 여러 개라 칼럼에 안 들어갑니다. 대신 업종의견과 Top picks 를 본문 앞 절반에서만 뽑습니다.

뒤쪽에는 모든 리포트에 붙는 컴플라이언스 부록이 있어서 전체를 뒤지면 과거 의견이 걸립니다. 한 리포트에서 `목표주가` 가 39번 나오는데 36번이 부록이었습니다.

### 발간일 칼럼은 이번에 안 넣음

`write_date` 에는 네이버가 주는 `writeDate` 가 들어가는데, 이건 발간일이 아니라 네이버에 올라온 날입니다. 앞으로 텔레그램도 같은 테이블에 넣을 예정인데 거기는 파일명에 증권사가 날짜를 박아줘서 그게 발간일입니다. 출처마다 기준이 달라집니다.

네이버 465건을 PDF 표지와 대조해보니 69% 는 같았고, 다른 것도 대부분 하루 아니면 사흘 차이였습니다. 사흘짜리는 금요일에 낸 주간 리포트를 월요일에 올리는 경우입니다.

제대로 맞추려면 제목과 표지에서 날짜를 뽑는 규칙을 따로 만들고 검증해야 합니다. 이번 PR 에는 안 넣었고, 발간일 칼럼을 따로 뺄지는 나중에 정하겠습니다.



---

## 리뷰 반영 (2026-09-18)

#19 에서 멘토님이 지적하신 자연키 충돌을 여기서 같이 고쳤습니다. 고치는 데 필요한 파일 대부분이 이 PR 에서 처음 생기는 거라 따로 뺄 수가 없었습니다. 그래서 이 PR 범위가 제목보다 조금 넓습니다.

`researchId` 가 네이버 카테고리 안에서만 유일한데 자연키에 `category` 를 쓰고 있었습니다. invest 와 daily 를 market 으로 합치는 순간 서로 다른 리포트가 같은 키가 됩니다. 실제로 확인해보니 invest 2026-01-16 자 37550 과 daily 2026-09-17 자 37550 이 둘 다 살아 있고, 같은 번호대에서 42건이 겹쳤습니다. 합쳐서 넣었으면 한쪽이 다른 쪽을 덮어썼을 텐데 에러는 나지 않습니다.

`source_category` 를 따로 두고 자연키를 그걸로 바꿨습니다. 출처가 준 분류는 그대로 남기고, `category` 는 우리 분류로 씁니다.

```
source_category   naver     company | industry | economy | invest | daily
                  telegram  채널명
category          company | industry | economy | market
자연키            (source, source_category, source_id)
```

같이 고친 것이 하나 더 있습니다. 상세 조회 URL 에 `category` 를 쓰고 있었는데 그대로 두면 `/api/research/market/37550` 이 되어 404 입니다. 원본 구분으로 바꿨고, 네이버 5종을 목록에서 상세까지 실제로 호출해서 확인했습니다.

`fetch_pdf_text` 와 테스트 함수명 지적도 반영했습니다.

- `a32d0f7` PDF 바이트 판정을 `pdf.py` 로 옮김. 텔레그램 수집기에서 같은 코드를 베끼고 있었습니다
- `a9fea3c` 테스트 함수명 16개를 영어로. backend/tests 와 맞췄습니다
- `7e3f0f3` 자연키 수정

## Alembic 전환

수집기의 `create_tables()` 호출을 제거했습니다. 이제 수집 전에 `ai/`에서
`uv run alembic upgrade head`로 테이블을 준비합니다.

#26의 `0001` 초기 테이블 정의를 재사용하고, `0018_source_category`에서 원본 분류 칼럼과
자연키를 변경합니다. 초기 리비전은 현재 모델의 `create_all()`을 호출하지 않고 당시 구조를
명시합니다. AI 변경 이력은 `alembic_version_ai`에 저장하고, 자동 비교도 AI 소유 테이블로 제한합니다.

기존 데이터는 유지합니다. company/industry/economy 등 원본 분류가 남아 있으면 그대로 사용하고,
market 행은 저장된 네이버 원본 URL과 리포트 ID를 대조해 invest/daily를 복구합니다.
복구할 수 없는 행이 있으면 변경 전체를 롤백합니다. 구 자연키로 되돌릴 때 충돌이 있어도 중단합니다.

### 기존 DB를 쓰는 경우

테이블을 삭제하지 마세요. 실제 칼럼·타입·인덱스·제약을 확인한 뒤 구조에 맞게 전환해야 합니다.
`stamp`는 스키마를 수정하거나 일치 여부를 검사하는 명령이 아닙니다.

- `0001`과 정확히 같은 DB: `uv run alembic stamp 0001` 후 `uv run alembic upgrade head`
- 이미 #18 모델과 정확히 같은 DB: 확인 후 `uv run alembic stamp 0018_source_category`, 이어서 `uv run alembic check`
- 일부만 반영됐거나 다른 구조: stamp하지 않고 차이를 먼저 확인

세부 절차는 `ai/README.md`에 적었습니다.

### 다른 PR과 통합할 때

#26의 `0002`와 이번 리비전은 모두 `0001`에서 출발합니다. 두 파일이 모두 있는 통합 브랜치에서
merge revision을 추가하고, AI 버전 테이블과 테이블 소유 목록을 맞춰야 합니다.
#26의 보고서 모델·테이블은 이번 변경에 포함하지 않았습니다.
#23·#24의 백엔드 baseline 및 설정 통합은 해당 PR에서 별도 조정이 필요합니다.

### 검증

- 별도 PostgreSQL 17에서 기존 테스트 포함 **33 passed**, Ruff 통과
- 빈 DB 설치, 기존 데이터 이전, 복구 실패 시 롤백, 자연키 변경과 downgrade 충돌 확인
- 같은 DB의 backend 테이블과 변경 이력 보존, `alembic check` 통과
- #26과 설정·모델을 합치고 merge revision을 추가한 임시 환경에서도 빈 DB·#18 적용 DB·#26 적용 DB에서 통합 head까지 적용 및 스키마 일치 확인


## 리뷰와 대화

### 리뷰: @hakyeoung / COMMENTED

시각: 2026-09-17T05:06:28Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/18#pullrequestreview-5231307716

(본문 없음)

### 코드 리뷰 댓글: @hakyeoung

시각: 2026-09-17T05:06:29Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/18#discussion_r4033307299
파일: `ai/app/services/analyst/naver.py` / 줄: 252
커밋: `099247a9f47e263c39ee0eeaabb7a948d0be4556`

````diff
@@ -0,0 +1,354 @@
+"""네이버 증권 리서치 어댑터.
+
+    목록  GET m.stock.naver.com/api/research/{category}?page=1&pageSize=100
+    상세  GET m.stock.naver.com/api/research/{category}/{researchId}
+    원문  GET stock.pstatic.net/stock-research/.../*.pdf
+
+인증이 없다. 확인한 것(2026-09-14 실측):
+    - 카테고리 5종. `daily` 와 `market` 은 researchId 까지 같은 동일 데이터라 daily 만 받는다.
+    - 상세 채움률(n=20): attachUrl·content·opinion 20/20, goalPrice 14/20.
+    - 요약(content)은 HTML 이고 태그를 걷으면 평균 517자.
+    - pageSize=100 까지 먹고 page=3000 이 2016년까지 간다. 백필 깊이는 제한이 아니다.
+    - 종목코드·투자의견·목표주가는 `company` 에만 있다. 나머지는 제목과 PDF 뿐이다.
+
+네이버가 싣는 증권사는 21곳뿐이다. 삼성·NH 는 0건, 한국투자는 2건이었다(3주 실측).
+대형사 리포트는 여기 안 들어온다 — 다른 경로로 따로 받아야 한다.
+
+## 왜 여기만 클래스인가
+
+backend 의 `services/news/naver.py` 나 `dart_client.py` 는 모듈 함수이고 httpx 를
+호출마다 새로 연다. 호출이 한 번이라 그래도 된다. 여기는 다르다 —
+카테고리 5개 × 목록 페이지 + **건당 상세 1회 + PDF 1회**라, 일주일치 500건이면
+HTTP 호출이 1,000회를 넘는다. 매번 AsyncClient 를 새로 열면 TCP 연결과 TLS
+핸드셰이크를 그만큼 반복한다. 커넥션을 재사용하려고 클래스로 뒀다.
+"""
+
+import asyncio
+import hashlib
+import html
+import logging
+import re
+import tempfile
+from datetime import UTC, date, datetime
+from pathlib import Path
+from typing import Any, Self
+
+import httpx
+
+from app.core.config import settings
+from app.services.analyst.pdf import PDF_MAX_BYTES, extract_pdf_text
+from app.services.analyst.schema import AnalystReportItem, PdfText
+
+logger = logging.getLogger(__name__)
+
+BASE_URL = "https://m.stock.naver.com/api/research"
+CATEGORIES = ("company", "industry", "economy", "invest", "daily")
+
+# 네이버 웹에서 오는 요청처럼 보이게 한다. 없으면 차단되는 경로가 있다.
+HEADERS = {
+    "User-Agent": (
+        "Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15 "
+        "(KHTML, like Gecko) Version/17.0 Mobile/15E148 Safari/604.1"
+    ),
+    "Referer": "https://m.stock.naver.com/research",
+}
+
+
+class NaverResearchError(Exception):
+    """네이버 리서치 호출 실패. 원인 예외는 __cause__ 에 남는다."""
+
+
+# ---------------------------------------------------------------- 파싱 유틸
+
+_TAG = re.compile(r"<[^>]+>")
+_WS = re.compile(r"[ \t ]+")
+
+
+def strip_html(raw: str | None) -> str | None:
+    """네이버 요약은 <p><br><strong> 이 섞인 HTML 이다. 줄바꿈은 살리고 태그만 걷는다."""
+    if not raw:
+        return None
+    text = re.sub(r"<br\s*/?>", "\n", raw, flags=re.IGNORECASE)
+    text = re.sub(r"</p\s*>", "\n\n", text, flags=re.IGNORECASE)
+    text = _TAG.sub("", text)
+    text = html.unescape(text)
+    text = _WS.sub(" ", text)
+    text = re.sub(r"\n{3,}", "\n\n", text)
+    return text.strip() or None
+
+
+def to_int(raw: Any) -> int | None:
+    """네이버는 숫자를 문자열로 준다. 빈 값·콤마·'-' 가 섞여 온다."""
+    if raw is None:
+        return None
+    s = str(raw).strip().replace(",", "")
+    if s in ("", "-", "0"):
+        return None if s in ("", "-") else 0
+    try:
+        return int(float(s))
+    except ValueError:
+        return None
+
+
+# 한국IR협의회가 AI 로 생성해 올리는 기업소개 자료. 제목이 "[AI] " 로 시작한다.
+# 받지 않는다 — 애널리스트 리포트가 아니고, 실제로 데이터가 틀려 있다.
+#   아이디스홀딩스(054800) 제목 = "[AI] 풍력 터빈용 피치·요 베어링 제조기업"
+#   씨에스베어링(297090)   제목 = "[AI] 풍력 터빈용 피치·요 베어링 제조기업"  ← 같은 제목
+#   에르코스(435570)       제목 = "[AI] 영상보안·모니터·프린터 지주회사"     ← 앞 종목 것
+# 요약 본문은 종목과 맞는데 제목만 한 칸씩 밀려 있다. 네이버 API 원본이 그렇다(직접 확인).
+# 투자의견·목표주가도 전부 비어 있어서 우리가 쓸 값이 없다.
+_AI_TITLE = re.compile(r"^\s*\[\s*AI\s*\]")
+
+
+def is_ai_generated(item: AnalystReportItem) -> bool:
+    return bool(_AI_TITLE.match(item.title or ""))
+
+
+# ------------------------------------------------ 산업 리포트: 업종의견 + Top picks
+#
+# 산업 리포트는 종목이 아니라 업종에 의견을 낸다. 네이버가 이 값을 안 주므로 PDF 에서 뽑는다.
+# **본문 앞 절반만 본다** — 뒤쪽에는 모든 리포트에 붙는 컴플라이언스 부록("투자의견 비율",
+# "투자의견 변동 내역 및 목표주가 괴리율")이 있고 거기엔 과거 의견이 잔뜩 적혀 있다.
+# 실측: 한 리포트에서 "목표주가" 39회 중 36회가 부록이었다.
+_SECTOR_OP = re.compile(
+    r"(Overweight|OVERWEIGHT|비중\s?확대|Neutral|NEUTRAL|중\s?립"
+    r"|Underweight|UNDERWEIGHT|비중\s?축소|Positive|긍정적|Negative|부정적)"
+)
+_TOPPICK = re.compile(
+    r"(?:Top[\s\-]?picks?|탑\s?픽|최선호주|최선호\s?종목)\s*[:：]?\s*([^\n]{2,80})", re.IGNORECASE
+)
+_PICK_SPLIT = re.compile(r"[,·/]| 및 |과 |와 ")
+# Top pick 자리에 딸려 들어오는 말들. 종목명이 아니다.
+_NOT_A_PICK = {
+    "유지", "및", "관심종목", "차선호주", "제시", "상향", "하향", "기타",
+    "자동차", "2차전지", "top", "picks", "pick",
+}
+
+
+def _norm_sector(s: str) -> str:
+    s = s.lower().replace(" ", "")
+    if s in ("overweight", "비중확대", "positive", "긍정적"):
+        return "비중확대"
+    if s in ("neutral", "중립"):
+        return "중립"
+    if s in ("underweight", "비중축소", "negative", "부정적"):
+        return "비중축소"
+    return s
+
+
+def extract_sector_view(body: str | None) -> tuple[str | None, list[str]]:
+    """(업종의견, Top picks). 못 찾으면 (None, [])."""
+    if not body:
+        return None, []
+    head = body[: min(len(body) // 2, 8000)]
+
+    m = _SECTOR_OP.search(head)
+    opinion = _norm_sector(m.group(1)) if m else None
+
+    picks: list[str] = []
+    m = _TOPPICK.search(head)
+    if m:
+        # 차선호주·관심종목은 Top pick 이 아니다. 거기서 끊는다.
+        raw = re.split(r"\s{2,}|차선호|관심종목|제시|이다|입니다", m.group(1))[0]
+        for tok in _PICK_SPLIT.split(raw):
+            t = tok.strip(" ‘’'\"()[]·:")
+            if (
+                2 <= len(t) <= 14
+                and re.search(r"[가-힣A-Za-z]", t)
+                and t.lower() not in _NOT_A_PICK
+                and not t.isdigit()
+                and not re.search(r"[\]\[]", t)
+            ):
+                picks.append(t)
+    return opinion, picks[:5]
+
+
+def parse_list_row(category: str, row: dict[str, Any]) -> AnalystReportItem:
+    return AnalystReportItem(
+        source_id=str(row["researchId"]),
+        category=category,
+        title=(row.get("title") or "").strip(),
+        broker=(row.get("brokerName") or "").strip(),
+        write_date=date.fromisoformat(row["writeDate"]),
+        item_code=row.get("itemCode") or None,
+        item_name=row.get("itemName") or None,
+        read_count=to_int(row.get("readCount")),
+        end_url=row.get("endUrl"),
+        raw=row,
+    )
+
+
+def merge_detail(item: AnalystReportItem, detail: dict[str, Any]) -> AnalystReportItem:
+    """상세 응답을 목록 행에 덮어쓴다. 상세에만 있는 게 요약·목표주가·PDF 링크다."""
+    content = detail.get("content")
+    item.attach_url = detail.get("attachUrl") or None
+    item.summary_html = content or None
+    item.summary_text = strip_html(content)
+    item.opinion = (detail.get("opinion") or "").strip() or None
+    item.goal_price = to_int(detail.get("goalPrice"))
+    # detail["prevGoalPrice"] 는 직전 목표주가가 아니라 현재가의 중복이다(모델 주석 참고).
+    item.price_at_write = to_int(detail.get("priceAtWriteDate"))
+    item.item_code = item.item_code or detail.get("itemCode") or None
+    item.item_name = item.item_name or detail.get("itemName") or None
+    item.raw = {"list": item.raw, "detail": detail}
+    return item
+
+
+def utcnow() -> datetime:
+    return datetime.now(UTC)
+
+
+# ---------------------------------------------------------------- 클라이언트
+
+
+class NaverResearchClient:
+    def __init__(self, client: httpx.AsyncClient | None = None) -> None:
+        self._own = client is None
+        self._client = client or httpx.AsyncClient(
+            timeout=settings.http_timeout_sec, headers=HEADERS, follow_redirects=True
+        )
+
+    async def aclose(self) -> None:
+        if self._own:
+            await self._client.aclose()
+
+    async def __aenter__(self) -> Self:
+        return self
+
+    async def __aexit__(self, *exc: object) -> None:
+        await self.aclose()
+
+    async def fetch_list(
+        self, category: str, page: int = 1, page_size: int = 100
+    ) -> tuple[list[dict[str, Any]], dict[str, Any], int]:
+        """(행들, 호출 파라미터, HTTP 상태)."""
+        params = {"page": page, "pageSize": page_size}
+        try:
+            r = await self._client.get(f"{BASE_URL}/{category}", params=params)
+            r.raise_for_status()
+        except httpx.HTTPError as exc:
+            raise NaverResearchError(f"목록 실패 {category} p{page}") from exc
+        rows = r.json()
+        return (rows if isinstance(rows, list) else []), params, r.status_code
+
+    async def fetch_detail(self, category: str, research_id: str) -> dict[str, Any]:
+        try:
+            r = await self._client.get(f"{BASE_URL}/{category}/{research_id}")
+            r.raise_for_status()
+        except httpx.HTTPError as exc:
+            raise NaverResearchError(f"상세 실패 {category}/{research_id}") from exc
+        return r.json().get("researchContent") or {}
+
+    async def fetch_pdf_text(self, url: str | None) -> PdfText:
````

이 메소드가 NaverResearchClient 안에 구현되어 있는 이유가 궁금합니다. pdf에서 텍스트를 뽑아내는건 네이버에서만이 아니라 텔레그램 쪽에서도 해야 할 수 있을텐데 이런 내용이 pdf.py에 있다면 재사용하는데 도움이 되지 않을까 생각이 들어서요. 심각한 오류는 아닌데 개인적으로 궁금해서 코멘트 남깁니다!

### 코드 리뷰 댓글: @stfifo

시각: 2026-09-17T05:14:40Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/18#discussion_r4033359756
파일: `ai/tests/test_analyst_naver.py` / 줄: 68
커밋: `720399e2eb62bd0382600eb3934d182614536602`

````diff
@@ -0,0 +1,189 @@
+"""네이버 리서치 응답 파싱. 실제 응답을 그대로 붙여 픽스처로 쓴다.
+
+네트워크를 타지 않는다 — 남의 서버 사정에 우리 테스트가 흔들리면 안 된다.
+"""
+
+from datetime import date
+
+import pytest
+
+from app.services.analyst.naver import (
+    NaverResearchClient,
+    extract_sector_view,
+    is_ai_generated,
+    merge_detail,
+    parse_list_row,
+    strip_html,
+    to_int,
+)
+
+# 2026-09-14 m.stock.naver.com/api/research/company?page=1 실제 응답 한 줄
+LIST_ROW = {
+    "researchCategory": "종목분석",
+    "category": "종목분석",
+    "itemCode": "271560",
+    "itemName": "오리온",
+    "researchId": 96141,
+    "title": "실적 안정성과 주주 환원으로 리레이팅 기대",
+    "brokerName": "DS투자증권",
+    "writeDate": "2026-09-14",
+    "readCount": "613",
+    "endUrl": "https://m.stock.naver.com/research/company/96141",
+}
+
+DETAIL = {
+    "itemCode": "271560",
+    "itemName": "오리온",
+    "researchId": 96141,
+    "title": "실적 안정성과 주주 환원으로 리레이팅 기대",
+    "brokerName": "DS투자증권",
+    "writeDate": "2026-09-14",
+    "readCount": "613",
+    "attachUrl": "https://stock.pstatic.net/stock-research/company/66/20260914_company_148994000.pdf",
+    "content": "<p><strong>26년 8월: 소비 둔화</strong></p><p><br>매출액 3,010억원(+9% YoY)</p>",
+    "opinion": "매수",
+    "goalPrice": "180000",
+    "prevGoalPrice": "119300",
+    "priceAtWriteDate": "119300",
+}
+
+# 산업분석에는 종목코드·투자의견·목표주가가 없다
+INDUSTRY_DETAIL = {
+    "researchId": "46078",
+    "title": "[매태호] 27년 국방예산으로 보는 K-방산 발전 방향성",
+    "brokerName": "DS투자증권",
+    "writeDate": "2026-09-14",
+    "readCount": "445",
+    "attachUrl": "https://stock.pstatic.net/stock-research/industry/199_20260914.pdf",
+    "content": "<p>방산 예산이 늘었다</p>",
+}
+
+
+def test_목록_한줄_파싱() -> None:
+    item = parse_list_row("company", LIST_ROW)
+    assert item.source_id == "96141"  # int 로 오지만 문자열로 통일한다
+    assert item.item_code == "271560"
+    assert item.write_date == date(2026, 9, 14)
+    assert item.read_count == 613
+
````

함수 이름을 한글로 작성한 이유 궁금합니다! 한글로 해도 정상적으로 작동하기는 하나 배포하거나 터미널 환경에서 인코딩 오류 발생할 수도 있어서 함수명은 영어로 작성하되 기능 설명은 주석으로 처리하는 게 더 좋을 것 같습니다

### 리뷰: @stfifo / COMMENTED

시각: 2026-09-17T05:14:40Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/18#pullrequestreview-5231378684

(본문 없음)

### 코드 리뷰 댓글: @kmgvv23

시각: 2026-09-17T16:17:37Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/18#discussion_r4039123907
파일: `ai/app/services/analyst/naver.py` / 줄: 252
커밋: `099247a9f47e263c39ee0eeaabb7a948d0be4556`

````diff
@@ -0,0 +1,354 @@
+"""네이버 증권 리서치 어댑터.
+
+    목록  GET m.stock.naver.com/api/research/{category}?page=1&pageSize=100
+    상세  GET m.stock.naver.com/api/research/{category}/{researchId}
+    원문  GET stock.pstatic.net/stock-research/.../*.pdf
+
+인증이 없다. 확인한 것(2026-09-14 실측):
+    - 카테고리 5종. `daily` 와 `market` 은 researchId 까지 같은 동일 데이터라 daily 만 받는다.
+    - 상세 채움률(n=20): attachUrl·content·opinion 20/20, goalPrice 14/20.
+    - 요약(content)은 HTML 이고 태그를 걷으면 평균 517자.
+    - pageSize=100 까지 먹고 page=3000 이 2016년까지 간다. 백필 깊이는 제한이 아니다.
+    - 종목코드·투자의견·목표주가는 `company` 에만 있다. 나머지는 제목과 PDF 뿐이다.
+
+네이버가 싣는 증권사는 21곳뿐이다. 삼성·NH 는 0건, 한국투자는 2건이었다(3주 실측).
+대형사 리포트는 여기 안 들어온다 — 다른 경로로 따로 받아야 한다.
+
+## 왜 여기만 클래스인가
+
+backend 의 `services/news/naver.py` 나 `dart_client.py` 는 모듈 함수이고 httpx 를
+호출마다 새로 연다. 호출이 한 번이라 그래도 된다. 여기는 다르다 —
+카테고리 5개 × 목록 페이지 + **건당 상세 1회 + PDF 1회**라, 일주일치 500건이면
+HTTP 호출이 1,000회를 넘는다. 매번 AsyncClient 를 새로 열면 TCP 연결과 TLS
+핸드셰이크를 그만큼 반복한다. 커넥션을 재사용하려고 클래스로 뒀다.
+"""
+
+import asyncio
+import hashlib
+import html
+import logging
+import re
+import tempfile
+from datetime import UTC, date, datetime
+from pathlib import Path
+from typing import Any, Self
+
+import httpx
+
+from app.core.config import settings
+from app.services.analyst.pdf import PDF_MAX_BYTES, extract_pdf_text
+from app.services.analyst.schema import AnalystReportItem, PdfText
+
+logger = logging.getLogger(__name__)
+
+BASE_URL = "https://m.stock.naver.com/api/research"
+CATEGORIES = ("company", "industry", "economy", "invest", "daily")
+
+# 네이버 웹에서 오는 요청처럼 보이게 한다. 없으면 차단되는 경로가 있다.
+HEADERS = {
+    "User-Agent": (
+        "Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15 "
+        "(KHTML, like Gecko) Version/17.0 Mobile/15E148 Safari/604.1"
+    ),
+    "Referer": "https://m.stock.naver.com/research",
+}
+
+
+class NaverResearchError(Exception):
+    """네이버 리서치 호출 실패. 원인 예외는 __cause__ 에 남는다."""
+
+
+# ---------------------------------------------------------------- 파싱 유틸
+
+_TAG = re.compile(r"<[^>]+>")
+_WS = re.compile(r"[ \t ]+")
+
+
+def strip_html(raw: str | None) -> str | None:
+    """네이버 요약은 <p><br><strong> 이 섞인 HTML 이다. 줄바꿈은 살리고 태그만 걷는다."""
+    if not raw:
+        return None
+    text = re.sub(r"<br\s*/?>", "\n", raw, flags=re.IGNORECASE)
+    text = re.sub(r"</p\s*>", "\n\n", text, flags=re.IGNORECASE)
+    text = _TAG.sub("", text)
+    text = html.unescape(text)
+    text = _WS.sub(" ", text)
+    text = re.sub(r"\n{3,}", "\n\n", text)
+    return text.strip() or None
+
+
+def to_int(raw: Any) -> int | None:
+    """네이버는 숫자를 문자열로 준다. 빈 값·콤마·'-' 가 섞여 온다."""
+    if raw is None:
+        return None
+    s = str(raw).strip().replace(",", "")
+    if s in ("", "-", "0"):
+        return None if s in ("", "-") else 0
+    try:
+        return int(float(s))
+    except ValueError:
+        return None
+
+
+# 한국IR협의회가 AI 로 생성해 올리는 기업소개 자료. 제목이 "[AI] " 로 시작한다.
+# 받지 않는다 — 애널리스트 리포트가 아니고, 실제로 데이터가 틀려 있다.
+#   아이디스홀딩스(054800) 제목 = "[AI] 풍력 터빈용 피치·요 베어링 제조기업"
+#   씨에스베어링(297090)   제목 = "[AI] 풍력 터빈용 피치·요 베어링 제조기업"  ← 같은 제목
+#   에르코스(435570)       제목 = "[AI] 영상보안·모니터·프린터 지주회사"     ← 앞 종목 것
+# 요약 본문은 종목과 맞는데 제목만 한 칸씩 밀려 있다. 네이버 API 원본이 그렇다(직접 확인).
+# 투자의견·목표주가도 전부 비어 있어서 우리가 쓸 값이 없다.
+_AI_TITLE = re.compile(r"^\s*\[\s*AI\s*\]")
+
+
+def is_ai_generated(item: AnalystReportItem) -> bool:
+    return bool(_AI_TITLE.match(item.title or ""))
+
+
+# ------------------------------------------------ 산업 리포트: 업종의견 + Top picks
+#
+# 산업 리포트는 종목이 아니라 업종에 의견을 낸다. 네이버가 이 값을 안 주므로 PDF 에서 뽑는다.
+# **본문 앞 절반만 본다** — 뒤쪽에는 모든 리포트에 붙는 컴플라이언스 부록("투자의견 비율",
+# "투자의견 변동 내역 및 목표주가 괴리율")이 있고 거기엔 과거 의견이 잔뜩 적혀 있다.
+# 실측: 한 리포트에서 "목표주가" 39회 중 36회가 부록이었다.
+_SECTOR_OP = re.compile(
+    r"(Overweight|OVERWEIGHT|비중\s?확대|Neutral|NEUTRAL|중\s?립"
+    r"|Underweight|UNDERWEIGHT|비중\s?축소|Positive|긍정적|Negative|부정적)"
+)
+_TOPPICK = re.compile(
+    r"(?:Top[\s\-]?picks?|탑\s?픽|최선호주|최선호\s?종목)\s*[:：]?\s*([^\n]{2,80})", re.IGNORECASE
+)
+_PICK_SPLIT = re.compile(r"[,·/]| 및 |과 |와 ")
+# Top pick 자리에 딸려 들어오는 말들. 종목명이 아니다.
+_NOT_A_PICK = {
+    "유지", "및", "관심종목", "차선호주", "제시", "상향", "하향", "기타",
+    "자동차", "2차전지", "top", "picks", "pick",
+}
+
+
+def _norm_sector(s: str) -> str:
+    s = s.lower().replace(" ", "")
+    if s in ("overweight", "비중확대", "positive", "긍정적"):
+        return "비중확대"
+    if s in ("neutral", "중립"):
+        return "중립"
+    if s in ("underweight", "비중축소", "negative", "부정적"):
+        return "비중축소"
+    return s
+
+
+def extract_sector_view(body: str | None) -> tuple[str | None, list[str]]:
+    """(업종의견, Top picks). 못 찾으면 (None, [])."""
+    if not body:
+        return None, []
+    head = body[: min(len(body) // 2, 8000)]
+
+    m = _SECTOR_OP.search(head)
+    opinion = _norm_sector(m.group(1)) if m else None
+
+    picks: list[str] = []
+    m = _TOPPICK.search(head)
+    if m:
+        # 차선호주·관심종목은 Top pick 이 아니다. 거기서 끊는다.
+        raw = re.split(r"\s{2,}|차선호|관심종목|제시|이다|입니다", m.group(1))[0]
+        for tok in _PICK_SPLIT.split(raw):
+            t = tok.strip(" ‘’'\"()[]·:")
+            if (
+                2 <= len(t) <= 14
+                and re.search(r"[가-힣A-Za-z]", t)
+                and t.lower() not in _NOT_A_PICK
+                and not t.isdigit()
+                and not re.search(r"[\]\[]", t)
+            ):
+                picks.append(t)
+    return opinion, picks[:5]
+
+
+def parse_list_row(category: str, row: dict[str, Any]) -> AnalystReportItem:
+    return AnalystReportItem(
+        source_id=str(row["researchId"]),
+        category=category,
+        title=(row.get("title") or "").strip(),
+        broker=(row.get("brokerName") or "").strip(),
+        write_date=date.fromisoformat(row["writeDate"]),
+        item_code=row.get("itemCode") or None,
+        item_name=row.get("itemName") or None,
+        read_count=to_int(row.get("readCount")),
+        end_url=row.get("endUrl"),
+        raw=row,
+    )
+
+
+def merge_detail(item: AnalystReportItem, detail: dict[str, Any]) -> AnalystReportItem:
+    """상세 응답을 목록 행에 덮어쓴다. 상세에만 있는 게 요약·목표주가·PDF 링크다."""
+    content = detail.get("content")
+    item.attach_url = detail.get("attachUrl") or None
+    item.summary_html = content or None
+    item.summary_text = strip_html(content)
+    item.opinion = (detail.get("opinion") or "").strip() or None
+    item.goal_price = to_int(detail.get("goalPrice"))
+    # detail["prevGoalPrice"] 는 직전 목표주가가 아니라 현재가의 중복이다(모델 주석 참고).
+    item.price_at_write = to_int(detail.get("priceAtWriteDate"))
+    item.item_code = item.item_code or detail.get("itemCode") or None
+    item.item_name = item.item_name or detail.get("itemName") or None
+    item.raw = {"list": item.raw, "detail": detail}
+    return item
+
+
+def utcnow() -> datetime:
+    return datetime.now(UTC)
+
+
+# ---------------------------------------------------------------- 클라이언트
+
+
+class NaverResearchClient:
+    def __init__(self, client: httpx.AsyncClient | None = None) -> None:
+        self._own = client is None
+        self._client = client or httpx.AsyncClient(
+            timeout=settings.http_timeout_sec, headers=HEADERS, follow_redirects=True
+        )
+
+    async def aclose(self) -> None:
+        if self._own:
+            await self._client.aclose()
+
+    async def __aenter__(self) -> Self:
+        return self
+
+    async def __aexit__(self, *exc: object) -> None:
+        await self.aclose()
+
+    async def fetch_list(
+        self, category: str, page: int = 1, page_size: int = 100
+    ) -> tuple[list[dict[str, Any]], dict[str, Any], int]:
+        """(행들, 호출 파라미터, HTTP 상태)."""
+        params = {"page": page, "pageSize": page_size}
+        try:
+            r = await self._client.get(f"{BASE_URL}/{category}", params=params)
+            r.raise_for_status()
+        except httpx.HTTPError as exc:
+            raise NaverResearchError(f"목록 실패 {category} p{page}") from exc
+        rows = r.json()
+        return (rows if isinstance(rows, list) else []), params, r.status_code
+
+    async def fetch_detail(self, category: str, research_id: str) -> dict[str, Any]:
+        try:
+            r = await self._client.get(f"{BASE_URL}/{category}/{research_id}")
+            r.raise_for_status()
+        except httpx.HTTPError as exc:
+            raise NaverResearchError(f"상세 실패 {category}/{research_id}") from exc
+        return r.json().get("researchContent") or {}
+
+    async def fetch_pdf_text(self, url: str | None) -> PdfText:
````

네이버는 attachUrl로 http 요청, 텔레그램은 Telethon으로 pdf를 내려받는 방식입니다.
즉, 내려받는 과정이 달라서 자연스럽게 내려받은 후 텍스트 추출도 각자 코드에서 하는 구조인데,
학영님께서 말씀하신대로 그 텍스트 추출 방식은 같아서 pdf.py에 놔두는게 좋을 것 같아 수정하였습니다.

### 리뷰: @kmgvv23 / COMMENTED

시각: 2026-09-17T16:17:37Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/18#pullrequestreview-5238526111

(본문 없음)

### 코드 리뷰 댓글: @kmgvv23

시각: 2026-09-17T16:25:38Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/18#discussion_r4039193781
파일: `ai/tests/test_analyst_naver.py` / 줄: 68
커밋: `720399e2eb62bd0382600eb3934d182614536602`

````diff
@@ -0,0 +1,189 @@
+"""네이버 리서치 응답 파싱. 실제 응답을 그대로 붙여 픽스처로 쓴다.
+
+네트워크를 타지 않는다 — 남의 서버 사정에 우리 테스트가 흔들리면 안 된다.
+"""
+
+from datetime import date
+
+import pytest
+
+from app.services.analyst.naver import (
+    NaverResearchClient,
+    extract_sector_view,
+    is_ai_generated,
+    merge_detail,
+    parse_list_row,
+    strip_html,
+    to_int,
+)
+
+# 2026-09-14 m.stock.naver.com/api/research/company?page=1 실제 응답 한 줄
+LIST_ROW = {
+    "researchCategory": "종목분석",
+    "category": "종목분석",
+    "itemCode": "271560",
+    "itemName": "오리온",
+    "researchId": 96141,
+    "title": "실적 안정성과 주주 환원으로 리레이팅 기대",
+    "brokerName": "DS투자증권",
+    "writeDate": "2026-09-14",
+    "readCount": "613",
+    "endUrl": "https://m.stock.naver.com/research/company/96141",
+}
+
+DETAIL = {
+    "itemCode": "271560",
+    "itemName": "오리온",
+    "researchId": 96141,
+    "title": "실적 안정성과 주주 환원으로 리레이팅 기대",
+    "brokerName": "DS투자증권",
+    "writeDate": "2026-09-14",
+    "readCount": "613",
+    "attachUrl": "https://stock.pstatic.net/stock-research/company/66/20260914_company_148994000.pdf",
+    "content": "<p><strong>26년 8월: 소비 둔화</strong></p><p><br>매출액 3,010억원(+9% YoY)</p>",
+    "opinion": "매수",
+    "goalPrice": "180000",
+    "prevGoalPrice": "119300",
+    "priceAtWriteDate": "119300",
+}
+
+# 산업분석에는 종목코드·투자의견·목표주가가 없다
+INDUSTRY_DETAIL = {
+    "researchId": "46078",
+    "title": "[매태호] 27년 국방예산으로 보는 K-방산 발전 방향성",
+    "brokerName": "DS투자증권",
+    "writeDate": "2026-09-14",
+    "readCount": "445",
+    "attachUrl": "https://stock.pstatic.net/stock-research/industry/199_20260914.pdf",
+    "content": "<p>방산 예산이 늘었다</p>",
+}
+
+
+def test_목록_한줄_파싱() -> None:
+    item = parse_list_row("company", LIST_ROW)
+    assert item.source_id == "96141"  # int 로 오지만 문자열로 통일한다
+    assert item.item_code == "271560"
+    assert item.write_date == date(2026, 9, 14)
+    assert item.read_count == 613
+
````

테스트 코드들이어서 스킵하고 확인을 제대로 안했었네요...
영문으로 수정해두었습니다.
앞으로 최대한 꼼꼼히 구조 파악하고 확인하겠습니다....!

### 리뷰: @kmgvv23 / COMMENTED

시각: 2026-09-17T16:25:39Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/18#pullrequestreview-5238610170

(본문 없음)

### 코드 리뷰 댓글: @swimny

시각: 2026-09-21T00:26:16Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/18#discussion_r4058581041
파일: `ai/app/collectors/analyst_report.py` / 줄: 25
커밋: `7e3f0f3106ea45c0c5086804b95fa4f734e33739`

````diff
@@ -0,0 +1,131 @@
+"""증권사 리포트 수집 → 정규화 → DB 저장.
+
+    uv run python -m app.collectors.analyst_report --days 7
+    uv run python -m app.collectors.analyst_report --days 1 --category company
+    uv run python -m app.collectors.analyst_report --days 1 --limit 5 --no-pdf   # 빠른 확인용
+
+도는 순서:
+    ① 카테고리별 목록을 since 이후까지 넘긴다 (호출 1~2회)
+    ② 이미 DB 에 있는 researchId 는 건너뛴다 — 증분의 핵심
+    ③ 남은 것만 상세 + PDF 본문을 받는다 (항목당 2회)
+    ④ upsert 한다. 카테고리 단위로 커밋해서 중간에 끊겨도 앞은 남는다
+
+목표주가·투자의견은 PDF 에서 뽑지 않는다. 네이버가 준 값만 쓴다.
+PDF 는 텍스트만 남기고 파일은 버린다(services/analyst/naver.py 참고).
+
+지금은 --days 로 직접 돌린다. 나중에 스케줄러가 하루 1~2회 돌리게 할 예정이다.
+리포트는 장 전후로 올라오지 실시간으로 쏟아지는 게 아니라 그 정도면 충분하다.
+"""
+
+import argparse
+import asyncio
+import logging
+from datetime import datetime, timedelta, timezone
+
+from app.core.database import SessionLocal, create_tables
````

학영님 PR에서 alembic을 도입하면서 create_tables()를 지우는데, 이 부분이랑 82번째 줄에서 그걸 쓰고 있습니다. 둘 다 머지되면 수집기가 실행 자체가 안 될 것 같습니다.
둘 다 develop에 아직 반영되지 않아서 서로 못 본 상황인 것 같습니다. 제가 alembic 쪽은 잘 몰라서 어떻게 고치는 게 맞는건지 잘 모르겠어서, 학영님이랑 같이 봐주시면 감사하겠습니다.

### 리뷰: @swimny / COMMENTED

시각: 2026-09-21T01:53:00Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/18#pullrequestreview-5262280606

(본문 없음)

### 코드 리뷰 댓글: @hakyeoung

시각: 2026-09-21T04:55:42Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/18#discussion_r4059353486
파일: `ai/app/collectors/analyst_report.py` / 줄: 25
커밋: `7e3f0f3106ea45c0c5086804b95fa4f734e33739`

````diff
@@ -0,0 +1,131 @@
+"""증권사 리포트 수집 → 정규화 → DB 저장.
+
+    uv run python -m app.collectors.analyst_report --days 7
+    uv run python -m app.collectors.analyst_report --days 1 --category company
+    uv run python -m app.collectors.analyst_report --days 1 --limit 5 --no-pdf   # 빠른 확인용
+
+도는 순서:
+    ① 카테고리별 목록을 since 이후까지 넘긴다 (호출 1~2회)
+    ② 이미 DB 에 있는 researchId 는 건너뛴다 — 증분의 핵심
+    ③ 남은 것만 상세 + PDF 본문을 받는다 (항목당 2회)
+    ④ upsert 한다. 카테고리 단위로 커밋해서 중간에 끊겨도 앞은 남는다
+
+목표주가·투자의견은 PDF 에서 뽑지 않는다. 네이버가 준 값만 쓴다.
+PDF 는 텍스트만 남기고 파일은 버린다(services/analyst/naver.py 참고).
+
+지금은 --days 로 직접 돌린다. 나중에 스케줄러가 하루 1~2회 돌리게 할 예정이다.
+리포트는 장 전후로 올라오지 실시간으로 쏟아지는 게 아니라 그 정도면 충분하다.
+"""
+
+import argparse
+import asyncio
+import logging
+from datetime import datetime, timedelta, timezone
+
+from app.core.database import SessionLocal, create_tables
````

회의 때 alembic 사용하기로 했으니까 alembic을 사용하는 형태로 민규님 코드가 수정되어야 할 것 같습니다..!

### 리뷰: @hakyeoung / COMMENTED

시각: 2026-09-21T04:55:42Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/18#pullrequestreview-5263157616

(본문 없음)

### 코드 리뷰 댓글: @kmgvv23

시각: 2026-09-22T04:15:19Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/18#discussion_r4068399646
파일: `ai/app/collectors/analyst_report.py` / 줄: 25
커밋: `7e3f0f3106ea45c0c5086804b95fa4f734e33739`

````diff
@@ -0,0 +1,131 @@
+"""증권사 리포트 수집 → 정규화 → DB 저장.
+
+    uv run python -m app.collectors.analyst_report --days 7
+    uv run python -m app.collectors.analyst_report --days 1 --category company
+    uv run python -m app.collectors.analyst_report --days 1 --limit 5 --no-pdf   # 빠른 확인용
+
+도는 순서:
+    ① 카테고리별 목록을 since 이후까지 넘긴다 (호출 1~2회)
+    ② 이미 DB 에 있는 researchId 는 건너뛴다 — 증분의 핵심
+    ③ 남은 것만 상세 + PDF 본문을 받는다 (항목당 2회)
+    ④ upsert 한다. 카테고리 단위로 커밋해서 중간에 끊겨도 앞은 남는다
+
+목표주가·투자의견은 PDF 에서 뽑지 않는다. 네이버가 준 값만 쓴다.
+PDF 는 텍스트만 남기고 파일은 버린다(services/analyst/naver.py 참고).
+
+지금은 --days 로 직접 돌린다. 나중에 스케줄러가 하루 1~2회 돌리게 할 예정이다.
+리포트는 장 전후로 올라오지 실시간으로 쏟아지는 게 아니라 그 정도면 충분하다.
+"""
+
+import argparse
+import asyncio
+import logging
+from datetime import datetime, timedelta, timezone
+
+from app.core.database import SessionLocal, create_tables
````

수정했습니다 확인 부탁드립니다!

### 리뷰: @kmgvv23 / COMMENTED

시각: 2026-09-22T04:15:19Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/18#pullrequestreview-5274081247

(본문 없음)

### 코드 리뷰 댓글: @hakyeoung

시각: 2026-09-22T10:12:46Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/18#discussion_r4070526095
파일: `ai/alembic/versions/0018_analyst_source_category.py` / 줄: 65
커밋: `099247a9f47e263c39ee0eeaabb7a948d0be4556`

````diff
@@ -0,0 +1,76 @@
+"""Preserve source category in the analyst report natural key.
+
+Revision ID: 0018_source_category
+Revises: 0001
+
+market으로 통합된 기존 행은 원본 URL에서 invest/daily를 복구한다.
+분류를 복구할 수 없으면 전체 변경을 롤백한다.
+"""
+
+import sqlalchemy as sa
+
+from alembic import op
+
+revision = "0018_source_category"
+down_revision = "0001"
+branch_labels = None
+depends_on = None
+
+
+def upgrade() -> None:
+    op.add_column("analyst_reports", sa.Column("source_category", sa.String(32), nullable=True))
+    op.execute("""
+        UPDATE analyst_reports
+        SET source_category = category
+        WHERE source = 'naver' AND category IN ('company', 'industry', 'economy', 'invest', 'daily')
+    """)
+    # 다른 리포트의 URL로 분류하지 않도록 source_id도 대조한다.
+    # SQLAlchemy가 정규식의 :api를 바인드 변수로 해석하지 않도록 콜론을 이스케이프한다.
+    op.execute(r"""
+        WITH original AS (
+            SELECT id, regexp_match(end_url,
+                '^https?://m[.]stock[.]naver[.]com/(?\:api/)?research/(invest|daily)/([0-9]+)(?:[?#].*)?$'
+            ) AS parts
+            FROM analyst_reports WHERE source = 'naver' AND category = 'market'
+        )
+        UPDATE analyst_reports AS report
+        SET source_category = original.parts[1]
+        FROM original
+        WHERE report.id = original.id AND original.parts[2] = report.source_id
+    """)
+    op.execute("""
+        DO $$ BEGIN
+            IF EXISTS (SELECT 1 FROM analyst_reports WHERE source_category IS NULL) THEN
+                RAISE EXCEPTION 'Cannot recover source_category. No rows were deleted. '
+                    'Inspect legacy source/category/end_url and restore verified original metadata '
+                    'before retrying; do not stamp past this migration.';
+            END IF;
+        END $$
+    """)
````

여기에서 에러를 raise하면
sqlalchemy.exc.InternalError: (psycopg.errors.RaiseException) Cannot recover...
이런 형식으로 나온다고 합니다.

그런데 이 부분을 파이썬으로도 구현이 가능하다고 하는데
```
if unresolved:
    samples = conn.execute(sa.text(
        "SELECT id, category, end_url FROM analyst_reports "
        "WHERE source_category IS NULL LIMIT 5"
    )).all()
    raise RuntimeError(f"복구 실패 {unresolved}건. 예: {samples}")
```
이런 식으로 하면 에러가 발생했을 때 조금 더 쉽게 찾을 수 있을거라 생각이 들어 코멘트 남겨봅니다..!

### 리뷰: @hakyeoung / COMMENTED

시각: 2026-09-22T10:12:46Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/18#pullrequestreview-5276679598

(본문 없음)

### 리뷰: @hakyeoung / APPROVED

시각: 2026-09-22T10:14:32Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/18#pullrequestreview-5276696154

제가 제안드린 부분 말고는 딱히 수정할 부분이 안보였던 것 같습니다 이것도 취향차이라고 생각해서 코멘트 한 번 확인만 해주시고 다른 분들 코멘트 없으면 병합해도 될 듯 합니다!

### 코드 리뷰 댓글: @kmgvv23

시각: 2026-09-22T16:10:40Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/18#discussion_r4073791614
파일: `ai/alembic/versions/0018_analyst_source_category.py` / 줄: 65
커밋: `099247a9f47e263c39ee0eeaabb7a948d0be4556`

````diff
@@ -0,0 +1,76 @@
+"""Preserve source category in the analyst report natural key.
+
+Revision ID: 0018_source_category
+Revises: 0001
+
+market으로 통합된 기존 행은 원본 URL에서 invest/daily를 복구한다.
+분류를 복구할 수 없으면 전체 변경을 롤백한다.
+"""
+
+import sqlalchemy as sa
+
+from alembic import op
+
+revision = "0018_source_category"
+down_revision = "0001"
+branch_labels = None
+depends_on = None
+
+
+def upgrade() -> None:
+    op.add_column("analyst_reports", sa.Column("source_category", sa.String(32), nullable=True))
+    op.execute("""
+        UPDATE analyst_reports
+        SET source_category = category
+        WHERE source = 'naver' AND category IN ('company', 'industry', 'economy', 'invest', 'daily')
+    """)
+    # 다른 리포트의 URL로 분류하지 않도록 source_id도 대조한다.
+    # SQLAlchemy가 정규식의 :api를 바인드 변수로 해석하지 않도록 콜론을 이스케이프한다.
+    op.execute(r"""
+        WITH original AS (
+            SELECT id, regexp_match(end_url,
+                '^https?://m[.]stock[.]naver[.]com/(?\:api/)?research/(invest|daily)/([0-9]+)(?:[?#].*)?$'
+            ) AS parts
+            FROM analyst_reports WHERE source = 'naver' AND category = 'market'
+        )
+        UPDATE analyst_reports AS report
+        SET source_category = original.parts[1]
+        FROM original
+        WHERE report.id = original.id AND original.parts[2] = report.source_id
+    """)
+    op.execute("""
+        DO $$ BEGIN
+            IF EXISTS (SELECT 1 FROM analyst_reports WHERE source_category IS NULL) THEN
+                RAISE EXCEPTION 'Cannot recover source_category. No rows were deleted. '
+                    'Inspect legacy source/category/end_url and restore verified original metadata '
+                    'before retrying; do not stamp past this migration.';
+            END IF;
+        END $$
+    """)
````

리뷰 감사합니다!
저희 구현상 현재 DB에 접속하지 않고 실행할 SQL만 출력하는 --sql도 지원하는데, 파이썬에서 DB 조회 결과를 읽으려고 하면 이 기능이 깨진다고 합니다.
그러나 에러 발생 시 원인을 쉽게 찾을 수 있도록 교체하는 것은 좋은 것 같아, 오류 실패 건수, ID등 확인 가능하도록 수정하였습니다. 감사합니다.

### 리뷰: @kmgvv23 / COMMENTED

시각: 2026-09-22T16:10:40Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/18#pullrequestreview-5280645089

(본문 없음)

## 변경 파일

- `ai/README.md` (+117 / -3)
- `ai/alembic.ini` (+44 / -0)
- `ai/alembic/env.py` (+71 / -0)
- `ai/alembic/script.py.mako` (+24 / -0)
- `ai/alembic/versions/0001_baseline_analyst_reports.py` (+86 / -0)
- `ai/alembic/versions/0018_analyst_source_category.py` (+92 / -0)
- `ai/app/collectors/__init__.py` (+0 / -0)
- `ai/app/collectors/analyst_report.py` (+130 / -0)
- `ai/app/core/database.py` (+1 / -19)
- `ai/app/models/analyst_report.py` (+26 / -4)
- `ai/app/repositories/__init__.py` (+0 / -0)
- `ai/app/repositories/analyst_report.py` (+57 / -0)
- `ai/app/services/__init__.py` (+0 / -0)
- `ai/app/services/analyst/__init__.py` (+0 / -0)
- `ai/app/services/analyst/naver.py` (+330 / -0)
- `ai/app/services/analyst/pdf.py` (+115 / -0)
- `ai/app/services/analyst/schema.py` (+73 / -0)
- `ai/pyproject.toml` (+6 / -0)
- `ai/tests/test_analyst_naver.py` (+235 / -0)
- `ai/tests/test_analyst_pdf.py` (+122 / -0)
- `ai/tests/test_analyst_report_model.py` (+24 / -8)
- `ai/tests/test_migrations.py` (+216 / -0)
- `ai/uv.lock` (+174 / -0)
