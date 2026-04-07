# Stock Price Widget

현재 웹페이지 위에 여러 주식/코인 시세 위젯을 배치하는 Chrome 확장입니다.

## 현재 상태

시세 공급자 연결은 서버로 이동했고, 확장은 서버 API/실시간 채널만 사용합니다.

- 서버: 검색, 스냅샷 조회, 실시간 구독 fan-out 담당
- 확장: 위젯 렌더링, 사용자 상호작용, 서버 연결 담당
- 남은 일: 배포/운영 안정성 보강

## 기능 (무료)

### 위젯 생성 및 배치
- 팝업에서 주식/코인 모드를 고르고 종목명 또는 티커를 검색
- 검색 결과에서 `생성` 클릭 후 웹페이지에서 `Shift + 드래그`로 위치 지정
- 최대 3개 위젯 생성 가능
- 위젯은 `Shift + 드래그`로 이동
- 우측 핸들은 `Shift + 드래그`로 폭 조절
- 우하단 핸들은 `Shift + 드래그`로 스케일 조절
- 위젯 영역 아래 웹페이지는 그대로 클릭 가능 (클릭 통과)

### 위젯 관리
- 우클릭으로 컨텍스트 메뉴 열기
  - 📌 고정하기 / 고정 해제 (뷰포트에 고정)
  - 🗑 위젯 삭제
- 팝업에서 위젯 목록 확인, 개별/전체 삭제
- 팝업 위치찾기 버튼으로 해당 위젯으로 스크롤 및 강조 표시
- 팝업 눈 버튼으로 위젯 숨기기/표시 전환
- `최소화 모드`: 가격 중심의 간단한 표시로 전환

### 데이터
- 국내주식, 해외주식(NASDAQ/NYSE/AMEX), 코인 실시간 시세
- 가격 변동 색상 표시 (상승/하락)
- 거래소 라벨 표시

## 기능 (PRO)

- 위젯 최대 10개
- 차트 보기 (1분봉, 일봉, 주봉, 월봉)

## 설치

1. Chrome에서 `chrome://extensions/` 열기
2. `개발자 모드` 활성화
3. `압축해제된 확장 프로그램 로드` 클릭
4. `extension/` 폴더 선택

## 데이터 소스

- 코인 검색/시세/실시간: Gate.io
- 국내주식 검색: KRX 상장법인 목록 + 한국투자증권 종목코드 조회
- 국내주식 시세/실시간: 한국투자증권 Open API
- 해외주식(NASDAQ/NYSE/AMEX) 시세: 한국투자증권 Open API

## 서버 설정

- 익스텐션은 `extension/server-config.js`에서 서버 주소를 읽습니다.
- 서버의 KIS 자격증명은 `KIS_APP_KEY`, `KIS_APP_SECRET` 환경변수를 우선 사용합니다.
- 환경변수가 없으면 `kis-config.local.js` 또는 `extension/kis-config.local.js`를 로드합니다.
- Gate.io는 현재 공개 API만 사용하므로 별도 로컬 키가 필요하지 않습니다.
- `kis-config.local.js`, `extension/kis-config.local.js`는 `.gitignore`에 추가되어 Git에 포함되지 않습니다.

### 예제 파일

- `extension/kis-config.example.js`
- `extension/server-config.js`

## 참고

- 국내주식은 종목코드 6자리(`005930`)와 종목명(`삼성전자`) 검색을 지원합니다.
- 코인은 Gate WebSocket 기준으로 실시간 가격을 반영합니다.
- 국내주식은 한국투자증권 WebSocket 기준으로 실시간 가격을 반영합니다.
- 해외주식은 `티커.거래소` 형식으로 검색합니다. (예: `AAPL.NAS`, `TSLA.NAS`, `NVDA.NYS`)
- `chrome://` 같은 브라우저 내부 페이지에서는 동작하지 않습니다.

## 서버 개발 시작

현재는 의존성 없는 최소 서버, WebSocket/SSE fan-out, Gate.io WebSocket, KIS WebSocket 연동까지 구성된 상태입니다.

1. `npm run server`
2. `GET /health`
3. `GET /search?q=삼성&mode=stock`
4. `GET /quotes?symbols=005930,BTC-USDT`
5. `GET /realtime?symbols=005930,BTC-USDT`
6. `WS /ws`
7. `POST /clients/:clientId/subscriptions`
8. 필요하면 `POST /debug/seed`로 테스트 quote 주입

예시:

```bash
curl http://localhost:8787/health
curl 'http://localhost:8787/search?q=BTC&mode=crypto'
curl http://localhost:8787/clients
curl -X POST http://localhost:8787/debug/seed \
  -H 'content-type: application/json' \
  -d '{
    "quotes": [
      {
        "symbol": "005930",
        "shortName": "삼성전자",
        "price": 71200,
        "change": 1200,
        "changePercent": 1.71,
        "currency": "KRW",
        "marketState": "TRADING",
        "exchangeName": "KRX"
      }
    ]
  }'
curl 'http://localhost:8787/quotes?symbols=005930'
```

SSE 실시간 연결 예시:

```bash
curl -N 'http://localhost:8787/realtime?symbols=005930,BTC-USDT'
```

WebSocket 실시간 연결 예시:

```js
const ws = new WebSocket("ws://127.0.0.1:8787/ws");
ws.onopen = () => {
  ws.send(JSON.stringify({ mode: "replace", symbols: ["005930", "BTC-USDT"] }));
};
```

응답 흐름:

- `ready`: 클라이언트 ID 발급
- `snapshot`: 현재 구독 심볼의 최신 스냅샷
- `quote`: 구독 심볼에 대한 업데이트 push
- `heartbeat`: 연결 유지용 이벤트

현재 익스텐션 background는 `/ws`를 통해 서버와 실시간 연결하고, `/quotes`는 초기 스냅샷/수동 갱신용으로 사용합니다. 공급자 API 호출은 서버가 담당합니다.

## 기본 보호장치

- `/search`: IP 기준 분당 30회
- `/quotes`: IP 기준 분당 60회
- WebSocket 메시지: 연결 기준 10초당 20회
- 클라이언트당 최대 구독 심볼 수: 50개

제한값은 [server/config.js](/Users/yoochangheon/stock-price-extension/server/config.js)에서 한 번에 조정할 수 있습니다.

## 서버 주소 설정

- 로컬 기본값은 `http://127.0.0.1:8787`
- Render 배포 시 `extension/server-config.js`의 `baseUrl`을 배포 URL(`https://<service>.onrender.com`)로 바꾸면 됩니다.
