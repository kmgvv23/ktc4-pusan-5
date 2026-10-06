# #29 [AI] feature:  뉴스 링크 본문 수집·정제 tool 추가

원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/29

작성자: @swimny
작성 시각: 2026-09-22T11:29:03Z
보관 시각 (UTC): 2026-10-06T15:19:25.015347+00:00

## PR 본문

## 이번에 한 일

뉴스 링크를 받아서 그 뉴스 내용을 정제하는 tool (python 함수) 제작 (`ai/app/services/news_link/`)

**정제 방식**
1. 링크 열람: 단축 URL(`buly.kr` 등)을 따라가 최종 주소와 도메인을 확보합니다. PDF·페이월이라 본문을 못 읽어도 도메인은 남깁니다.
2. 기사 본문 문단 수집:  기사 영역을 통째로 뽑으면 "기사 공유" 같은 UI 문구가 첫 문장이 돼서, 문단 단위로 모읍니다.
3. 기사가 아닌 것을 걷어냄 
    - 버리기: 저작권 안내(`ⓒ 무단 전재 및 재배포 금지`), 유료 구독 광고, 기자 서명 줄 등을 문단째로
    - 앞머리 떼기:  `(서울=연합뉴스) OOO 기자 = ` 처럼 앞부분만 떼고 뒤 문장은 그대로 둡니다. 연합뉴스는 기자 이름 바로 뒤에 첫 문장이 붙어 있어서, 문단을 통째로 버리면 리드 문장이 같이 날아갑니다.
4. **앞 3문장 잘라냄**: 뒤에 고민했던 점 3번에서 이어서 얘기가 나옵니다.

필드는 #26 의 `stock_report_system.md` 가 정한 `messages[].link_bodies` 형식에 맞췄습니다
(`url` / `final_url` / `domain` / `title` / `excerpt` / `fetched_at`).


### 이번 PR에서 구현하지 않은 것
- 텔레그램 채널 연결 (추후에 연결하려고 놔뒀습니다)
- DB에 저장

## 고민했던 점 · 다른 선택지

### 1. 채널 연결에 대한 고민
현재는 채널 연결을 하지 않고 딱 수집 후 정제하는 것만 구현하는 것으로 결정했고, 추후 빠르게 채널 연결하면 될 것 같습니다. 웹 프리뷰 보는 형식으로 불러올 것 같습니다.

### 2. backend 부분에서의 뉴스 수집 부분, `news` db
현재 저희가 백엔드 서버와 AI 서버를 따로 쓰기로 얘기가 됐었습니다. 그런데, 확인을 해보니 이미 backend에서 네이버 뉴스를 불러와서 수집하는 코드가 존재하고 있었고, 이와 관련한 `news` db도 존재하는 것을 확인했습니다. 
서버를 따로 쓰기로 한 시점에서 백엔드 부분에서는 이 코드와 db가 존재해야 할 이유가 옅어졌고, 그래서 불러오는 API 자체도 AI에서 따로 불러오는 것으로 하는 것이 어떤지에 대한 얘기를 해보고 싶습니다. 

### 3. 발췌 방식 고민
셋 중에 골랐습니다.

| | 비용 | 인용 검증 |
|---|---|---|
| **A. 앞 3문장 잘라내기** (선택) | LLM 호출 0 | **살아 있음** |
| B. AI가 중요한 부분을 요약·추출 | 링크마다 LLM 호출 | **죽음** |
| C. 본문 전량 저장 | payload 7배 | 살아 있음 |

B는 인용 검증(보고서에 실린 문장이 원문에 실제로 있는지 문자열로 대조하는 것)이 안되고, C는 입력 토큰량이 너무 커지고, 전문을 쌓는 것이 저작권 측면에서도 부담되는 부분이 있어 제외하면서 A를 선택했습니다. 
근데 A도 보면 기사 앞 3문장이 항상 핵심은 아니기 때문에 중요 수치가 중간에 있으면 그걸 놓칠 수 있는 문제가 있습니다. 실제로 돌려본 10건에 대해서는 서두에 핵심이 들어 있어서 괜찮아 보이지만, 추후에 합의를 통해 수정하는 방향으로 해야할 것 같습니다.

## 확인해보셨으면 하는 것
현재 어떤 식으로 뉴스가 정제되는지에 대한 확인을 해주셨으면 합니다. 텔레그램 채널을 연결하는 대신 테스트파일에서 텔레그램 채널에서 가져온 링크를 직접 입력해서 정제하는 방식으로 테스트를 진행했습니다.

실행 방식은 다음과 같습니다.
```bash
cd ai

# 1. 안 깨졌는지 — 33 passed, 5 deselected
uv run pytest

# 2. 실제 뉴스 10건이 어떻게 잘리는지 눈으로 — 여기가 핵심입니다
uv run pytest -m network -s

# 3. 아무 기사나 하나 넣고 어느 규칙이 뭘 지웠는지
uv run python -m app.services.news_link https://www.yna.co.kr/view/AKR20260917177300002 --paragraphs

# 4. 규칙 목록 (네트워크 안 씁니다)
uv run python -m app.services.news_link --rules
```

2번을 돌리면 이렇게 나옵니다.

```
globalmktinsight   www.yna.co.kr  [ok] 156자
  제목  생산자물가 한 달 만에 반등…8월 폭염·유가 상승에 0.2%↑
  발췌  지난달 국내 생산자물가가 폭염과 유가상승 등의 영향으로 한 달 만에 반등했다.
        18일 한은에 따르면 올해 8월 생산자물가는 129.64로 전월(129.43) 대비 0.2% 상승했다.
        폭염 영향으로 농림수산품이 오르고, 국제유가 상승으로 전력·가스·수도 및 폐기물이 오른 영향이다.
```

3번은 어느 규칙이 무엇을 지웠는지 문단마다 보여줍니다.

```
 1 ✂ [byline_paren] (서울=연합뉴스) 이도흔 기자 =
     ✓ 지난달 국내 생산자물가가 폭염과 유가상승 등의 영향으로 한 달 만에 반등했다.
 2 ✓ 18일 한은에 따르면 올해 8월 생산자물가는 129.64로 전월(129.43) 대비 0.2% 상승했다.
```

## 리뷰와 대화

### 코드 리뷰 댓글: @hakyeoung

시각: 2026-09-22T12:59:25Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/29#discussion_r4071858291
파일: `ai/app/services/news_link/extract.py` / 줄: 79
커밋: `c3ac3294acabdf939a41936a223858f56a70156b`

````diff
@@ -0,0 +1,151 @@
+"""기사 HTML → (제목, 정제된 본문).
+
+**문단 단위로 모은다.** 기사 영역을 통째로 get_text 하면 바이라인·음성재생 UI
+문구·공유 버튼 텍스트가 앞에 붙어서 첫 문장이 그 쓰레기가 된다.
+
+`article_paragraphs` 와 `extract_body` 를 나눠 둔 것은 **정제 전 문단**을 봐야 하는
+쪽이 있기 때문이다. 정제된 결과만 돌려주면 "무엇이 지워졌는지" 를 보여줄 수가 없고,
+그게 안 보이면 clean.py 의 규칙을 손볼 수 없다.
+
+backend 는 같은 일을 trafilatura 로 한다. 여기서 bs4 를 쓰는 것은 기사 영역
+선택자를 우리가 직접 쥐고 있어야 하기 때문이다 — 텔레그램에 올라오는 링크는 국내
+매체·증권사 블로그·외신이 섞여서, 실패하면 어느 선택자를 고쳐야 하는지 보여야 한다.
+"""
+
+import re
+
+from bs4 import BeautifulSoup
+
+from app.services.news_link.clean import clean_paragraphs
+
+ARTICLE_SELECTORS = [
+    "#dic_area", "#newsct_article",  # 네이버 뉴스
+    "div.se-main-container",  # 네이버 블로그 (스마트에디터)
+    "#articleBodyContents", "div#harmonyContainer",
+    "[itemprop=articleBody]", "div.article-body", "div.article_body",
+    "div.news_end", "#CmAdContent", "div.paragraph",  # YTN
+    "article",
+]
+PARA_SELECTORS = "p, div.se-text-paragraph"
+
+# 문단으로 셀 만큼 긴가. **내용 판단은 여기서 하지 않는다.** 광고·저작권·바이라인
+# 같은 "기사가 아닌 것" 은 clean.py 가 전담한다. 두 군데서 나눠 판단하면 규칙이
+# 갈라져, 어느 쪽을 고쳐야 하는지 모르게 된다.
+MIN_PARA_CHARS = 25
+
+
+def _squash(text: str) -> str:
+    return re.sub(r"[ \t\xa0]+", " ", text).strip()
+
+
+def _keep_para(text: str) -> bool:
+    return len(text) >= MIN_PARA_CHARS
+
+
+def _common_ancestor(nodes: list) -> object | None:
+    """노드들을 모두 품는 가장 가까운 조상. 하나도 없으면 None."""
+    if not nodes:
+        return None
+    chain = [nodes[0], *nodes[0].parents]
+    for node in nodes[1:]:
+        other = {id(parent) for parent in [node, *node.parents]}
+        chain = [parent for parent in chain if id(parent) in other]
+        if not chain:
+            return None
+    return chain[0]
+
+
+def article_paragraphs(html_text: str) -> tuple[str, list[str], int]:
````

함수의 리턴값 중 마지막 정수가 무슨 의미인지 파악하기 어려운 것 같습니다..
리턴하는 값이 1과 2뿐인걸로 확인했는데 상수로 선언해주시면 코드 이해하기 더 편할 것 같습니다..!

### 리뷰: @hakyeoung / COMMENTED

시각: 2026-09-22T12:59:25Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/29#pullrequestreview-5278360330

(본문 없음)

### PR 대화 댓글: @kmgvv23

시각: 2026-09-22T13:08:54Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/29#issuecomment-5777092682

고생하셨습니다!
그런데 의문이 드는 점이 기사 앞문장 3개만 가져오는 것이 의미가 있는것인지 궁금합니다.

만약 금일 마감이어서 우선 구현한 곳까지 올리신 거라면 이해되지만,
텔레그램에서 가져오는 것을 제하더라도 뉴스 관련된 정보들 중 어떤 것을 뽑아내서 DB에 저장할 것인지 설계하는 것부터 실제 품질 테스트까지 있어야 될 것 같습니다.

### 코드 리뷰 댓글: @swimny

시각: 2026-09-22T13:24:26Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/29#discussion_r4072126059
파일: `ai/app/services/news_link/extract.py` / 줄: 79
커밋: `c3ac3294acabdf939a41936a223858f56a70156b`

````diff
@@ -0,0 +1,151 @@
+"""기사 HTML → (제목, 정제된 본문).
+
+**문단 단위로 모은다.** 기사 영역을 통째로 get_text 하면 바이라인·음성재생 UI
+문구·공유 버튼 텍스트가 앞에 붙어서 첫 문장이 그 쓰레기가 된다.
+
+`article_paragraphs` 와 `extract_body` 를 나눠 둔 것은 **정제 전 문단**을 봐야 하는
+쪽이 있기 때문이다. 정제된 결과만 돌려주면 "무엇이 지워졌는지" 를 보여줄 수가 없고,
+그게 안 보이면 clean.py 의 규칙을 손볼 수 없다.
+
+backend 는 같은 일을 trafilatura 로 한다. 여기서 bs4 를 쓰는 것은 기사 영역
+선택자를 우리가 직접 쥐고 있어야 하기 때문이다 — 텔레그램에 올라오는 링크는 국내
+매체·증권사 블로그·외신이 섞여서, 실패하면 어느 선택자를 고쳐야 하는지 보여야 한다.
+"""
+
+import re
+
+from bs4 import BeautifulSoup
+
+from app.services.news_link.clean import clean_paragraphs
+
+ARTICLE_SELECTORS = [
+    "#dic_area", "#newsct_article",  # 네이버 뉴스
+    "div.se-main-container",  # 네이버 블로그 (스마트에디터)
+    "#articleBodyContents", "div#harmonyContainer",
+    "[itemprop=articleBody]", "div.article-body", "div.article_body",
+    "div.news_end", "#CmAdContent", "div.paragraph",  # YTN
+    "article",
+]
+PARA_SELECTORS = "p, div.se-text-paragraph"
+
+# 문단으로 셀 만큼 긴가. **내용 판단은 여기서 하지 않는다.** 광고·저작권·바이라인
+# 같은 "기사가 아닌 것" 은 clean.py 가 전담한다. 두 군데서 나눠 판단하면 규칙이
+# 갈라져, 어느 쪽을 고쳐야 하는지 모르게 된다.
+MIN_PARA_CHARS = 25
+
+
+def _squash(text: str) -> str:
+    return re.sub(r"[ \t\xa0]+", " ", text).strip()
+
+
+def _keep_para(text: str) -> bool:
+    return len(text) >= MIN_PARA_CHARS
+
+
+def _common_ancestor(nodes: list) -> object | None:
+    """노드들을 모두 품는 가장 가까운 조상. 하나도 없으면 None."""
+    if not nodes:
+        return None
+    chain = [nodes[0], *nodes[0].parents]
+    for node in nodes[1:]:
+        other = {id(parent) for parent in [node, *node.parents]}
+        chain = [parent for parent in chain if id(parent) in other]
+        if not chain:
+            return None
+    return chain[0]
+
+
+def article_paragraphs(html_text: str) -> tuple[str, list[str], int]:
````

말씀주신 부분 포함 매직넘버 해당 부분들을 상수로 정리했습니다.

### 리뷰: @swimny / COMMENTED

시각: 2026-09-22T13:24:26Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/29#pullrequestreview-5278685737

(본문 없음)

### PR 대화 댓글: @swimny

시각: 2026-09-22T13:38:13Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/29#issuecomment-5777549528

@kmgvv23 
고민은 오래 했는데 결론이 애매하게 나서, 말씀하신 지적이 맞습니다.

### 1. 
앞 3문장만 가져오는 것은 현재로선 절충안이고, 추후 수정 전제입니다. 전문을 뽑는 건 저작권 문제와 비용 문제로 안 된다고 판단했고, 지금 뉴스 링크로 테스트해본 10개 모두 중심이 되는 문장이 서두에 있어서 일단 앞 3문장을 뽑는 형태로 만들었습니다.

고민했던 다른 방법은 본문 전체에서 중요 문장을 판단해 남기는 것인데, 이건 AI를 써야 하고 불러올 때마다 호출이 들어가서 토큰 사용량이 많이 늘 것 같아 보류했습니다. 추후 논의가 필요할 것 같습니다.

### 2. 
DB 설계는 진행되어야 하는 게 맞습니다만, 충돌하는 부분이 있어 재고 중이었습니다.

저장하려고 한 것은 최종 URL, 도메인, 발췌 내용, fetched_at 네 가지입니다. (fetched_at 은 나중에 수정·추가된 기사가 사후 정보로 새어드는 걸 막는 용도입니다.)

문제는 이게 이미 있는 `news` DB와 엮인다는 점입니다. `backend/app/services/news` 에 네이버 뉴스를 API로 받아 정제하는 부분이 이미 있어서, 제가 만드는 게 기존 것과 충돌하거나 이미 있는 걸 새로 만드는 꼴이 됩니다. 이전에 AI 파트가 서버를 따로 쓰기로 한 것에 따르면 이쪽 DB 관리도 AI 파트가 맡는 게 맞아 보이는데, 아직 결정된 게 없어 섣불리 진행하지 못했습니다.

이 부분을 어디서 정하면 될지 알려주시면 그에 맞춰 진행하겠습니다. 기존 정제 쪽을 제 발췌 형태로 바꿔도 되는지, 아니면 칸을 따로 두는지도 같이 정해야 할 것 같습니다.

### 3. 
품질 테스트는 1번에서 말씀드린 링크 10개 확인 정도가 제가 해본 전부이고, 그 이상은 보고서가 나와야 볼 수 있는 구조라고 생각해 보류했습니다. 혹시 이 얘기가 아니라 다른 품질 테스트를 말씀하신 거라면 어떤 걸 말씀하신 건지 알려주시면 좋겠습니다.

### PR 대화 댓글: @kmgvv23

시각: 2026-09-22T13:53:50Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/29#issuecomment-5777783030

답변 감사합니다.
1. 토큰 관련해서는 제가 증권사 리포트 요약을 생성하는데는 openrouter로 deepseek0731이라는 싼 모델을 이용했는데, 400건 하는데0.71달러 사용한 것으로 확인됩니다. 아마 뉴스도 비슷한 방식일 듯 하여 언급하신 필요없는 텍스트들을 1차 정제한 원문 따로 저장, 그리고 분량이 길다면 llm으로 요약한 것도 저장하는 방법도 고려해보면 좋을 것 같습니다.(추후에 구조가 더 잡히면, llm으로 판정하여 관련 종목, 관련 섹터 칼럼도 추가하면 될 듯 합니다.)
2. 이건 회의가 필요한 부분인 것 같습니다. 아마 AI쪽으로 옮겨오면 될 듯 합니다. 그리고 1번과 비슷한 맥락에서, 제가 만든 애널리스트 리포트 구조도 참고해서 저희가 같이 확정하면 될 듯합니다.
3. 품질 테스트는 제가 처음에 생각한 1번구조(원문+llm으로 요약(추후에는 관련종목 관련섹터 칼럼까지))에 대한 것, 수연님께서 언급해주신 필요없는 정보들(광고메세지 등의 문구)이 잘 정제되었는지 관련된 내용이었습니다.

이번 PR과 별개로, 텔레그램 뉴스 관련 메세지들을 추려 본 결과, 뉴스 링크를 첨부하고 직접 요약해주는 것, 뉴스 링크에 자신의 의견 및 코멘트를 다는 것 등 다양한 유형이 존재하는 걸 확인했습니다. 추후에 단순 링크만이 아닌, 이부분도 어떻게 함께 뉴스파트에서 정리할건지도 생각해보고 같이 회의해보면 좋을 것 같습니다.

수고하셨습니다!

### 리뷰: @kmgvv23 / APPROVED

시각: 2026-09-22T15:16:01Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/29#pullrequestreview-5280044422

(본문 없음)

## 변경 파일

- `ai/app/services/news_link/__init__.py` (+15 / -0)
- `ai/app/services/news_link/__main__.py` (+140 / -0)
- `ai/app/services/news_link/clean.py` (+152 / -0)
- `ai/app/services/news_link/extract.py` (+176 / -0)
- `ai/app/services/news_link/fetch.py` (+223 / -0)
- `ai/app/services/news_link/schema.py` (+45 / -0)
- `ai/app/services/news_link/sentence.py` (+57 / -0)
- `ai/pyproject.toml` (+8 / -0)
- `ai/tests/test_news_link_clean.py` (+67 / -0)
- `ai/tests/test_news_link_fetch.py` (+162 / -0)
- `ai/tests/test_news_link_live.py` (+142 / -0)
- `ai/tests/test_news_link_sentence.py` (+62 / -0)
- `ai/uv.lock` (+24 / -0)
