# #19 [6주차] 도커 세팅, DB 스키마 초안, 소셜 로그인, 홈 및 개별종목 ui 재구성

원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19

작성자: @stfifo
작성 시각: 2026-09-16T08:27:31Z
보관 시각 (UTC): 2026-10-06T15:19:25.015347+00:00

## PR 본문

<!--
팀 repo 의 .github/pull_request_template.md 로 배포된다.

⚠️ 이 템플릿은 `develop → main` (멘토 리뷰) PR 과 팀 내부 PR 에 **똑같이** 뜬다.
   GitHub 은 base 브랜치별 템플릿 분기를 지원하지 않는다. 그래서 두 경우 모두에서
   말이 되도록 짧게 썼다 — 팀 내부 PR 이면 아래 항목 중 필요한 것만 채우면 된다.

⚠️ 팀별로 제품·개발 방법론이 다르다(티켓). 체크리스트를 강제하지 않는 이유다.
   팀 규칙을 추가하고 싶으면 이 파일을 자유롭게 고쳐도 된다 — `.github/workflows/` 와 달리
   이 파일은 운영이 덮어쓰지 않는다.
-->
## 지난 리뷰 반영 (재리뷰 요청)

- https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/21 : [FE] 로그인·차트 피드백 반영

- https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/22 : [BE] 로그인·외부 API 피드백 반영

- https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/20 :  [BE] report_citation 정합성

## 이번에 한 일
- https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/9 : docker DB 세팅 및 뉴스 관련 수집
- https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/10 : 소셜 로그인 ui만
- https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/11 : DART, ECOS API 호출 클라이언트
- https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/12 : ai 리포트 저장용 db 스키마 초안
- https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/13 : 소셜 로그인 백엔드
- https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/14 : 소셜 로그인 FE-BE 연동
- https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/15 : 개념 페이지 관련 라우터와 개념 페이지에 상세 내용 추가 (진입점(목록·검색·브리핑 링크) 미구현 상태)
- https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/17 : 홈과 개별 종목 구성 ui 재구성하였습니다. 회의 후 다시 구성을 수정해서 아직 리팩토링이 미흡한 상태입니다..

## 리뷰에서 봐주셨으면 하는 곳

<!--
멘토 리뷰용 PR 이라면 여기가 가장 중요합니다.
"전체 봐주세요" 보다 "이 부분이 이 방식이 맞는지 모르겠습니다" 가 훨씬 좋은 리뷰를 받습니다.
예) 로그인 상태를 Context 로 관리했는데 이 규모에 과한지 / N+1 이 걱정되는 쿼리 / 에러 처리 위치
-->

- [#pr 12](https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/12) 에서 ai 리포트 저장용 db 스키마에서 source_card 하나에 텔레그램/뉴스/공시를 전부 같이 저장하도록 했는데 종류마다 테이블을 따로 만드는 대신, 종류를 구분하는 값(card_type)만 다르게 주고 한 테이블에 몰아넣었습니다. 나중에 종류가 많이 늘어나면 분리할 것 같습니다. **미리 테이블을 구분하는 게 좋을지 아니면 지금 방식대로 해도 괜찮을지 궁금합니다**

- backend/app/repositories/user.py:18-28 에서 사용자 확인(SELECT로 있는지 봄) → 행동(없으면 새로 만듦) 구조 → 같은 kakao_id로 요청 A, B가 거의 동시에 들어왔을 때 문제 발생할 수 있음을 알았습니다.
  [해당 pr 링크](https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/13)
  - A, B 둘 다 user is None이라고 판단 → 둘 다 22~28번 줄로 가서 각자 User(kakao_id=X, ...)를 만들고 session.add(user)
  - A가 먼저 await session.flush()(34번 줄) → INSERT 성공, DB에 X라는 kakao_id 행 생김
  - B도 await session.flush() → 이미 X가 존재하는데 또 넣는 셈이 되어버려서ix_users_kakao_id 유니크 인덱스가 거부 → IntegrityError
그래서 찾아보니 [**"있는지 확인" + "없으면 넣기"]를 하나로 묶어서 명령으로 보내라고 하는데 이렇게 수정방향이 맞는지 리뷰 받고 싶습니다**

- 로그인 모달 상태가 현재 페이지 전체를 리렌더시키는 문제가 있었습니다. (자식 컴포넌트들은 props가 하나도 안 바뀌어도 함수 본문이 다시 실행됨...) 아래는 해결책으로 생각해본 사항입니다
  - 1번. React.memo로 라우트 페이지를 감싸서 페이지 함수의 재실행 막기
  - **2번. loginOpen state를 Header 쪽으로 내려서 상태 위치를 바꾸기**
  - 3번. Routes를 children으로 끌어올려서 state를 그대로 두고 구조만 바꿈 (App에서 <Routes>...</Route> 를 만들어서 loginOpen을 든 컴포넌트에 children 적용
  - 위 세가지 방법에서 1번으로 해결해도 상태 위치가 잘못되었다는 문제가 남아있다는 점, 3번은 memo를 안써도 되지만 코드가 직관적일 것 같지 않아 2번 방향으로 고치려고 하는데 **2번 방식이 괜찮은 게 맞는지 리뷰 받고 싶습니다.**


- 차트 구현 과정에서 겪은 장단점을 바탕으로 라이브러리 계속 사용 여부 및 방향성에 대해 고민
  [해당 pr 링크](https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/17)
  현재 주가/거래량 및 단순 재무 추이 시각화를 위해 lightweight-charts 라이브러리를 사용해 왔습니다. 기본 툴, 캔버스 렌더링 성능 등 장점이 있었지만 단점도 있어 그냥 svg로 할지 아니면 이대로 라이브러리 사용할지 고민됩니다.
  라이브러리 단점으로는 ailwind 토큰 적용을 위해 getComputedStyle 기반 헬퍼 함수를 추가해야 하는 등 스타일 마찰과 같은 세부 ui 제어에 한계가 있다는 점이었습니다.
  **결론적으로, 없어도 되는 부분은 최대한 차트를 피하고 (list 형식으로 제공 혹은 간단한 건 svg로 차트 보여줌), 주가 차트와 같은 확대 축소 등 기능이 많이 필요한 부분은 라이브러리 사용으로 생각 중입니다. 혹시 라이브러리를 다른 걸로 교체하거나 아니면 차트 보여주는 방식을 통일하는 등 더 나은 방향이 있을지 의견 궁금합니다!**

- **AI 개발시, 데이터 다양한 방식으로 축적을 하고, 그 공유된 데이터를 가지고 아키텍쳐 설계하는 테스트를 계속 진행해야 될 것 같은데, 언제 시점에 공유 DB를 하면 좋을지 조언을 구하고 싶습니다** (ex. 양식이 정해졌을때 , 설계 방향이 명확하게 정해졌을 때 등등)


## 고민했던 점 · 다른 선택지

<!-- 두 방법 사이에서 고른 게 있다면 왜 이걸 골랐는지. 없으면 지워도 됩니다. -->

- 리포트 본문을 처음엔 그냥 JSON 하나에 통째로 저장하려고 했는데, 실제 리포트 예시를 보고 "이유별로 하나씩 행을 나누는 게" 나중에 다루기(검색, 통계 등) 편할 것 같아서 지금처럼 report_block을 따로 만들었습니다.
- 카카오 인가 코드 수신 관련하여 FastAPI엔 OAuth2 콜백 자동 처리 프레임워크가 없어서 백엔드 콜백 방식을 택해도 이득이 없다고 판단하여 프론트 수신 방식으로 채택했습니다



## 리뷰와 대화

### 코드 리뷰 댓글: @mingmingmon

시각: 2026-09-16T23:04:02Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4031548764
파일: `frontend/src/features/auth/kakaoAuth.ts` / 줄: 15
커밋: `b3ec961f4cdec8bd64230fff6603351a7be4dbe7`

````diff
@@ -0,0 +1,10 @@
+const KAKAO_AUTH_URL = 'https://kauth.kakao.com/oauth/authorize';
+
+export function buildKakaoAuthUrl(): string {
+  const params = new URLSearchParams({
+    client_id: import.meta.env.VITE_KAKAO_CLIENT_ID,
+    redirect_uri: import.meta.env.VITE_KAKAO_REDIRECT_URI,
+    response_type: 'code',
+  });
````

프론트에서 인가 코드를 받는 구조 자체는 좋은데, OAuth state 검증이 빠져 있는 것 같습니다. [kakao developers 문서 2.3](https://developers.kakao.com/docs/en/getting-started/security-guideline#state-param)에서도 CSRF 방지를 위해 로그인 요청마다 고유한 난수 state를 넣고 callback에서 동일한 값인지 확인하도록 권장하고 있습니다.

로그인 시작 시 state를 생성해 session/cookie 등에 보관 → authorize URL에 추가 → callback에서 받은 state와 비교한 뒤 code를 백엔드에 전달하는 흐름을 추가하면 좋을 것 같습니다.

### 코드 리뷰 댓글: @mingmingmon

시각: 2026-09-16T23:07:52Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4031567964
파일: `backend/app/repositories/user.py` / 줄: 23
커밋: `9e8dd0965e7a239a3d4bc57c531adce8088cda26`

````diff
@@ -0,0 +1,40 @@
+from datetime import UTC, datetime
+
+from sqlalchemy import select
+from sqlalchemy.ext.asyncio import AsyncSession
+
+from app.models import User
+
+
+async def get_or_create_user(
+    session: AsyncSession,
+    *,
+    kakao_id: str,
+    nickname: str,
+    email: str,
+) -> User:
+    #kakao_id 로 조회
+    #없으면 새로 만들고 / 있으면 정보 불러옴
+    user = (
+        await session.execute(select(User).where(User.kakao_id == kakao_id))
+    ).scalar_one_or_none()
+
+    if user is None:
+        user = User(
````

PR 본문에서 분석하신 race condition이 맞습니다!

이 경우에는 애플리케이션에서 SELECT → 없으면 INSERT를 나누기보다 PostgreSQL의 INSERT ... ON CONFLICT ... DO UPDATE/NOTHING RETURNING 같은 atomic upsert로 묶는 방향이 가장 자연스러워 보입니다.

로그인할 때 nickname / last_login_at도 갱신해야 하니 ON CONFLICT(kakao_id) DO UPDATE ... RETURNING 형태도 고려해볼 수 있을 것 같습니다. 가능하면 서로 다른 session에서 동일 kakao_id로 동시에 호출하는 테스트도 하나 있으면 좋을 것 같아요

### 코드 리뷰 댓글: @mingmingmon

시각: 2026-09-16T23:08:28Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4031571406
파일: `backend/app/models/report.py` / 줄: 96
커밋: `b3ec961f4cdec8bd64230fff6603351a7be4dbe7`

````diff
@@ -0,0 +1,83 @@
+"""
+report 결과물이 최종적으로 쌓이는 테이블.
+report_block: 리포트를 구성하는 블록(이유/참고/확인 못한 것 등) 단위.
+report_citation: 어떤 블록(또는 리포트 전체)이 어떤 근거카드를 인용했는지 (역추적용 다리).
+"""
+from datetime import UTC, datetime
+
+from sqlalchemy import BigInteger, DateTime, ForeignKey, String, Text
+from sqlalchemy.orm import Mapped, mapped_column, relationship
+
+from app.core.database import Base
+
+
+class Report(Base):
+    __tablename__ = "report"
+
+    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)
+
+    report_type: Mapped[str] = mapped_column(String(20), nullable=False)  # STOCK / MARKET
+    stock_code: Mapped[str | None] = mapped_column(String(12), nullable=True)
+
+    title: Mapped[str] = mapped_column(String(200), nullable=False)
+    # 통짜 텍스트 요약. 구조화된 본문은 report_block으로 관리한다.
+    body: Mapped[str | None] = mapped_column(Text, nullable=True)
+
+    verdict: Mapped[str | None] = mapped_column(String(20), nullable=True)  # CONFIRMED / UNCONFIRMED 등
+    status: Mapped[str] = mapped_column(String(20), nullable=False, default="DRAFT")  # DRAFT/GATED/PUBLISHED
+
+    generated_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
+    created_at: Mapped[datetime] = mapped_column(
+        DateTime(timezone=True), default=lambda: datetime.now(UTC)
+    )
+
+    blocks: Mapped[list["ReportBlock"]] = relationship(back_populates="report")
+    citations: Mapped[list["ReportCitation"]] = relationship(back_populates="report")
+
+
+class ReportBlock(Base):
+    """이유(FACTOR) / 참고(NOTE) / 확인 못한 것(UNCONFIRMED) 등 리포트 본문의 블록 하나.
+
+    FACTOR는 claim(한줄 주장)+detail(설명)을 함께 쓰고, NOTE/UNCONFIRMED는
+    claim 없이 detail만 채운다. block_type 값 목록은 카드 분류처럼 아직
+    확정 전이라 자유 문자열로 둔다.
+    """
+
+    __tablename__ = "report_block"
+
+    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)
+    report_id: Mapped[int] = mapped_column(ForeignKey("report.id"), nullable=False)
+
+    block_type: Mapped[str] = mapped_column(String(20), nullable=False)  # FACTOR / NOTE / UNCONFIRMED
+    order_index: Mapped[int] = mapped_column(nullable=False, default=0)
+
+    claim: Mapped[str | None] = mapped_column(String(300), nullable=True)  # FACTOR 전용
+    detail: Mapped[str] = mapped_column(Text, nullable=False)
+
+    created_at: Mapped[datetime] = mapped_column(
+        DateTime(timezone=True), default=lambda: datetime.now(UTC)
+    )
+
+    report: Mapped["Report"] = relationship(back_populates="blocks")
+    citations: Mapped[list["ReportCitation"]] = relationship(back_populates="block")
+
+
+class ReportCitation(Base):
+    __tablename__ = "report_citation"
+
+    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)
+    report_id: Mapped[int] = mapped_column(ForeignKey("report.id"), nullable=False)
+    # 블록 단위 인용이면 채워짐. 리포트 전체에 걸린 인용(블록으로 안 쪼개진 경우)이면 NULL.
+    block_id: Mapped[int | None] = mapped_column(ForeignKey("report_block.id"), nullable=True)
+    source_card_id: Mapped[int] = mapped_column(ForeignKey("source_card.id"), nullable=False)
````

report_id와 block_id를 둘 다 저장하면 조회는 편해질 것 같은데, 현재 FK만으로는 block_id가 반드시 같은 report_id 소속이라는 보장이 없는 것 같습니다..

예를 들어 report_id=1인데 report 2의 block_id를 넣는 row도 DB 레벨에서는 유효해 보입니다. report_id를 block을 통해 유도하도록 중복을 없애거나, (block_id, report_id) 관계를 composite FK/constraint로 보장하는 방식 등을 고려해보면 좋을 것 같습니다.

### 코드 리뷰 댓글: @mingmingmon

시각: 2026-09-16T23:09:29Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4031576807
파일: `backend/app/main.py` / 줄: 27
커밋: `b3ec961f4cdec8bd64230fff6603351a7be4dbe7`

````diff
@@ -1,18 +1,34 @@
+from contextlib import asynccontextmanager
+
 from fastapi import FastAPI
 from fastapi.exceptions import RequestValidationError
 from fastapi.middleware.cors import CORSMiddleware
+from sqlalchemy import text
 from starlette.exceptions import HTTPException as StarletteHTTPException
 
+import app.models
 from app.core.config import settings
+from app.core.database import Base, engine
 from app.core.errors import (
     AppError,
     app_error_handler,
     http_exception_handler,
     unhandled_exception_handler,
     validation_error_handler,
 )
+from app.routers import concepts
+from app.routers.auth import router as auth_router
+
+
+@asynccontextmanager
+async def lifespan(app: FastAPI):
+    async with engine.begin() as conn:
+        await conn.run_sync(Base.metadata.create_all)
+    yield
````

지금 로컬 개발 초기 단계에서는 create_all()로 빠르게 가는 게 괜찮아 보이는데, 말씀하신 공유 DB를 쓰기 시작하기 전에는 Alembic 같은 migration 체계를 도입하는 게 좋을 것 같습니다.

특히 backend와 ai가 같은 PostgreSQL을 보면서 서로 다른 Base를 가지고 있어서, 공유 DB가 생긴 뒤부터는 “누가 어떤 schema 변경을 소유하는지”와 migration history가 명확해야 할 것 같습니다. 공유 DB 도입 시점의 선행조건으로 migration 체계를 잡아두는 건 어떨까요?

### 코드 리뷰 댓글: @mingmingmon

시각: 2026-09-16T23:11:28Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4031586784
파일: `frontend/src/features/stock/components/PriceVolumeChart.tsx` / 줄: 107
커밋: `b3ec961f4cdec8bd64230fff6603351a7be4dbe7`

````diff
@@ -0,0 +1,79 @@
+import { useEffect, useRef } from 'react';
+import { CandlestickSeries, ColorType, HistogramSeries, createChart } from 'lightweight-charts';
+import { cssVar } from '../chartColors';
+import type { Candle } from '../mock';
+
+export function PriceVolumeChart({ candles }: { candles: Candle[] }) {
+  const containerRef = useRef<HTMLDivElement>(null);
+
+  useEffect(() => {
+    const el = containerRef.current;
+    if (!el) return;
+
+    const up = cssVar('--color-up', 'crimson');
+    const down = cssVar('--color-down', 'royalblue');
+    const upFill = cssVar('--color-up-100', 'mistyrose');
+    const downFill = cssVar('--color-down-100', 'aliceblue');
+    const divider = cssVar('--color-divider', 'lightgray');
+    const axisText = cssVar('--color-neutral-700', 'gray');
+
+    const chart = createChart(el, {
+      width: el.clientWidth,
+      height: 320,
+      layout: {
+        background: { type: ColorType.Solid, color: 'transparent' },
+        textColor: axisText,
+        attributionLogo: false,
+      },
+      grid: { vertLines: { visible: false }, horzLines: { color: divider } },
+      rightPriceScale: { borderColor: divider },
+      timeScale: { borderColor: divider },
+    });
+
+    const priceSeries = chart.addSeries(CandlestickSeries, {
+      upColor: up,
+      downColor: down,
+      borderVisible: false,
+      wickUpColor: up,
+      wickDownColor: down,
+    });
+    priceSeries.setData(
+      candles.map((c) => ({
+        time: c.time,
+        open: c.open,
+        high: c.high,
+        low: c.low,
+        close: c.close,
+      })),
+    );
+
+    const volumeSeries = chart.addSeries(HistogramSeries, {
+      priceFormat: { type: 'volume' },
+      priceScaleId: 'volume',
+    });
+    chart.priceScale('volume').applyOptions({ scaleMargins: { top: 0.75, bottom: 0 } });
+    chart.priceScale('right').applyOptions({ scaleMargins: { top: 0.05, bottom: 0.3 } });
+    volumeSeries.setData(
+      candles.map((c) => ({
+        time: c.time,
+        value: c.volume,
+        color: c.close >= c.open ? upFill : downFill,
+      })),
+    );
+
+    chart.timeScale().fitContent();
+
+    // 너비만 바꾸면 기존 확대 배율이 그대로 남아 최근 구간만 보이고 왼쪽이 빈다. 리사이즈 때마다 다시 맞춘다.
+    const handleResize = () => {
+      chart.applyOptions({ width: el.clientWidth });
+      chart.timeScale().fitContent();
+    };
+    window.addEventListener('resize', handleResize);
+    return () => {
+      window.removeEventListener('resize', handleResize);
+      chart.remove();
+    };
+  }, [candles]);
````

말씀해주신 것처럼 로그인 모달 state 위치 문제와 별개로 차트 쪽에서도 리렌더 비용이 크게 발생할 수 있을 것 같습니다. 2번 방식으로 해결하는 것에 동의합니다! 그런데 조금 더 신경쓰면 좋을 것 같은 부분들을 아래에 정리했습니다.

부모에서 history.slice(...)가 렌더마다 새 배열을 만들고 있고, 여기 effect가 [candles]에 의존해서 candles reference가 바뀔 때마다 기존 chart를 remove한 뒤 새 chart를 생성하게 됩니다.

차트 인스턴스 생성 effect와 데이터 갱신 effect를 분리해서 chart 자체는 mount 시 한 번만 만들고, candles 변경 시에는 series의 setData만 호출하는 방식은 어떨까요? 부모의 slice 결과도 필요하다면 useMemo로 안정화할 수 있을 것 같아요!

### 코드 리뷰 댓글: @mingmingmon

시각: 2026-09-16T23:14:16Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4031604250
파일: `backend/app/core/config.py` / 줄: 30
커밋: `b3ec961f4cdec8bd64230fff6603351a7be4dbe7`

````diff
@@ -1,11 +1,34 @@
+"""환경변수 설정. 비밀키는 여기서만 읽는다 (backend/CLAUDE.md)."""
+
 from pydantic_settings import BaseSettings, SettingsConfigDict
 
 
 class Settings(BaseSettings):
-    model_config = SettingsConfigDict(env_file=".env", extra="ignore")
+    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8", extra="ignore")
 
     app_name: str = "BASIS API"
     cors_origins: list[str] = ["http://localhost:5173"]
+    database_url: str = "postgresql+asyncpg://basis:basis@localhost:5432/basis"
+
+    # 외부 API 키
+    naver_client_id: str = ""
+    naver_client_secret: str = ""
+    kis_app_key: str = ""
+    kis_app_secret: str = ""
+    dart_api_key: str = ""
+    ecos_api_key: str = ""
+
+    # 카카오 로그인
+    kakao_client_id: str = ""
+    kakao_client_secret: str = ""
+    kakao_redirect_uri: str = ""
+
+    # 자체 세션(JWT)
+    jwt_secret_key: str = ""
+    jwt_algorithm: str = "HS256"
+    jwt_expire_minutes: int = 20160  # 14일
````

JWT secret은 설정 누락 시 빈 문자열로 서버가 정상 실행되는 것보다 startup 단계에서 바로 실패하는 편이 안전해 보입니다.

외부 API key는 해당 기능을 사용하지 않을 수도 있어서 default가 있어도 괜찮지만, 인증이 활성화된 상태의 jwt_secret_key는 required setting으로 두거나 최소 길이 검증을 두는 건 어떨까요?

### 코드 리뷰 댓글: @mingmingmon

시각: 2026-09-16T23:15:03Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4031608753
파일: `backend/app/services/dart_client.py` / 줄: 21
커밋: `9e8dd0965e7a239a3d4bc57c531adce8088cda26`

````diff
@@ -0,0 +1,32 @@
+import httpx
+
+from app.core.config import settings
+
+BASE_URL = "https://opendart.fss.or.kr/api"
+
+
+async def fetch_disclosure_list(corp_code: str, bgn_de: str, end_de: str) -> dict:
+    """GET /api/list.json — 공시 목록 조회."""
+    async with httpx.AsyncClient() as client:
+        response = await client.get(
+            f"{BASE_URL}/list.json",
+            params={
+                "crtfc_key": settings.dart_api_key,
+                "corp_code": corp_code,
+                "bgn_de": bgn_de,
+                "end_de": end_de,
+            },
+        )
+        response.raise_for_status()
+        return response.json()
````

DART는 HTTP status와 별개로 JSON body의 status 값으로 API 오류를 내려주는 것으로 알고 있습니다. 

현재 raise_for_status()만 통과하면 그대로 성공 응답처럼 반환되어서, 예를 들어 020(요청 제한 초과) 같은 응답도 caller가 직접 해석해야 할 것 같아요.

여기 client boundary에서 status == "000"을 확인하고 공통 Dart API exception으로 변환하는 방식은 어떨까요? 013(조회 데이터 없음)처럼 도메인상 정상적인 empty로 볼 값은 별도로 정의하면 좋을 것 같습니다.

### 코드 리뷰 댓글: @mingmingmon

시각: 2026-09-16T23:15:53Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4031614207
파일: `ai/app/models/analyst_report.py` / 줄: 68
커밋: `b3ec961f4cdec8bd64230fff6603351a7be4dbe7`

````diff
@@ -0,0 +1,140 @@
+"""② core — 증권사 애널리스트 리포트.
+
+한 행 = 리포트 한 편. 지금은 네이버 증권 리서치만 넣지만 `source` 컬럼으로
+텔레그램·한경컨센서스 같은 다른 출처를 같은 표에 받는다.
+
+PDF 원본은 저장하지 않는다. 임시 폴더에 받아 텍스트만 뽑고 파일은 지운다.
+    - 1주일치 PDF만 해도 수백 MB인데, 우리가 쓰는 건 본문 텍스트뿐이다.
+    - `attach_url`이 살아 있어서(2025-12 리포트도 지금 200으로 응답) 파서를
+      고치면 다시 받아 재파싱할 수 있다. 그래서 버려도 되는 것이다.
+    - `pdf_sha256`을 남겨 같은 파일이 다른 출처로 또 들어와도 알아본다.
+
+요약(summary_text)과 본문(body_text)을 따로 두는 이유: 요약은 네이버가 준
+확정 텍스트(평균 517자)고 본문은 우리가 PDF에서 뽑은 것이라 품질이 다르다.
+섞으면 나중에 무엇을 인용했는지 못 가린다.
+"""
+
+from datetime import date, datetime
+from decimal import Decimal
+
+from sqlalchemy import (
+    BigInteger,
+    Date,
+    DateTime,
+    Index,
+    Integer,
+    Numeric,
+    String,
+    Text,
+    UniqueConstraint,
+    func,
+)
+from sqlalchemy.dialects.postgresql import JSONB
+from sqlalchemy.orm import Mapped, mapped_column
+
+from app.core.database import Base
+
+
+class AnalystReport(Base):
+    __tablename__ = "analyst_reports"
+    __table_args__ = (
+        # researchId는 카테고리마다 별도 시퀀스다(company 96141, industry 46078).
+        # 그래서 (출처, 카테고리, 원본ID)가 자연키다. 재수집해도 중복이 안 생긴다.
+        UniqueConstraint("source", "category", "source_id", name="uq_analyst_report_source_id"),
+        Index("ix_analyst_report_code_date", "item_code", "write_date"),
+        Index("ix_analyst_report_date", "write_date"),
+        Index("ix_analyst_report_broker_date", "broker", "write_date"),
+        # 본문 추출 대기열을 뽑는 인덱스
+        Index("ix_analyst_report_body_status", "body_status", "write_date"),
+    )
+
+    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)
+
+    # --- 출처 --------------------------------------------------------
+    source: Mapped[str] = mapped_column(String(16), default="naver")  # naver | telegram
+    source_id: Mapped[str] = mapped_column(String(64))  # 네이버 researchId
+
+    # 리포트 **종류**. 출처가 아니다 — 출처는 source 가 말한다.
+    #     company    종목분석      특정 종목 하나
+    #     industry   산업분석      업종·테마. 종목 여러 개
+    #     economy    경제분석      금리·물가·고용 같은 거시지표와 통화정책
+    #     market     시황·투자전략  장마감·브리핑·자산배분·포트폴리오
+    #
+    # 네이버는 invest(투자전략)와 daily(시황)를 따로 주는데 **합쳐서 market 으로 받는다.**
+    # 네이버 229건으로 채점해보니 둘이 안 갈린다(3종 62~69% → 합치면 84~86%).
+    # 네이버 라벨 자체가 흔들려서다 — 같은 'Weekly' 가 증권사에 따라 invest 이기도
+    # daily 이기도 하다. 증권사가 자기 발간물을 어디에 올릴지 정하는 거라
+    # 제목·본문으로는 복원이 안 된다. 구분이 안 되는 걸 나눠두면 그 칼럼을 못 믿는다.
+    category: Mapped[str] = mapped_column(String(16))
````

여기

### 코드 리뷰 댓글: @mingmingmon

시각: 2026-09-16T23:16:36Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4031618498
파일: `ai/app/models/analyst_report.py` / 줄: 43
커밋: `b3ec961f4cdec8bd64230fff6603351a7be4dbe7`

````diff
@@ -0,0 +1,140 @@
+"""② core — 증권사 애널리스트 리포트.
+
+한 행 = 리포트 한 편. 지금은 네이버 증권 리서치만 넣지만 `source` 컬럼으로
+텔레그램·한경컨센서스 같은 다른 출처를 같은 표에 받는다.
+
+PDF 원본은 저장하지 않는다. 임시 폴더에 받아 텍스트만 뽑고 파일은 지운다.
+    - 1주일치 PDF만 해도 수백 MB인데, 우리가 쓰는 건 본문 텍스트뿐이다.
+    - `attach_url`이 살아 있어서(2025-12 리포트도 지금 200으로 응답) 파서를
+      고치면 다시 받아 재파싱할 수 있다. 그래서 버려도 되는 것이다.
+    - `pdf_sha256`을 남겨 같은 파일이 다른 출처로 또 들어와도 알아본다.
+
+요약(summary_text)과 본문(body_text)을 따로 두는 이유: 요약은 네이버가 준
+확정 텍스트(평균 517자)고 본문은 우리가 PDF에서 뽑은 것이라 품질이 다르다.
+섞으면 나중에 무엇을 인용했는지 못 가린다.
+"""
+
+from datetime import date, datetime
+from decimal import Decimal
+
+from sqlalchemy import (
+    BigInteger,
+    Date,
+    DateTime,
+    Index,
+    Integer,
+    Numeric,
+    String,
+    Text,
+    UniqueConstraint,
+    func,
+)
+from sqlalchemy.dialects.postgresql import JSONB
+from sqlalchemy.orm import Mapped, mapped_column
+
+from app.core.database import Base
+
+
+class AnalystReport(Base):
+    __tablename__ = "analyst_reports"
+    __table_args__ = (
+        # researchId는 카테고리마다 별도 시퀀스다(company 96141, industry 46078).
+        # 그래서 (출처, 카테고리, 원본ID)가 자연키다. 재수집해도 중복이 안 생긴다.
+        UniqueConstraint("source", "category", "source_id", name="uq_analyst_report_source_id"),
````

[이 부분의 설명](https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19/changes#r4031614207)대로 source_id가 원본 category별로 별도 namespace라면, invest와 daily를 둘 다 내부 market으로 normalize하는 순간 natural key가 충돌할 가능성이 생길 수도 있을 것 같아요..

내부 분석용 category=market은 유지하되 source_category 같은 원본 분류를 별도로 보존하고 unique key에는 원본 category를 사용하는 방법도 고려해볼 수 있을 것 같습니다!

### 코드 리뷰 댓글: @mingmingmon

시각: 2026-09-16T23:17:47Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4031624597
파일: `frontend/src/features/auth/AuthProvider.tsx` / 줄: 20
커밋: `88453a92043d571ad9e9edca46acb51b9e0aa17a`

````diff
@@ -0,0 +1,42 @@
+import { useEffect, useState, type ReactNode } from 'react';
+import { fetchMe, loginWithKakao as requestKakaoLogin, logout as requestLogout } from '@/lib/api';
+import type { User } from '@/lib/types';
+import { AuthContext, type AuthStatus } from './useAuth';
+
+// 앱 진입 시 GET /api/auth/me 로 세션을 복원
+// 새로고침해도 로그인 상태가 유지
+export function AuthProvider({ children }: { children: ReactNode }) {
+  const [user, setUser] = useState<User | null>(null);
+  const [status, setStatus] = useState<AuthStatus>('loading');
+
+  useEffect(() => {
+    fetchMe()
+      .then((me) => {
+        setUser(me);
+        setStatus('authenticated');
+      })
+      .catch(() => {
+        setUser(null);
+        setStatus('unauthenticated');
````

현재는 /me가 401인 경우뿐 아니라 네트워크 오류나 5xx도 모두 unauthenticated로 처리되는 것 같습니다. 일시적인 서버 장애 때문에 기존 로그인 사용자가 로그아웃된 것처럼 보일 수 있는데, 이 디테일도 고민해보시면 좋을 것 같아요!

### 리뷰: @mingmingmon / CHANGES_REQUESTED

시각: 2026-09-16T23:27:16Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#pullrequestreview-5229207941

### 질문 1. source_card 하나에 뉴스/공시/텔레그램 다 넣어도 되나요?

지금 단계에서는 source_card를 종류별로 미리 쪼개지 않아도 괜찮다고 생각합니다. 오히려 AI 입장에서는 뉴스/공시/텔레그램을 동일한 “근거” 인터페이스로 조회할 수 있다는 장점이 있어 보여요.

다만 원본 저장 영역과 AI가 소비하는 normalized 영역은 구분해서 생각하면 좋을 것 같습니다. 뉴스나 공시처럼 source별 필드와 수집 방식이 크게 다른 데이터는 각 raw/source-specific table에 저장하고, 그중 분석에 필요한 내용을 source_card라는 공통 형태로 정규화하는 방식입니다.

source_card 자체를 종류별로 분리하는 시점은 종류 개수가 많아졌을 때라기보다, card_type마다 필수 필드/제약/index/query pattern이 너무 달라져서 하나의 schema로 관리하는 비용이 커졌을 때라고 보는 게 좋을 것 같습니다.

대신 지금 payload: JSONB를 자유롭게 두고 있으니, DB table을 나누지 않더라도 card_type별 payload schema는 Pydantic 등의 application-level schema로 검증해두는 걸 추천합니다.

### 질문 2. 동일 카카오 ID race condition 수정 방향 맞나요?

네, 맞습니다. 이 부분을 찾아내셨다니,, 훌륭합니다 👍 PostgreSQL을 쓰고 있으므로 ON CONFLICT upsert가 가장 자연스러운 것 같아요. 코드 리뷰에 언급한 부분에서 더 자세히 참고해주세요!


### 질문 3. 로그인 모달 state를 Header로 내리는 2번 방식이 맞나요?

현재 로그인 모달을 여는 주체가 Header뿐이라면 말씀하신 2번 방향이 가장 자연스럽다고 생각합니다. React.memo로 rerender를 막는 것보다 상태 자체를 필요한 범위로 좁히는 게 우선이라고 봅니다.

다만 이후 관심종목/보고서 저장처럼 페이지 내부에서도 로그인 모달을 열어야 한다면 Header가 modal state를 소유하는 구조가 다시 애매해질 수 있으니, 그 시점에는 LoginModalProvider처럼 로그인 UI 상태만 별도 context로 분리하는 것도 고려하면 좋겠습니다.

그리고 현재 차트는 별개로 candles 변경마다 chart instance를 다시 생성하는 구조라서, 모달 state 위치 수정과 함께 그 부분도 확인해보시면 좋을 것 같습니다.

그렇지만 이 정도의 수정은 나중에 mvp 다 구현하고 남는 시간에 해도 되니까 지금은 당장 수정해야하는 범위를 판단하고 진행하시면 마감기한 조절에 도움이 될 것 같습니다!


### 질문 4. lightweight-charts 계속 쓸까요, SVG로 통일할까요?

말씀하신 혼용 방향이 괜찮다고 생각합니다. 차트를 모두 같은 구현체로 통일하기보다 필요한 interaction 수준을 기준으로 도구를 고르는 편이 좋을 것 같습니다.

가격 차트처럼 확대/축소, crosshair, 거래량 overlay 등이 필요한 부분은 lightweight-charts를 유지하고, 단순 추이는 SVG, 정확한 값 전달이 중요한 곳은 list/table을 쓰는 식입니다.

대신 구현 기술을 통일하기보다는 색상·숫자 포맷·tooltip·기간 선택·loading/empty 같은 사용자 경험을 공통 규칙으로 맞추는 게 더 중요해 보입니다.

다만 현재 PriceVolumeChart는 candles가 바뀔 때 chart instance 전체를 다시 만드는 구조라, 라이브러리를 교체하기 전에 lifecycle을 먼저 분리해보고 실제로 스타일링/유지보수 비용이 여전히 큰지 판단해보는 걸 추천합니다.

이것 또한 mvp 기능이 다 구현되어야만 디테일을 신경 쓴 걸 주장할 수 있기에,, 고민해보시고 일정 조율에 참고해주세요!


### 질문 5. AI 팀 공유 DB는 언제부터 쓰는 게 좋나요?

제가 기준을 잡는다면 "두 명 이상의 개발자가 같은 데이터를 바탕으로 end-to-end 흐름을 검증해야 하는 순간"인 것 같습니다.

```
collector
  ↓
raw news/report
  ↓
source_card
  ↓
agent
  ↓
report / citation
  ↓
API
```
이 중 한 vertical slice가 실제로 연결되기 시작하면 공유 DB가 유용한 시점이라고 생각합니다. 

대신 공유 DB를 먼저 열기보다 migration 체계, table/schema ownership, seed 또는 snapshot 데이터를 먼저 잡는 걸 추천합니다. 특히 지금처럼 backend와 ai가 같은 PostgreSQL을 보면서 각각 create_all을 수행하는 구조에서는 공유 DB가 생긴 이후 수동으로 schema를 맞추기 시작하면 금방 꼬일 가능성이 있습니다..

각자 개발은 local DB + fixture로 계속하고, 공유 DB는 integration/staging 데이터베이스로 쓰는 구조가 좋을 것 같아요.

즉 “schema가 완성되면 공유한다”보다는 “첫 실제 통합 테스트가 필요해질 때, migration 체계를 갖춘 상태로 공유한다” 정도로 기준을 잡으면 좋겠습니다.



### 코드 리뷰 댓글: @stfifo

시각: 2026-09-17T23:35:04Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4042290458
파일: `frontend/src/features/auth/AuthProvider.tsx` / 줄: 20
커밋: `88453a92043d571ad9e9edca46acb51b9e0aa17a`

````diff
@@ -0,0 +1,42 @@
+import { useEffect, useState, type ReactNode } from 'react';
+import { fetchMe, loginWithKakao as requestKakaoLogin, logout as requestLogout } from '@/lib/api';
+import type { User } from '@/lib/types';
+import { AuthContext, type AuthStatus } from './useAuth';
+
+// 앱 진입 시 GET /api/auth/me 로 세션을 복원
+// 새로고침해도 로그인 상태가 유지
+export function AuthProvider({ children }: { children: ReactNode }) {
+  const [user, setUser] = useState<User | null>(null);
+  const [status, setStatus] = useState<AuthStatus>('loading');
+
+  useEffect(() => {
+    fetchMe()
+      .then((me) => {
+        setUser(me);
+        setStatus('authenticated');
+      })
+      .catch(() => {
+        setUser(null);
+        setStatus('unauthenticated');
````

디테일 조언해주셔서 감사합니다!! 추가로 찾아보니 에러 유형은 client의 action level에 따라 아래 4가지로 나눈다고 하는데
- 인증만료 및 유효하지 않음 : 401 Unauthorized (혹은 403 Forbidden)
- 일시적 서버 장애 : 5xx
- 네트워크 연결문제 : Timeout, Fetch Error
- 기타 client 에러 : 기타 4xx

여기서 궁금한 건 현업에서도 크게 4가지로만 분기처리하는지 아니면 필요에 따라 좀 더 세분화하여 처리하는지가 궁금합니다!

### 리뷰: @stfifo / COMMENTED

시각: 2026-09-17T23:35:05Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#pullrequestreview-5242450784

(본문 없음)

### 코드 리뷰 댓글: @kmgvv23

시각: 2026-09-18T01:03:30Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4042797885
파일: `ai/app/models/analyst_report.py` / 줄: 43
커밋: `b3ec961f4cdec8bd64230fff6603351a7be4dbe7`

````diff
@@ -0,0 +1,140 @@
+"""② core — 증권사 애널리스트 리포트.
+
+한 행 = 리포트 한 편. 지금은 네이버 증권 리서치만 넣지만 `source` 컬럼으로
+텔레그램·한경컨센서스 같은 다른 출처를 같은 표에 받는다.
+
+PDF 원본은 저장하지 않는다. 임시 폴더에 받아 텍스트만 뽑고 파일은 지운다.
+    - 1주일치 PDF만 해도 수백 MB인데, 우리가 쓰는 건 본문 텍스트뿐이다.
+    - `attach_url`이 살아 있어서(2025-12 리포트도 지금 200으로 응답) 파서를
+      고치면 다시 받아 재파싱할 수 있다. 그래서 버려도 되는 것이다.
+    - `pdf_sha256`을 남겨 같은 파일이 다른 출처로 또 들어와도 알아본다.
+
+요약(summary_text)과 본문(body_text)을 따로 두는 이유: 요약은 네이버가 준
+확정 텍스트(평균 517자)고 본문은 우리가 PDF에서 뽑은 것이라 품질이 다르다.
+섞으면 나중에 무엇을 인용했는지 못 가린다.
+"""
+
+from datetime import date, datetime
+from decimal import Decimal
+
+from sqlalchemy import (
+    BigInteger,
+    Date,
+    DateTime,
+    Index,
+    Integer,
+    Numeric,
+    String,
+    Text,
+    UniqueConstraint,
+    func,
+)
+from sqlalchemy.dialects.postgresql import JSONB
+from sqlalchemy.orm import Mapped, mapped_column
+
+from app.core.database import Base
+
+
+class AnalystReport(Base):
+    __tablename__ = "analyst_reports"
+    __table_args__ = (
+        # researchId는 카테고리마다 별도 시퀀스다(company 96141, industry 46078).
+        # 그래서 (출처, 카테고리, 원본ID)가 자연키다. 재수집해도 중복이 안 생긴다.
+        UniqueConstraint("source", "category", "source_id", name="uq_analyst_report_source_id"),
````

멘토님 바쁘신 와중에 정성스럽게 코드리뷰 해주셔서 감사합니다!
기존에는 category를 invest와 daily로 그대로 분류해뒀다가, 이후에 pdf 분류 자체가 애매하다는 것을 인지하고, 두 category를 하나로 통합하였습니다. 그러나 이 과정에서 source_id 충돌을 차마 인지하지 못했습니다.
실제로 테스트해보니 약 9개월의 텀을 두고 source_id가 중복이 있었습니다.

멘토님 말씀대로, 원본 분류를 그대로 유지한채, 내부 분석용 Market도 쓰겠습니다!

### 리뷰: @kmgvv23 / COMMENTED

시각: 2026-09-18T01:03:30Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#pullrequestreview-5243047159

(본문 없음)

### 코드 리뷰 댓글: @jiw0nahn

시각: 2026-09-18T06:07:41Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4044241151
파일: `backend/app/models/report.py` / 줄: 96
커밋: `b3ec961f4cdec8bd64230fff6603351a7be4dbe7`

````diff
@@ -0,0 +1,83 @@
+"""
+report 결과물이 최종적으로 쌓이는 테이블.
+report_block: 리포트를 구성하는 블록(이유/참고/확인 못한 것 등) 단위.
+report_citation: 어떤 블록(또는 리포트 전체)이 어떤 근거카드를 인용했는지 (역추적용 다리).
+"""
+from datetime import UTC, datetime
+
+from sqlalchemy import BigInteger, DateTime, ForeignKey, String, Text
+from sqlalchemy.orm import Mapped, mapped_column, relationship
+
+from app.core.database import Base
+
+
+class Report(Base):
+    __tablename__ = "report"
+
+    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)
+
+    report_type: Mapped[str] = mapped_column(String(20), nullable=False)  # STOCK / MARKET
+    stock_code: Mapped[str | None] = mapped_column(String(12), nullable=True)
+
+    title: Mapped[str] = mapped_column(String(200), nullable=False)
+    # 통짜 텍스트 요약. 구조화된 본문은 report_block으로 관리한다.
+    body: Mapped[str | None] = mapped_column(Text, nullable=True)
+
+    verdict: Mapped[str | None] = mapped_column(String(20), nullable=True)  # CONFIRMED / UNCONFIRMED 등
+    status: Mapped[str] = mapped_column(String(20), nullable=False, default="DRAFT")  # DRAFT/GATED/PUBLISHED
+
+    generated_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
+    created_at: Mapped[datetime] = mapped_column(
+        DateTime(timezone=True), default=lambda: datetime.now(UTC)
+    )
+
+    blocks: Mapped[list["ReportBlock"]] = relationship(back_populates="report")
+    citations: Mapped[list["ReportCitation"]] = relationship(back_populates="report")
+
+
+class ReportBlock(Base):
+    """이유(FACTOR) / 참고(NOTE) / 확인 못한 것(UNCONFIRMED) 등 리포트 본문의 블록 하나.
+
+    FACTOR는 claim(한줄 주장)+detail(설명)을 함께 쓰고, NOTE/UNCONFIRMED는
+    claim 없이 detail만 채운다. block_type 값 목록은 카드 분류처럼 아직
+    확정 전이라 자유 문자열로 둔다.
+    """
+
+    __tablename__ = "report_block"
+
+    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)
+    report_id: Mapped[int] = mapped_column(ForeignKey("report.id"), nullable=False)
+
+    block_type: Mapped[str] = mapped_column(String(20), nullable=False)  # FACTOR / NOTE / UNCONFIRMED
+    order_index: Mapped[int] = mapped_column(nullable=False, default=0)
+
+    claim: Mapped[str | None] = mapped_column(String(300), nullable=True)  # FACTOR 전용
+    detail: Mapped[str] = mapped_column(Text, nullable=False)
+
+    created_at: Mapped[datetime] = mapped_column(
+        DateTime(timezone=True), default=lambda: datetime.now(UTC)
+    )
+
+    report: Mapped["Report"] = relationship(back_populates="blocks")
+    citations: Mapped[list["ReportCitation"]] = relationship(back_populates="block")
+
+
+class ReportCitation(Base):
+    __tablename__ = "report_citation"
+
+    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)
+    report_id: Mapped[int] = mapped_column(ForeignKey("report.id"), nullable=False)
+    # 블록 단위 인용이면 채워짐. 리포트 전체에 걸린 인용(블록으로 안 쪼개진 경우)이면 NULL.
+    block_id: Mapped[int | None] = mapped_column(ForeignKey("report_block.id"), nullable=True)
+    source_card_id: Mapped[int] = mapped_column(ForeignKey("source_card.id"), nullable=False)
````

안녕하세요 멘토님! 자세한 리뷰 감사합니다.
(block_id, report_id) 관계를 composite FK로 설정해서 보장하는 방식으로 수정하도록 했습니다. 감사합니다!

### 리뷰: @jiw0nahn / COMMENTED

시각: 2026-09-18T06:07:41Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#pullrequestreview-5244754807

(본문 없음)

### 코드 리뷰 댓글: @stfifo

시각: 2026-09-18T06:33:04Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4044369533
파일: `frontend/src/features/stock/components/PriceVolumeChart.tsx` / 줄: 107
커밋: `b3ec961f4cdec8bd64230fff6603351a7be4dbe7`

````diff
@@ -0,0 +1,79 @@
+import { useEffect, useRef } from 'react';
+import { CandlestickSeries, ColorType, HistogramSeries, createChart } from 'lightweight-charts';
+import { cssVar } from '../chartColors';
+import type { Candle } from '../mock';
+
+export function PriceVolumeChart({ candles }: { candles: Candle[] }) {
+  const containerRef = useRef<HTMLDivElement>(null);
+
+  useEffect(() => {
+    const el = containerRef.current;
+    if (!el) return;
+
+    const up = cssVar('--color-up', 'crimson');
+    const down = cssVar('--color-down', 'royalblue');
+    const upFill = cssVar('--color-up-100', 'mistyrose');
+    const downFill = cssVar('--color-down-100', 'aliceblue');
+    const divider = cssVar('--color-divider', 'lightgray');
+    const axisText = cssVar('--color-neutral-700', 'gray');
+
+    const chart = createChart(el, {
+      width: el.clientWidth,
+      height: 320,
+      layout: {
+        background: { type: ColorType.Solid, color: 'transparent' },
+        textColor: axisText,
+        attributionLogo: false,
+      },
+      grid: { vertLines: { visible: false }, horzLines: { color: divider } },
+      rightPriceScale: { borderColor: divider },
+      timeScale: { borderColor: divider },
+    });
+
+    const priceSeries = chart.addSeries(CandlestickSeries, {
+      upColor: up,
+      downColor: down,
+      borderVisible: false,
+      wickUpColor: up,
+      wickDownColor: down,
+    });
+    priceSeries.setData(
+      candles.map((c) => ({
+        time: c.time,
+        open: c.open,
+        high: c.high,
+        low: c.low,
+        close: c.close,
+      })),
+    );
+
+    const volumeSeries = chart.addSeries(HistogramSeries, {
+      priceFormat: { type: 'volume' },
+      priceScaleId: 'volume',
+    });
+    chart.priceScale('volume').applyOptions({ scaleMargins: { top: 0.75, bottom: 0 } });
+    chart.priceScale('right').applyOptions({ scaleMargins: { top: 0.05, bottom: 0.3 } });
+    volumeSeries.setData(
+      candles.map((c) => ({
+        time: c.time,
+        value: c.volume,
+        color: c.close >= c.open ? upFill : downFill,
+      })),
+    );
+
+    chart.timeScale().fitContent();
+
+    // 너비만 바꾸면 기존 확대 배율이 그대로 남아 최근 구간만 보이고 왼쪽이 빈다. 리사이즈 때마다 다시 맞춘다.
+    const handleResize = () => {
+      chart.applyOptions({ width: el.clientWidth });
+      chart.timeScale().fitContent();
+    };
+    window.addEventListener('resize', handleResize);
+    return () => {
+      window.removeEventListener('resize', handleResize);
+      chart.remove();
+    };
+  }, [candles]);
````

조언 감사합니다! PriceVolumeChart를 effect 분리해서 차트 재계산 같은 실행 빈도는 줄이도록 수정했습니다. 추가로 로그인 모달 state 처럼 리포트 관련에서도 reportStatus가 바뀔 때마다 함수 본문 재실행되는 문제 발견해서 수정했습니다! 

추가로 이런 디테일한 부분을 인식하고 관찰하는 능력은 어떻게 높일 수 있는지 조언을 구하고 싶습니다. 저는 코드 수정할 때 프론트 부분은 `"컴포넌트의 단일 책임 원칙, Props 타입 검증 및 기본값 설정, Context의 전역 상태 등 react 최적화에 따라 어떤 부분을 수정하면 좋을지 에러 및 디버깅해~"` 하는 방식으로 클로드 코드에 프롬프트를 넣어서 해결하다보니 디테일한 부분을 인식하는 게 조금 잘 안되는 것 같습니다. 이런 디버깅 과정은 직접 하는 게 아무래도 능력 기르는데 더 도움이 될까요?? (직접 하게 되면 시간이 좀 오래 걸려서 스프린트 기한을 못맞출 것 같단 걱정도 있습니다) 아니면 "이 코드는 정확히 어떤 조건에서 실행되는가"를 말로 내뱉어보는 정도로도 괜찮을지 조언을 구하고 싶습니다!

### 리뷰: @stfifo / COMMENTED

시각: 2026-09-18T06:33:04Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#pullrequestreview-5244914837

(본문 없음)

### 코드 리뷰 댓글: @stfifo

시각: 2026-09-18T07:30:52Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4044715280
파일: `frontend/src/features/auth/kakaoAuth.ts` / 줄: 15
커밋: `b3ec961f4cdec8bd64230fff6603351a7be4dbe7`

````diff
@@ -0,0 +1,10 @@
+const KAKAO_AUTH_URL = 'https://kauth.kakao.com/oauth/authorize';
+
+export function buildKakaoAuthUrl(): string {
+  const params = new URLSearchParams({
+    client_id: import.meta.env.VITE_KAKAO_CLIENT_ID,
+    redirect_uri: import.meta.env.VITE_KAKAO_REDIRECT_URI,
+    response_type: 'code',
+  });
````

조언 감사합니다!

### 리뷰: @stfifo / COMMENTED

시각: 2026-09-18T07:30:53Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#pullrequestreview-5245339086

(본문 없음)

### 코드 리뷰 댓글: @stfifo

시각: 2026-09-18T11:07:42Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4046160791
파일: `backend/app/core/config.py` / 줄: 30
커밋: `b3ec961f4cdec8bd64230fff6603351a7be4dbe7`

````diff
@@ -1,11 +1,34 @@
+"""환경변수 설정. 비밀키는 여기서만 읽는다 (backend/CLAUDE.md)."""
+
 from pydantic_settings import BaseSettings, SettingsConfigDict
 
 
 class Settings(BaseSettings):
-    model_config = SettingsConfigDict(env_file=".env", extra="ignore")
+    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8", extra="ignore")
 
     app_name: str = "BASIS API"
     cors_origins: list[str] = ["http://localhost:5173"]
+    database_url: str = "postgresql+asyncpg://basis:basis@localhost:5432/basis"
+
+    # 외부 API 키
+    naver_client_id: str = ""
+    naver_client_secret: str = ""
+    kis_app_key: str = ""
+    kis_app_secret: str = ""
+    dart_api_key: str = ""
+    ecos_api_key: str = ""
+
+    # 카카오 로그인
+    kakao_client_id: str = ""
+    kakao_client_secret: str = ""
+    kakao_redirect_uri: str = ""
+
+    # 자체 세션(JWT)
+    jwt_secret_key: str = ""
+    jwt_algorithm: str = "HS256"
+    jwt_expire_minutes: int = 20160  # 14일
````

`.env.example`의 주석에서 예시로 나와있는 JWT 토큰의 암호화 알고리즘인 HS256의 Secret Key 권장 최소 길이는 32바이트라서 최소 길이는 32자로 제한해두었습니다. 조언 감사합니다!

### 리뷰: @stfifo / COMMENTED

시각: 2026-09-18T11:07:42Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#pullrequestreview-5247099899

(본문 없음)

### 코드 리뷰 댓글: @stfifo

시각: 2026-09-18T11:53:07Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4046457641
파일: `backend/app/services/dart_client.py` / 줄: 21
커밋: `9e8dd0965e7a239a3d4bc57c531adce8088cda26`

````diff
@@ -0,0 +1,32 @@
+import httpx
+
+from app.core.config import settings
+
+BASE_URL = "https://opendart.fss.or.kr/api"
+
+
+async def fetch_disclosure_list(corp_code: str, bgn_de: str, end_de: str) -> dict:
+    """GET /api/list.json — 공시 목록 조회."""
+    async with httpx.AsyncClient() as client:
+        response = await client.get(
+            f"{BASE_URL}/list.json",
+            params={
+                "crtfc_key": settings.dart_api_key,
+                "corp_code": corp_code,
+                "bgn_de": bgn_de,
+                "end_de": end_de,
+            },
+        )
+        response.raise_for_status()
+        return response.json()
````

조언해주신대로 수정했습니다. 추가적으로 dart의 고유번호 요청(GET /api/corpCode.xml) 응답의 경우, 출력 포맷이 Zip FILE (binary)이고 에러 응답은 xml 형식이라 HTTP 레벨에서는 정상으로 응답하여(HTTP 200) 현재 함수의 성공과 실패를 구분할 수 없는 문제점이 있었습니다. 이 문제에 대해서는 다음 주차 때 해당 정보를 실제로 사용하면 고치도록 하겠습니다!

### 리뷰: @stfifo / COMMENTED

시각: 2026-09-18T11:53:07Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#pullrequestreview-5247459247

(본문 없음)

### 코드 리뷰 댓글: @stfifo

시각: 2026-09-18T12:19:06Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4046643713
파일: `backend/app/repositories/user.py` / 줄: 23
커밋: `9e8dd0965e7a239a3d4bc57c531adce8088cda26`

````diff
@@ -0,0 +1,40 @@
+from datetime import UTC, datetime
+
+from sqlalchemy import select
+from sqlalchemy.ext.asyncio import AsyncSession
+
+from app.models import User
+
+
+async def get_or_create_user(
+    session: AsyncSession,
+    *,
+    kakao_id: str,
+    nickname: str,
+    email: str,
+) -> User:
+    #kakao_id 로 조회
+    #없으면 새로 만들고 / 있으면 정보 불러옴
+    user = (
+        await session.execute(select(User).where(User.kakao_id == kakao_id))
+    ).scalar_one_or_none()
+
+    if user is None:
+        user = User(
````

신규 가입이든 기존 로그인이든 흐름이 동일하여 DO UPDATE ... RETURNING 방식으로 진행했습니다. 테스트 관련 조언도 감사합니다!

### 리뷰: @stfifo / COMMENTED

시각: 2026-09-18T12:19:06Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#pullrequestreview-5247679333

(본문 없음)

### 코드 리뷰 댓글: @jiw0nahn

시각: 2026-09-18T22:03:56Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4051047263
파일: `backend/app/main.py` / 줄: 27
커밋: `b3ec961f4cdec8bd64230fff6603351a7be4dbe7`

````diff
@@ -1,18 +1,34 @@
+from contextlib import asynccontextmanager
+
 from fastapi import FastAPI
 from fastapi.exceptions import RequestValidationError
 from fastapi.middleware.cors import CORSMiddleware
+from sqlalchemy import text
 from starlette.exceptions import HTTPException as StarletteHTTPException
 
+import app.models
 from app.core.config import settings
+from app.core.database import Base, engine
 from app.core.errors import (
     AppError,
     app_error_handler,
     http_exception_handler,
     unhandled_exception_handler,
     validation_error_handler,
 )
+from app.routers import concepts
+from app.routers.auth import router as auth_router
+
+
+@asynccontextmanager
+async def lifespan(app: FastAPI):
+    async with engine.begin() as conn:
+        await conn.run_sync(Base.metadata.create_all)
+    yield
````

말씀해주신대로 마이그레이션 체계를 도입하기로 결정했습니다!
그리고 누가 어떤 Schema 변경을 소유하는지를 명확하게 해야할 것 같아서 추후 더 논의해보도록 했습니다. 항상 정성스러운 리뷰 감사합니다. 

### 리뷰: @jiw0nahn / COMMENTED

시각: 2026-09-18T22:03:56Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#pullrequestreview-5253041236

(본문 없음)

### 코드 리뷰 댓글: @mingmingmon

시각: 2026-09-19T11:44:42Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4053093957
파일: `frontend/src/features/stock/components/PriceVolumeChart.tsx` / 줄: 107
커밋: `b3ec961f4cdec8bd64230fff6603351a7be4dbe7`

````diff
@@ -0,0 +1,79 @@
+import { useEffect, useRef } from 'react';
+import { CandlestickSeries, ColorType, HistogramSeries, createChart } from 'lightweight-charts';
+import { cssVar } from '../chartColors';
+import type { Candle } from '../mock';
+
+export function PriceVolumeChart({ candles }: { candles: Candle[] }) {
+  const containerRef = useRef<HTMLDivElement>(null);
+
+  useEffect(() => {
+    const el = containerRef.current;
+    if (!el) return;
+
+    const up = cssVar('--color-up', 'crimson');
+    const down = cssVar('--color-down', 'royalblue');
+    const upFill = cssVar('--color-up-100', 'mistyrose');
+    const downFill = cssVar('--color-down-100', 'aliceblue');
+    const divider = cssVar('--color-divider', 'lightgray');
+    const axisText = cssVar('--color-neutral-700', 'gray');
+
+    const chart = createChart(el, {
+      width: el.clientWidth,
+      height: 320,
+      layout: {
+        background: { type: ColorType.Solid, color: 'transparent' },
+        textColor: axisText,
+        attributionLogo: false,
+      },
+      grid: { vertLines: { visible: false }, horzLines: { color: divider } },
+      rightPriceScale: { borderColor: divider },
+      timeScale: { borderColor: divider },
+    });
+
+    const priceSeries = chart.addSeries(CandlestickSeries, {
+      upColor: up,
+      downColor: down,
+      borderVisible: false,
+      wickUpColor: up,
+      wickDownColor: down,
+    });
+    priceSeries.setData(
+      candles.map((c) => ({
+        time: c.time,
+        open: c.open,
+        high: c.high,
+        low: c.low,
+        close: c.close,
+      })),
+    );
+
+    const volumeSeries = chart.addSeries(HistogramSeries, {
+      priceFormat: { type: 'volume' },
+      priceScaleId: 'volume',
+    });
+    chart.priceScale('volume').applyOptions({ scaleMargins: { top: 0.75, bottom: 0 } });
+    chart.priceScale('right').applyOptions({ scaleMargins: { top: 0.05, bottom: 0.3 } });
+    volumeSeries.setData(
+      candles.map((c) => ({
+        time: c.time,
+        value: c.volume,
+        color: c.close >= c.open ? upFill : downFill,
+      })),
+    );
+
+    chart.timeScale().fitContent();
+
+    // 너비만 바꾸면 기존 확대 배율이 그대로 남아 최근 구간만 보이고 왼쪽이 빈다. 리사이즈 때마다 다시 맞춘다.
+    const handleResize = () => {
+      chart.applyOptions({ width: el.clientWidth });
+      chart.timeScale().fitContent();
+    };
+    window.addEventListener('resize', handleResize);
+    return () => {
+      window.removeEventListener('resize', handleResize);
+      chart.remove();
+    };
+  }, [candles]);
````

좋은 질문입니다! 개인적으로는 AI를 안 쓰고 전부 직접 디버깅하는 것보다, **AI를 쓰더라도 먼저 본인이 실행 흐름을 한 번 설명해보는 습관**을 들이는 게 가장 효과적이라고 생각합니다.

이번 케이스도 결국 핵심은 “이 코드는 언제 다시 실행되는가?”를 따라가보는 것이었습니다.

예를 들어 React 코드를 볼 때는 먼저

* 이 컴포넌트는 어떤 state/props 변화에서 다시 렌더되는가?
* 렌더될 때 새로 만들어지는 객체나 배열은 무엇인가?
* `useEffect` dependency 중 reference가 매번 바뀌는 값은 없는가?
* effect가 다시 실행되면 cleanup에서는 무엇을 없애고, 다시 무엇을 만드는가?
* 이 작업이 단순 계산인지, DOM/차트/네트워크처럼 비용이 큰 작업인지?

정도만 말로 따라가봐도 이런 문제를 발견할 확률이 많이 올라갑니다.

그래서 `"React 최적화 관점에서 수정해줘"`라고 바로 AI에게 맡기기보다는,

1. 먼저 현재 코드가 어떤 조건에서 실행되는지 직접 설명해보기
2. “여기가 불필요하게 다시 실행될 것 같다”는 가설을 하나 세우기
3. React DevTools Profiler나 로그로 실제 실행 횟수를 확인하기
4. 그다음 AI에게 가설 검증이나 개선 방법을 물어보기

정도의 흐름을 추천합니다.

이번에 `reportStatus` 쪽에서 비슷한 문제를 직접 추가로 발견하신 것도 좋은 흐름이라고 생각합니다. 이런 식으로 한 번 발견한 패턴을 다른 코드에도 적용해보는 게 결국 관찰력을 키우는 가장 빠른 방법이라고 생각합니다!


### 리뷰: @mingmingmon / COMMENTED

시각: 2026-09-19T11:44:42Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#pullrequestreview-5255632803

(본문 없음)

### 코드 리뷰 댓글: @mingmingmon

시각: 2026-09-19T11:45:24Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4053095013
파일: `frontend/src/features/auth/AuthProvider.tsx` / 줄: 20
커밋: `88453a92043d571ad9e9edca46acb51b9e0aa17a`

````diff
@@ -0,0 +1,42 @@
+import { useEffect, useState, type ReactNode } from 'react';
+import { fetchMe, loginWithKakao as requestKakaoLogin, logout as requestLogout } from '@/lib/api';
+import type { User } from '@/lib/types';
+import { AuthContext, type AuthStatus } from './useAuth';
+
+// 앱 진입 시 GET /api/auth/me 로 세션을 복원
+// 새로고침해도 로그인 상태가 유지
+export function AuthProvider({ children }: { children: ReactNode }) {
+  const [user, setUser] = useState<User | null>(null);
+  const [status, setStatus] = useState<AuthStatus>('loading');
+
+  useEffect(() => {
+    fetchMe()
+      .then((me) => {
+        setUser(me);
+        setStatus('authenticated');
+      })
+      .catch(() => {
+        setUser(null);
+        setStatus('unauthenticated');
````

보통 현업에서도 에러를 무조건 정해진 4가지 종류로만 나눈다기보다는, **에러 종류에 따라 클라이언트가 취해야 할 행동이 달라지는 만큼** 세분화하는 편이라고 생각하시면 좋습니다.

예를 들어 지금 `/me` 요청을 기준으로 보면 저는 우선 이렇게 생각할 것 같습니다.

* `401 Unauthorized`: 인증 정보가 없거나 만료됨 → `unauthenticated`로 전환
* `403 Forbidden`: 인증은 되었지만 해당 리소스에 대한 권한이 없음 → 401과는 별개의 권한 오류
* `5xx`: 서버의 일시적인 문제 → 로그인 해제로 판단하지 않고 error/unknown 상태
* timeout / network error: 서버 응답 자체를 확인하지 못함 → 역시 로그인 해제로 판단하지 않고 재시도 가능 상태
* 기타 `4xx`: 요청 자체가 잘못된 경우 → 상황에 따라 별도 처리

특히 `401`과 `403`은 구분해두는 게 좋습니다.
`401`은 보통 **"누구인지 인증되지 않았다"**, `403`은 **"누구인지는 알지만 이 작업을 할 권한이 없다"**에 가깝기 때문에 UI에서 취할 행동도 달라질 수 있습니다.

그렇다고 HTTP status code마다 전부 별도의 상태를 만들 필요는 없습니다. 예를 들어 500, 502, 503을 UI에서 모두 똑같이 "일시적인 서버 오류, 재시도"로 처리한다면 프론트에서는 하나의 `serverError` 범주로 묶어도 괜찮습니다.

결국 기준은 **HTTP 코드가 몇 번이냐보다 사용자가 다음에 무엇을 해야 하느냐**라고 생각하면 좋습니다.

예를 들어

* 다시 로그인해야 한다
* 잠시 후 재시도하면 된다
* 권한이 없다고 안내해야 한다
* 잘못된 입력을 수정해야 한다

처럼 행동이 달라지는 지점에서 분류를 나누면 됩니다.

그리고 규모가 커지면 모든 컴포넌트에서 `if status === 401 ...`을 반복하기보다는 HTTP client 계층에서 공통적인 에러 타입으로 한 번 변환하고, 각 feature에서는 필요한 것만 처리하는 구조도 많이 사용합니다.

이번 `/me`의 경우에는 우선 `401 → unauthenticated`, `network/5xx → auth 상태를 확정하지 못한 error` 정도만 구분해도 충분해 보입니다. 처음부터 모든 가능성을 세밀하게 모델링하기보다 실제로 필요한 UX가 생길 때 분류를 추가하는 게 좋을 것 같습니다!


### 리뷰: @mingmingmon / COMMENTED

시각: 2026-09-19T11:45:24Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#pullrequestreview-5255633953

(본문 없음)

### 리뷰: @mingmingmon / APPROVED

시각: 2026-09-19T11:46:59Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#pullrequestreview-5255636727

다들 각자 구현한 파트에 대해서 피드백 드린 내용을 반영해주셔서 감사합니다!

리뷰에서 궁금한 점이나 어려운 부분이 있다면 답글에 달아주시면 2차 리뷰에서 확인하도록 하겠습니다! 편하게 달아주세요 :)

6주차도 고생하셨습니다!

### 코드 리뷰 댓글: @stfifo

시각: 2026-09-19T14:52:05Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4053500558
파일: `frontend/src/features/stock/components/PriceVolumeChart.tsx` / 줄: 107
커밋: `b3ec961f4cdec8bd64230fff6603351a7be4dbe7`

````diff
@@ -0,0 +1,79 @@
+import { useEffect, useRef } from 'react';
+import { CandlestickSeries, ColorType, HistogramSeries, createChart } from 'lightweight-charts';
+import { cssVar } from '../chartColors';
+import type { Candle } from '../mock';
+
+export function PriceVolumeChart({ candles }: { candles: Candle[] }) {
+  const containerRef = useRef<HTMLDivElement>(null);
+
+  useEffect(() => {
+    const el = containerRef.current;
+    if (!el) return;
+
+    const up = cssVar('--color-up', 'crimson');
+    const down = cssVar('--color-down', 'royalblue');
+    const upFill = cssVar('--color-up-100', 'mistyrose');
+    const downFill = cssVar('--color-down-100', 'aliceblue');
+    const divider = cssVar('--color-divider', 'lightgray');
+    const axisText = cssVar('--color-neutral-700', 'gray');
+
+    const chart = createChart(el, {
+      width: el.clientWidth,
+      height: 320,
+      layout: {
+        background: { type: ColorType.Solid, color: 'transparent' },
+        textColor: axisText,
+        attributionLogo: false,
+      },
+      grid: { vertLines: { visible: false }, horzLines: { color: divider } },
+      rightPriceScale: { borderColor: divider },
+      timeScale: { borderColor: divider },
+    });
+
+    const priceSeries = chart.addSeries(CandlestickSeries, {
+      upColor: up,
+      downColor: down,
+      borderVisible: false,
+      wickUpColor: up,
+      wickDownColor: down,
+    });
+    priceSeries.setData(
+      candles.map((c) => ({
+        time: c.time,
+        open: c.open,
+        high: c.high,
+        low: c.low,
+        close: c.close,
+      })),
+    );
+
+    const volumeSeries = chart.addSeries(HistogramSeries, {
+      priceFormat: { type: 'volume' },
+      priceScaleId: 'volume',
+    });
+    chart.priceScale('volume').applyOptions({ scaleMargins: { top: 0.75, bottom: 0 } });
+    chart.priceScale('right').applyOptions({ scaleMargins: { top: 0.05, bottom: 0.3 } });
+    volumeSeries.setData(
+      candles.map((c) => ({
+        time: c.time,
+        value: c.volume,
+        color: c.close >= c.open ? upFill : downFill,
+      })),
+    );
+
+    chart.timeScale().fitContent();
+
+    // 너비만 바꾸면 기존 확대 배율이 그대로 남아 최근 구간만 보이고 왼쪽이 빈다. 리사이즈 때마다 다시 맞춘다.
+    const handleResize = () => {
+      chart.applyOptions({ width: el.clientWidth });
+      chart.timeScale().fitContent();
+    };
+    window.addEventListener('resize', handleResize);
+    return () => {
+      window.removeEventListener('resize', handleResize);
+      chart.remove();
+    };
+  }, [candles]);
````

답변 감사합니다!!

### 리뷰: @stfifo / COMMENTED

시각: 2026-09-19T14:52:05Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#pullrequestreview-5256067467

(본문 없음)

### 코드 리뷰 댓글: @stfifo

시각: 2026-09-19T15:00:05Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#discussion_r4053523017
파일: `frontend/src/features/auth/AuthProvider.tsx` / 줄: 20
커밋: `88453a92043d571ad9e9edca46acb51b9e0aa17a`

````diff
@@ -0,0 +1,42 @@
+import { useEffect, useState, type ReactNode } from 'react';
+import { fetchMe, loginWithKakao as requestKakaoLogin, logout as requestLogout } from '@/lib/api';
+import type { User } from '@/lib/types';
+import { AuthContext, type AuthStatus } from './useAuth';
+
+// 앱 진입 시 GET /api/auth/me 로 세션을 복원
+// 새로고침해도 로그인 상태가 유지
+export function AuthProvider({ children }: { children: ReactNode }) {
+  const [user, setUser] = useState<User | null>(null);
+  const [status, setStatus] = useState<AuthStatus>('loading');
+
+  useEffect(() => {
+    fetchMe()
+      .then((me) => {
+        setUser(me);
+        setStatus('authenticated');
+      })
+      .catch(() => {
+        setUser(null);
+        setStatus('unauthenticated');
````

기존 코드 바탕으로 예시와 함께 설명해주셔서 감사합니다!

### 리뷰: @stfifo / COMMENTED

시각: 2026-09-19T15:00:05Z
원문: https://github.com/kakaotechcampus-4/ktc4-pusan-5/pull/19#pullrequestreview-5256094655

(본문 없음)

## 변경 파일

- `.gitignore` (+4 / -0)
- `ai/.env.example` (+8 / -0)
- `ai/README.md` (+88 / -0)
- `ai/app/__init__.py` (+0 / -0)
- `ai/app/core/__init__.py` (+0 / -0)
- `ai/app/core/config.py` (+21 / -0)
- `ai/app/core/database.py` (+31 / -0)
- `ai/app/models/__init__.py` (+3 / -0)
- `ai/app/models/analyst_report.py` (+140 / -0)
- `ai/pyproject.toml` (+28 / -0)
- `ai/tests/__init__.py` (+0 / -0)
- `ai/tests/test_analyst_report_model.py` (+44 / -0)
- `ai/uv.lock` (+423 / -0)
- `backend/.env.example` (+28 / -0)
- `backend/app/__init__.py` (+0 / -0)
- `backend/app/collectors/news.py` (+51 / -0)
- `backend/app/core/__init__.py` (+0 / -0)
- `backend/app/core/config.py` (+25 / -1)
- `backend/app/core/database.py` (+20 / -0)
- `backend/app/core/dependencies.py` (+31 / -0)
- `backend/app/core/security.py` (+27 / -0)
- `backend/app/core/taxonomy.py` (+67 / -0)
- `backend/app/main.py` (+23 / -2)
- `backend/app/models/__init__.py` (+17 / -0)
- `backend/app/models/channel.py` (+37 / -0)
- `backend/app/models/concept.py` (+32 / -0)
- `backend/app/models/news.py` (+22 / -0)
- `backend/app/models/report.py` (+107 / -0)
- `backend/app/models/source_card.py` (+59 / -0)
- `backend/app/models/user.py` (+21 / -0)
- `backend/app/repositories/concept.py` (+61 / -0)
- `backend/app/repositories/news.py` (+13 / -0)
- `backend/app/repositories/user.py` (+38 / -0)
- `backend/app/routers/auth.py` (+76 / -0)
- `backend/app/routers/concepts.py` (+38 / -0)
- `backend/app/schemas/auth.py` (+11 / -0)
- `backend/app/schemas/base.py` (+12 / -0)
- `backend/app/schemas/concept.py` (+149 / -0)
- `backend/app/seeds/concepts.py` (+151 / -0)
- `backend/app/services/__init__.py` (+0 / -0)
- `backend/app/services/dart_client.py` (+53 / -0)
- `backend/app/services/ecos_client.py` (+37 / -0)
- `backend/app/services/kakao.py` (+65 / -0)
- `backend/app/services/news/__init__.py` (+3 / -0)
- `backend/app/services/news/__main__.py` (+52 / -0)
- `backend/app/services/news/clean.py` (+82 / -0)
- `backend/app/services/news/content.py` (+57 / -0)
- `backend/app/services/news/naver.py` (+71 / -0)
- `backend/app/services/news/schema.py` (+21 / -0)
- `backend/docker-compose.yml` (+22 / -0)
- `backend/pyproject.toml` (+13 / -0)
- `backend/seeds/concepts/capex.json` (+44 / -0)
- `backend/seeds/concepts/circular-financing.json` (+44 / -0)
- `backend/seeds/concepts/depreciation.json` (+35 / -0)
- `backend/seeds/concepts/disclosure.json` (+35 / -0)
- `backend/seeds/concepts/dividend.json` (+35 / -0)
- `backend/seeds/concepts/financial-statements.json` (+38 / -0)
- `backend/seeds/concepts/interest-rate.json` (+44 / -0)
- `backend/seeds/concepts/per.json` (+44 / -0)
- `backend/seeds/concepts/roe.json` (+35 / -0)
- `backend/seeds/concepts/shareholder-return.json` (+44 / -0)
- `backend/seeds/concepts/treasury-stock.json` (+45 / -0)
- `backend/seeds/taxonomy.json` (+134 / -0)
- `backend/tests/__init__.py` (+0 / -0)
- `backend/tests/test_auth_router.py` (+90 / -0)
- `backend/tests/test_clean.py` (+19 / -0)
- `backend/tests/test_collector.py` (+19 / -0)
- `backend/tests/test_concepts_router.py` (+130 / -0)
- `backend/tests/test_content.py` (+24 / -0)
- `backend/tests/test_kakao_service.py` (+61 / -0)
- `backend/tests/test_naver.py` (+46 / -0)
- `backend/tests/test_repository.py` (+35 / -0)
- `backend/tests/test_seed_concepts.py` (+133 / -0)
- `backend/tests/test_taxonomy.py` (+56 / -0)
- `backend/tests/test_user_repository.py` (+72 / -0)
- `backend/uv.lock` (+825 / -0)
- `frontend/.env.example` (+6 / -0)
- `frontend/CLAUDE.md` (+10 / -0)
- `frontend/convention.md` (+44 / -0)
- `frontend/package-lock.json` (+1270 / -81)
- `frontend/package.json` (+2 / -0)
- `frontend/src/App.tsx` (+13 / -6)
- `frontend/src/components/layout/Header.tsx` (+21 / -6)
- `frontend/src/components/layout/PageShell.tsx` (+27 / -7)
- `frontend/src/components/ui/Button.tsx` (+9 / -10)
- `frontend/src/components/ui/InfoTip.tsx` (+33 / -0)
- `frontend/src/components/ui/Markdown.tsx` (+58 / -0)
- `frontend/src/components/ui/Modal.tsx` (+49 / -0)
- `frontend/src/components/ui/Stat.tsx` (+3 / -3)
- `frontend/src/components/ui/Tabs.tsx` (+38 / -0)
- `frontend/src/components/ui/index.ts` (+4 / -0)
- `frontend/src/features/auth/AuthProvider.tsx` (+75 / -0)
- `frontend/src/features/auth/KakaoCallbackPage.tsx` (+88 / -0)
- `frontend/src/features/auth/LoginModal.tsx` (+35 / -0)
- `frontend/src/features/auth/components/KakaoLoginButton.tsx` (+28 / -0)
- `frontend/src/features/auth/kakaoAuth.ts` (+27 / -0)
- `frontend/src/features/auth/useAuth.ts` (+25 / -0)
- `frontend/src/features/concepts/ConceptDetail.tsx` (+91 / -0)
- `frontend/src/features/concepts/ConceptPage.tsx` (+8 / -0)
- `frontend/src/features/concepts/components/ConceptBody.tsx` (+13 / -0)
- `frontend/src/features/concepts/components/ConceptHeader.tsx` (+34 / -0)
- `frontend/src/features/concepts/components/ConceptMeta.tsx` (+29 / -0)
- `frontend/src/features/concepts/components/ConceptQuiz.tsx` (+79 / -0)
- `frontend/src/features/concepts/components/ConceptRelated.tsx` (+28 / -0)
- `frontend/src/features/concepts/mock.ts` (+120 / -0)
- `frontend/src/features/concepts/types.ts` (+56 / -0)
- `frontend/src/features/concepts/useConcept.ts` (+94 / -0)
- `frontend/src/features/home/HomePage.tsx` (+56 / -56)
- `frontend/src/features/home/components/HomeHero.tsx` (+3 / -28)
- `frontend/src/features/home/components/IndexTiles.tsx` (+3 / -2)
- `frontend/src/features/home/components/MarketFlowSection.tsx` (+134 / -0)
- `frontend/src/features/home/components/MarketInsightSection.tsx` (+64 / -0)
- `frontend/src/features/home/components/RankingSection.tsx` (+106 / -0)
- `frontend/src/features/home/components/SignalList.tsx` (+1 / -5)
- `frontend/src/features/home/components/StockSearchBox.tsx` (+82 / -0)
- `frontend/src/features/home/mock.ts` (+245 / -2)
- `frontend/src/features/home/useHomeSections.ts` (+66 / -0)
- `frontend/src/features/stock/StockBriefingPage.tsx` (+20 / -80)
- `frontend/src/features/stock/chartColors.ts` (+10 / -0)
- `frontend/src/features/stock/components/AiReportSection.tsx` (+67 / -0)
- `frontend/src/features/stock/components/AtAGlanceCard.tsx` (+76 / -0)
- `frontend/src/features/stock/components/BriefingSection.tsx` (+0 / -48)
- `frontend/src/features/stock/components/DetailInfoTab.tsx` (+27 / -0)
- `frontend/src/features/stock/components/FinancialHealthSummary.tsx` (+55 / -0)
- `frontend/src/features/stock/components/FinancialTrendTable.tsx` (+65 / -0)
- `frontend/src/features/stock/components/PriceActionSection.tsx` (+28 / -0)
- `frontend/src/features/stock/components/PriceVolumeChart.tsx` (+110 / -0)
- `frontend/src/features/stock/components/QuarterlyMetricsTable.tsx` (+118 / -0)
- `frontend/src/features/stock/components/StockInsightSection.tsx` (+44 / -0)
- `frontend/src/features/stock/components/StockStats.tsx` (+0 / -24)
- `frontend/src/features/stock/mock.ts` (+484 / -38)
- `frontend/src/features/stock/useStockDetail.ts` (+57 / -0)
- `frontend/src/lib/api.ts` (+94 / -0)
- `frontend/src/lib/format.ts` (+19 / -7)
- `frontend/src/lib/types.ts` (+12 / -0)
- `frontend/src/styles/theme.css` (+3 / -6)
