# Apex Crypto Trader — Codex 개발 프롬프트

너는 세계 최고 수준의 풀스택 엔지니어, 퀀트 트레이딩 시스템 아키텍트, 크립토 거래소 API 전문가, 프리미엄 SaaS UI/UX 디자이너다.

목표는 Vercel에 배포 가능한 업계 최고 수준의 자동 코인 트레이딩 웹앱을 개발하는 것이다.

앱 이름은 **Apex Crypto Trader** 로 한다.

이 앱은 단순 시그널 앱이 아니라 다음 기능을 모두 갖춘 실전형 자동매매 SaaS다.

1. Binance Futures 실시간 데이터 분석
2. 일봉 / 4시간봉 / 15분봉 멀티타임프레임 전략 분석
3. 자동 진입 시그널 탐지
4. 손절가 / 익절가 / 손익비 / 포지션 크기 자동 계산
5. Paper Trading
6. Live Trading
7. API 키 보안 저장 구조
8. 실거래 전 안전 확인 시스템
9. 주문 실행 로그
10. 포지션 모니터링
11. 거래 내역 대시보드
12. 프리미엄 다크모드 UI
13. Vercel 배포 가능 구조

초기 거래소는 **Binance Futures USDT-M** 기준으로 구현한다.

기본값은 반드시 **Signal Only** 또는 **Paper Trading Mode**다.  
Live Trading은 사용자가 명시적으로 활성화해야 한다.  
Live Trading 활성화 전에는 반드시 위험 확인 체크박스를 통과해야 한다.

---

## 1. 기술 스택

다음 기술 스택으로 개발한다.

- Next.js 14 이상 App Router
- TypeScript
- Tailwind CSS
- shadcn/ui
- Framer Motion
- Recharts 또는 Lightweight Charts
- Zustand
- Prisma
- PostgreSQL
- NextAuth 또는 자체 인증 구조
- Binance Futures API
- Server Actions 또는 API Routes

Vercel 배포를 전제로 한다.

환경변수는 다음을 사용한다.

```env
DATABASE_URL=
NEXTAUTH_SECRET=
BINANCE_API_KEY=
BINANCE_API_SECRET=
ENCRYPTION_SECRET=
```

중요:

- API Key와 Secret은 절대 클라이언트로 노출하지 않는다.
- 모든 주문 실행은 서버 API Route에서만 처리한다.
- 클라이언트에서는 주문 요청만 보낸다.
- API Secret은 localStorage, sessionStorage, browser cookie에 평문 저장하지 않는다.
- GitHub에 `.env`가 올라가지 않도록 `.gitignore`를 반드시 구성한다.

---

## 2. 핵심 전략 규칙

## STEP 1 — 방향 설정: 일봉 EMA 200

일봉 기준 EMA 200을 계산한다.

규칙:

- 일봉 종가 > EMA 200 → LONG_ONLY
- 일봉 종가 < EMA 200 → SHORT_ONLY
- EMA 200 근처 ±0.3% 이내 → NO_TRADE

절대 규칙:

- LONG_ONLY일 때 숏 주문 생성 금지
- SHORT_ONLY일 때 롱 주문 생성 금지
- 방향 필터 위반 시 해당 코인은 INVALID 처리
- 이 줄을 어기는 순간 시스템 무효
- 역방향 거래는 시스템상 불가능하게 막는다

UI 표시:

```txt
Daily Bias: LONG_ONLY / SHORT_ONLY / NO_TRADE / INVALID
```

---

## STEP 2 — 4시간봉 추세 확인

롱 조건:

- 4H EMA 20 > EMA 50
- 최근 구조가 HH / HL

숏 조건:

- 4H EMA 20 < EMA 50
- 최근 구조가 LH / LL

스윙 구조 계산:

- 좌우 2개 캔들보다 고점이 높으면 Swing High
- 좌우 2개 캔들보다 저점이 낮으면 Swing Low
- 최근 Swing High 2개와 Swing Low 2개를 비교한다.

롱:

- 최근 고점 > 이전 고점
- 최근 저점 > 이전 저점

숏:

- 최근 고점 < 이전 고점
- 최근 저점 < 이전 저점

조건 불충족 시:

- 해당 코인은 오늘 패스 처리한다.
- 앱은 차단 사유를 명확히 보여준다.

---

## STEP 3 — 15분봉 진입 트리거

진입은 15분봉에서만 발생한다.

### 롱 진입 조건

#### 1. 동적 지지

- 15m 가격이 EMA 20 또는 EMA 50 근처
- 근처 기준: 가격과 EMA 차이 0.3% 이내

#### 2. 피보나치 되돌림

- 최근 15m 상승파동의 Swing Low → Swing High 기준
- 현재가가 0.5 ~ 0.618 되돌림 구간 안에 있어야 한다.

#### 3. RSI 리셋

- RSI(14)가 40~50 구간까지 내려온 뒤
- 직전 RSI보다 현재 RSI가 상승 전환해야 한다.

#### 4. 확인봉

- 반등 양봉
- 종가 > 시가
- 아래꼬리가 몸통보다 길거나
- Bullish Engulfing
- 현재 거래량 > 최근 20개 평균 거래량

롱 진입은 1, 2, 3이 같은 가격대에서 겹치고 4가 발생할 때만 가능하다.  
따로 노는 신호는 무시한다.

### 숏 진입 조건

#### 1. 동적 저항

- 15m 가격이 EMA 20 또는 EMA 50 근처
- 근처 기준: 가격과 EMA 차이 0.3% 이내

#### 2. 피보나치 되돌림

- 최근 15m 하락파동의 Swing High → Swing Low 기준
- 현재가가 0.5 ~ 0.618 되돌림 구간 안에 있어야 한다.

#### 3. RSI 리셋

- RSI(14)가 50~60 구간까지 올라온 뒤
- 직전 RSI보다 현재 RSI가 하락 전환해야 한다.

#### 4. 확인봉

- 반락 음봉
- 종가 < 시가
- 윗꼬리가 몸통보다 길거나
- Bearish Engulfing
- 현재 거래량 > 최근 20개 평균 거래량

숏 진입도 1, 2, 3이 같은 가격대에서 겹치고 4가 발생할 때만 가능하다.

---

## STEP 4 — 청산 규칙

진입과 동시에 손절/익절을 미리 설정한다.

### 롱 청산

- 손절가 = 최근 눌림목 Swing Low 바로 아래
- 손절 버퍼 = 0.1%
- 1차 익절 = 직전 Swing High
- 1차 익절 도달 시 포지션 50% 청산
- 나머지 50%는 4H 종가가 EMA 20 아래로 마감하면 청산

### 숏 청산

- 손절가 = 최근 되돌림 Swing High 바로 위
- 손절 버퍼 = 0.1%
- 1차 익절 = 직전 Swing Low
- 1차 익절 도달 시 포지션 50% 청산
- 나머지 50%는 4H 종가가 EMA 20 위로 마감하면 청산

최소 손익비:

- 1차 익절 기준 R:R 1:2.5 이상만 진입
- 1:2.5 미만이면 진입 금지

---

## 3. Crypto 전용 Funding Rate 필터

진입 직전 Funding Rate를 확인한다.

롱:

- Funding Rate >= +0.1% 이면 진입 보류

숏:

- Funding Rate <= -0.1% 이면 진입 보류

UI 표시:

```txt
Normal
Crowded Long
Crowded Short
Entry Blocked by Funding
```

차단 메시지 예시:

```txt
Funding 과열로 진입 보류
군중 포지션 과열 위험
진입 가능
```

---

## 4. 자금관리 규칙

모든 거래는 계좌 기준 1회 최대 손실 1%로 고정한다.

입력값:

- 계좌 총액
- 진입가
- 손절가
- 리스크 비율, 기본값 1%

계산식:

```txt
허용 손실 = 계좌 총액 × 0.01
손절폭 % = abs(진입가 - 손절가) / 진입가
포지션 크기 = 허용 손실 / 손절폭 %
실효 레버리지 = 포지션 크기 / 계좌 총액
```

예시:

```txt
계좌 10,000,000원
손절폭 2%
허용 손실 100,000원
포지션 크기 5,000,000원
실효 레버리지 5배
```

앱은 레버리지를 먼저 정하지 않는다.  
항상 손절폭과 허용 손실 기준으로 포지션 크기를 역산한다.

추가 안전장치:

- 실효 레버리지 10배 초과 시 경고
- 실효 레버리지 20배 초과 시 주문 차단
- 하루 누적 손실 -3% 도달 시 신규 진입 중단
- 하루 연속 손실 3회 발생 시 신규 진입 중단
- 동일 심볼 중복 포지션 금지
- 방향 필터 위반 주문 금지
- 최소 손익비 미달 주문 금지
- Funding 과열 주문 금지

---

## 5. Live Trading 기능

Live Trading은 Binance Futures API로 구현한다.

필요 기능:

1. 계좌 잔고 조회
2. 현재 포지션 조회
3. 레버리지 설정
4. 시장가 진입
5. 지정가 1차 익절 주문
6. Stop Market 손절 주문
7. 포지션 50% 부분청산
8. 나머지 추세 추적 청산
9. 주문 취소
10. 긴급 전체 청산

주문 실행 흐름:

1. 전략 조건 통과
2. 리스크 계산
3. 포지션 크기 계산
4. 거래소 최소 주문 수량 검증
5. 실효 레버리지 계산
6. 안전장치 검증
7. 주문 전 Preview Modal 표시
8. 사용자가 Confirm 클릭
9. 서버에서 Binance 주문 실행
10. 주문 결과 저장
11. 대시보드에 실시간 반영

Live Trading에서 자동 진입은 별도 토글로 둔다.

모드:

- Signal Only
- Paper Trading
- Live Manual Confirm
- Live Auto Trading

기본값:

- Signal Only

Live Auto Trading은 가장 마지막에 구현한다.  
처음에는 Live Manual Confirm까지 완성한다.

---

## 6. 보안

API Key는 서버에서만 사용한다.
클라이언트 번들에 노출되면 안 된다.

사용자별 API 키를 저장할 경우:

- AES-256-GCM 방식으로 암호화한다.
- ENCRYPTION_SECRET 환경변수로 암호화 키를 관리한다.
- DB에는 암호화된 값만 저장한다.

절대 금지:

- API Secret을 브라우저로 전달
- localStorage에 API Secret 저장
- 콘솔에 API Secret 출력
- GitHub에 .env 업로드

API 권한 권장:

- Futures Trading 허용
- Withdraw 권한 비활성화

---

## 7. 데이터베이스 모델

Prisma schema를 작성한다.

필요 모델:

### User

- id
- email
- passwordHash 또는 authProvider
- createdAt

### ApiKey

- id
- userId
- exchange
- encryptedApiKey
- encryptedApiSecret
- createdAt
- updatedAt

### TradeSignal

- id
- userId
- symbol
- direction
- dailyBias
- trendStatus
- entryPrice
- stopLoss
- takeProfit1
- riskReward
- fundingRate
- status
- reason
- createdAt

### Order

- id
- userId
- symbol
- side
- orderType
- quantity
- price
- status
- exchangeOrderId
- mode
- createdAt

### Position

- id
- userId
- symbol
- direction
- entryPrice
- stopLoss
- takeProfit1
- quantity
- remainingQuantity
- status
- realizedPnl
- unrealizedPnl
- createdAt
- closedAt

### RiskState

- id
- userId
- date
- dailyPnl
- consecutiveLosses
- isTradingBlocked

---

## 8. UI / UX 디자인 방향

디자인은 업계 1위급 프리미엄 트레이딩 SaaS처럼 만든다.

스타일:

- 다크모드 기본
- 검정 / 딥네이비 / 그레이 기반
- 포인트 컬러는 네온 블루 또는 에메랄드
- 글래스모피즘 카드
- 부드러운 애니메이션
- 고급 트레이딩 터미널 느낌
- 모바일 반응형
- 데스크톱에서는 전문 트레이더용 대시보드 느낌

참고 느낌:

- TradingView
- Hyperliquid
- Binance Futures
- Linear
- Vercel Dashboard
- Bloomberg Terminal의 정보 밀도

핵심 디자인 원칙:

- 데이터 밀도는 높지만 복잡해 보이지 않게 만든다.
- 위험 상태는 즉시 눈에 띄게 만든다.
- 진입 가능/차단/대기 상태를 색상과 뱃지로 구분한다.
- 실거래 버튼은 의도적으로 무겁고 신중한 UX로 만든다.
- 모바일에서는 Scanner, Risk Guard, Position Panel 중심으로 재배치한다.

---

## 9. 페이지 구성

## 9-1. Landing Page

고급 SaaS 랜딩 페이지를 만든다.

섹션:

- Hero
- 핵심 가치 제안
- 전략 작동 방식
- 리스크 관리 엔진
- Paper Trading
- Live Trading
- Security
- Dashboard Preview
- Pricing Placeholder
- FAQ
- CTA

핵심 문구:

```txt
방향은 일봉이 정하고, 진입은 15분봉이 허락하며, 생존은 리스크 엔진이 통제합니다.
```

랜딩 페이지는 단순 소개가 아니라 사용자가 바로 신뢰를 느끼는 프리미엄 핀테크 SaaS 수준으로 만든다.

---

## 9-2. Dashboard

상단:

- 총 계좌 잔고
- 오늘 손익
- 오늘 리스크 사용량
- 활성 포지션
- 신규 진입 가능 코인 수
- Trading Mode

중앙:

- 심볼별 전략 분석 카드
- Daily Bias
- 4H Trend
- 15m Trigger
- Funding
- Risk Reward
- Entry Status

우측:

- Live Position Panel
- Risk Guard Panel
- Emergency Close Button

하단:

- Trade Log
- Signal History
- Order History

---

## 9-3. Strategy Scanner

사용자가 심볼 리스트를 입력한다.

예:

```txt
BTCUSDT, ETHUSDT, SOLUSDT, XRPUSDT, BNBUSDT
```

스캐너는 각 심볼에 대해 다음 데이터를 가져와 전략 조건을 평가한다.

- 1d candles
- 4h candles
- 15m candles
- funding rate

결과 상태:

- READY_TO_ENTER
- WAITING_PULLBACK
- TREND_INVALID
- DAILY_BIAS_BLOCKED
- FUNDING_BLOCKED
- RR_TOO_LOW
- NO_TRADE

---

## 9-4. Signal Detail Page

각 신호를 클릭하면 상세 분석을 보여준다.

표시:

- 일봉 EMA 200 위치
- 4H EMA 20/50 상태
- HH/HL 또는 LH/LL 구조
- 15m EMA 터치 여부
- 피보나치 구간
- RSI 리셋 여부
- 확인봉 여부
- 거래량 증가 여부
- Funding Rate
- 진입가
- 손절가
- 1차 익절가
- 손익비
- 포지션 크기
- 실효 레버리지
- 주문 가능 여부
- 차단 사유

---

## 9-5. Trading Execution Modal

실거래 주문 전 반드시 Preview Modal을 보여준다.

내용:

- 심볼
- 방향
- 진입 방식
- 진입가 예상
- 손절가
- 1차 익절가
- 포지션 크기
- 실효 레버리지
- 예상 최대 손실
- Funding Rate
- 손익비
- 모드

체크박스:

- 나는 이 거래의 최대 손실을 이해했다.
- 이 거래는 1% 리스크 규칙을 따른다.
- API 키에 출금 권한이 없어야 한다.
- Live Trading 실행을 승인한다.

체크박스를 모두 선택해야 Confirm 버튼이 활성화된다.

---

## 10. 코드 구조

다음 구조로 작성한다.

```txt
app/
  page.tsx
  dashboard/page.tsx
  scanner/page.tsx
  signals/[id]/page.tsx
  settings/page.tsx
  api/
    market/candles/route.ts
    market/funding/route.ts
    strategy/scan/route.ts
    trading/order/route.ts
    trading/close/route.ts
    trading/positions/route.ts
    account/balance/route.ts

components/
  layout/
  dashboard/
  scanner/
  trading/
  charts/
  ui/

lib/
  exchange/
    binance.ts
    types.ts
  indicators/
    ema.ts
    rsi.ts
    atr.ts
    fibonacci.ts
    swing.ts
  strategy/
    dailyBias.ts
    trendCheck.ts
    entryTrigger.ts
    exitRules.ts
    riskManager.ts
    scanner.ts
  trading/
    orderExecutor.ts
    positionManager.ts
    riskGuard.ts
  security/
    encryption.ts
  db.ts
  utils.ts

prisma/
  schema.prisma
```

---

## 11. 필수 구현 함수

다음 함수를 반드시 구현한다.

```ts
calculateEMA(values, period)
calculateRSI(candles, period)
detectSwingHighs(candles)
detectSwingLows(candles)
getDailyBias(dailyCandles)
checkFourHourTrend(fourHourCandles, direction)
calculateFibonacciZone(swingLow, swingHigh, direction)
checkEntryTrigger(fifteenMinuteCandles, direction)
calculateRiskPositionSize(accountBalance, entryPrice, stopLoss, riskPercent)
calculateRiskReward(entryPrice, stopLoss, takeProfit, direction)
checkFundingFilter(fundingRate, direction)
scanSymbol(symbol)
executePaperTrade(signal)
executeLiveTrade(signal, userApiKey)
closePosition(positionId)
emergencyCloseAllPositions()
```

모든 함수는 TypeScript 타입을 명확하게 작성한다.
전략 계산 함수는 UI와 분리된 순수 함수로 작성한다.

---

## 12. Binance API 구현

Binance Futures 엔드포인트를 사용한다.

필요 기능:

- Kline 조회
- Funding Rate 조회
- Account Balance 조회
- Position Risk 조회
- New Order
- Cancel Order
- Leverage 변경
- Stop Market Order
- Take Profit Order

서버 코드에서 HMAC SHA256 서명을 구현한다.

요청 실패 처리:

- Rate Limit
- Invalid API Key
- Insufficient Balance
- Min Notional Error
- Precision Error
- Network Error
- Exchange Maintenance

각 에러는 사용자에게 이해 가능한 메시지로 변환한다.

---

## 13. Risk Guard

Risk Guard를 만든다.

차단 조건:

- 일봉 방향과 반대 주문
- 4H 추세 불일치
- 15m 트리거 불충족
- Funding 과열
- 손익비 1:2.5 미만
- 실효 레버리지 20배 초과
- 하루 손실 -3% 초과
- 연속 손실 3회 이상
- 동일 심볼 포지션 존재
- 거래소 잔고 부족
- API 키 오류
- 주문 최소 수량 미달

Risk Guard가 false를 반환하면 주문 실행 금지.

Risk Guard는 다음 값을 반환한다.

```ts
type RiskGuardResult = {
  allowed: boolean
  severity: 'info' | 'warning' | 'danger'
  reasons: string[]
  blockedBy: string[]
}
```

---

## 14. 백테스트 준비

초기 버전에서 완전한 백테스트 엔진까지는 필수가 아니지만, 구조는 만들어둔다.

```txt
lib/backtest/
  backtestEngine.ts
  metrics.ts
```

향후 계산할 지표:

- 승률
- 평균 손익비
- 최대 낙폭
- Profit Factor
- Expectancy
- 연속 손실
- 월별 수익률
- 심볼별 성과

---

## 15. 품질 기준

코드는 실제 프로덕션 수준으로 작성한다.

필수:

- TypeScript strict mode
- 명확한 타입
- 에러 핸들링
- 로딩 상태
- 빈 상태 UI
- 모바일 반응형
- 서버/클라이언트 분리
- API Secret 노출 방지
- 재사용 가능한 컴포넌트
- 전략 로직과 UI 완전 분리
- 계산 함수는 테스트 가능하게 순수 함수로 작성
- 모든 위험한 주문 로직은 서버에서만 처리
- 주문 실행 전 Preview Modal 필수
- 자동매매 기능은 명시적 토글 없이는 비활성화

---

## 16. 최종 결과물 기준

완성된 앱은 다음이 가능해야 한다.

1. 사용자가 계좌 금액과 심볼 리스트를 입력한다.
2. 앱이 Binance Futures 데이터를 가져온다.
3. 일봉 EMA 200으로 방향을 정한다.
4. 4H 추세를 검증한다.
5. 15m 진입 조건을 검증한다.
6. Funding Rate 필터를 적용한다.
7. 손절가, 익절가, 손익비, 포지션 크기를 계산한다.
8. 진입 가능 여부와 차단 사유를 보여준다.
9. Paper Trading으로 가상 진입한다.
10. Live Trading Mode에서 사용자가 확인하면 실제 주문을 실행한다.
11. 주문과 포지션을 대시보드에서 추적한다.
12. 긴급 전체 청산 버튼을 제공한다.
13. Vercel에 배포 가능하다.

---

# 보강 프롬프트

현재 결과물을 더 고급스럽게 개선하라.

특히 다음을 강화하라.

1. UI를 TradingView, Hyperliquid, Linear, Vercel Dashboard 수준의 프리미엄 다크모드로 개선한다.
2. 전략 분석 결과를 단순 텍스트가 아니라 점수화된 카드 UI로 표시한다.
3. Risk Guard가 왜 주문을 차단했는지 명확히 표시한다.
4. Live Trading 버튼은 위험 확인 Modal 없이는 절대 실행되지 않게 수정한다.
5. API Secret은 서버에서만 사용하고 클라이언트에 절대 노출되지 않게 검증한다.
6. Binance 주문 실패 케이스를 모두 사용자 친화적 메시지로 변환한다.
7. 모바일에서도 대시보드가 깨지지 않게 반응형 개선한다.
8. 모든 계산 함수에 TypeScript 타입을 명확히 추가한다.
9. 빈 데이터, 로딩, 에러 상태 UI를 고급스럽게 구현한다.
10. 실제 프로덕션 SaaS처럼 완성도를 높인다.

---

# 추가 보강 요구사항

아래 사항까지 반영하여 업계 1위급 완성도를 목표로 한다.

## A. 앱 완성도 강화

- 대시보드 첫 화면에서 사용자가 3초 안에 현재 상태를 이해할 수 있게 만든다.
- 각 코인별 상태를 READY / WAIT / BLOCKED / RISK ALERT 로 구분한다.
- 각 상태마다 색상, 아이콘, 짧은 설명을 제공한다.
- 사용자는 “왜 진입 가능한지”보다 “왜 진입하면 안 되는지”를 먼저 볼 수 있어야 한다.

## B. 실거래 안전성 강화

- 실거래 모드는 기본 비활성화한다.
- Live Auto Trading은 별도 잠금 상태로 둔다.
- Live Auto Trading 활성화 전에는 다음 확인을 요구한다.
  - Paper Trading 10회 이상 실행
  - API 키 연결 완료
  - 출금 권한 비활성화 안내 확인
  - Risk Guard 활성화 확인
  - 하루 최대 손실 제한 설정 확인
- Emergency Close 버튼은 항상 접근 가능하게 우측 상단 또는 포지션 패널에 고정한다.

## C. UX 카피 강화

차단 메시지는 단순히 “불가”라고 하지 말고 구체적으로 작성한다.

예시:

```txt
진입 차단: 일봉 방향은 LONG_ONLY인데 현재 신호는 SHORT입니다.
진입 차단: 손익비가 1:1.8로 최소 기준 1:2.5에 미달합니다.
진입 보류: Funding Rate가 +0.124%로 롱 포지션 과열 구간입니다.
진입 대기: 4H 추세는 유효하지만 15m 눌림목 확인봉이 아직 없습니다.
```

## D. 차트 UX 강화

Signal Detail Page에는 차트를 추가한다.

표시할 요소:

- 15m 캔들
- EMA 20
- EMA 50
- Fibonacci 0.5
- Fibonacci 0.618
- Entry Price
- Stop Loss
- Take Profit 1
- 현재가

차트 위에 진입 가능/차단 사유를 뱃지로 표시한다.

## E. 코드 안정성 강화

- 모든 API Route는 try/catch 처리한다.
- 서버 에러는 사용자에게 안전한 메시지만 반환한다.
- 내부 에러 상세는 서버 로그에만 기록한다.
- Binance API 응답 타입을 명확히 정의한다.
- 숫자 계산은 NaN, Infinity 방어 로직을 추가한다.
- 캔들 데이터가 부족하면 전략 계산을 중단하고 명확한 이유를 반환한다.

## F. Vercel 배포 강화

Vercel 배포 기준으로 다음 문서를 포함한다.

- README.md
- 환경변수 설정 방법
- Binance API Key 설정 방법
- PostgreSQL 연결 방법
- Paper Trading 실행 방법
- Live Trading 활성화 방법
- 위험 고지 문구

README에는 다음을 명확히 적는다.

```txt
이 앱은 자동매매 기능을 포함하지만, 기본값은 Signal Only입니다.
Live Trading은 사용자가 직접 활성화해야 하며 모든 거래 책임은 사용자에게 있습니다.
API Key에는 출금 권한을 절대 부여하지 마세요.
```

---

# 최우선 구현 순서

반드시 다음 순서로 구현한다.

1. 프로젝트 기본 구조 생성
2. 프리미엄 다크모드 UI 레이아웃
3. 전략 계산 엔진
4. Binance 데이터 연동
5. 시그널 스캐너
6. 리스크 계산기
7. Risk Guard
8. Paper Trading
9. 주문 Preview Modal
10. Live Manual Confirm Trading
11. 포지션 모니터링
12. Emergency Close
13. Vercel 배포 문서
14. Live Auto Trading 잠금 구조

---

# 매우 중요

단순한 데모 앱처럼 만들지 마라.
진짜 사용자가 돈을 걸고 쓸 수 있는 수준의 안정성, 보안성, UI 퀄리티를 목표로 한다.

하지만 기본값은 반드시 Signal Only 또는 Paper Trading이어야 한다.
Live Trading은 사용자가 명시적으로 켜야 한다.

모든 코드를 실제 파일 기준으로 작성하고, 누락 없이 실행 가능한 Next.js 프로젝트로 완성하라.
