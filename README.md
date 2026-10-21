# Multi-Cloud Incident \& Price Intelligence Pipeline

> GCP · AWS · Azure의 \*\*공개 장애 이력\*\*과 \*\*서울 리전 VM 요금\*\*을 n8n으로 자동 수집·정규화하고,
> 알림 · 주간 리포트 · 대시보드로 연결한 멀티클라우드 모니터링 파이프라인

* **대시보드 (Looker Studio)**: https://datastudio.google.com/s/krz871kIFOY
* **기간**: 2026.10 \~ (매시간 / 매일 자동 실행 중)
* **도구**: n8n (Railway 셀프호스팅) · Google Sheets · Looker Studio · Telegram Bot · Gmail
* **제약 조건**: AI 노드 미사용 — 모든 판단 로직을 규칙 기반(JavaScript Code 노드)으로 구현

!\[대시보드](screenshots/dashboard.png)

\---

## 1\. 왜 만들었나

클라우드를 선택하거나 운영할 때 고객이 가장 먼저 묻는 두 가지는 \*\*"얼마나 안정적인가(가용성)"\*\*와 \*\*"얼마나 드는가(비용)"\*\*입니다.
하지만 이 정보는 3사의 상태 페이지와 가격표에 형식이 제각각으로 흩어져 있어, 사람이 매번 직접 확인하고 비교해야 합니다.

이 프로젝트는 그 확인 작업을 자동화해서,
**① 장애는 생기는 즉시 알리고, ② 요금은 매일 같은 기준으로 비교하고, ③ 일주일 치를 리포트로 요약**합니다.

\---

## 2\. 전체 구조

```mermaid
flowchart LR
  subgraph Sources\[공개 데이터 소스]
    A1\[GCP Status<br/>incidents.json]
    A2\[AWS Health<br/>currentevents]
    A3\[Azure Status<br/>RSS]
    B1\[Azure Retail<br/>Prices API]
    B2\[GCP Cloud Billing<br/>Catalog API]
  end

  subgraph n8n\[n8n on Railway]
    W1\[① 장애 수집기<br/>매시간]
    W2\[② 요금 트래커<br/>매일 09:00]
    W3\[③ 주간 리포트<br/>매주 월 09:00]
    W0\[⚠️ 에러 알림]
  end

  A1 \& A2 \& A3 --> W1
  B1 \& B2 --> W2
  W1 -->|upsert| S1\[(Sheets<br/>incidents)]
  W2 -->|upsert| S2\[(Sheets<br/>prices)]
  S1 \& S2 --> W3
  S1 \& S2 --> D\[Looker Studio<br/>대시보드]
  W1 -->|서울·심각 장애| T\[Telegram]
  W2 -->|요금 변동| T
  W3 -->|HTML 리포트| G\[Gmail]
  W3 -->|요약| T
  W1 \& W2 \& W3 -.실패 시.-> W0 --> T
```

|워크플로우|주기|하는 일|
|-|-|-|
|`00\_error-alert`|실패 시|다른 워크플로우가 실패하면 워크플로우명·노드·에러 내용을 텔레그램으로 전송|
|`01\_multicloud-incident-collector`|매시간|3사 장애 수집 → 공통 스키마로 정규화 → 신규/변경/종료 판정 → 시트 업서트 → 조건부 알림|
|`02\_seoul-vm-price-tracker`|매일 09:00|서울 리전 4 vCPU / 16GB VM 요금 수집 → 전일 대비 변동·최저가 계산 → 시트 업서트 → 변동 시 알림|
|`03\_weekly-report`|매주 월 09:00|지난 7일 장애 + 최신 요금을 집계해 핵심 요약이 포함된 HTML 리포트 발송|

\---

## 3\. 핵심 설계 포인트

### ① 서로 다른 3사 데이터를 하나의 스키마로

3사는 형식이 모두 다릅니다. 각각 전용 Code 노드에서 아래 공통 스키마로 변환한 뒤 합칩니다.

|소스|원본 형식|특이사항|
|-|-|-|
|GCP|JSON (최근 이력 전체)|시작·종료 시각, 심각도, 영향 리전 포함|
|AWS|JSON, **UTF-16 인코딩**|그대로 읽으면 깨져서 바이너리로 받아 직접 디코딩 / **진행 중 장애만** 공개|
|Azure|RSS|장애가 없으면 피드가 비어 있음 / **진행 중 장애만** 공개|

```
incident\_key · provider · service · regions · severity · status · title
started\_at · ended\_at · duration\_min · first\_seen · last\_seen · url
```

### ② "현재 장애만" 주는 소스에서 이력을 직접 만들기

AWS와 Azure는 지금 진행 중인 장애만 보여주기 때문에, 이전 실행 결과와 비교해서
**피드에서 사라진 장애를 '종료'로 기록**하는 방식으로 이력을 쌓았습니다.
단, 해당 제공사 조회 자체가 실패한 경우에는 종료로 잘못 처리하지 않도록 **조회 성공 여부 마커**를 따로 둡니다.

### ③ 알림 피로 줄이기

* **첫 실행에는 알림을 보내지 않음** (과거 이력이 한꺼번에 알림으로 쏟아지는 것 방지)
* 신규 장애 중 **심각도 high/disruption 이거나 서울 리전 관련**인 것만 알림
* 요금은 **전일 대비 실제로 바뀐 경우에만** 알림 → 대부분의 날은 조용한 것이 정상

### ④ 중복 없는 저장 (Upsert)

* 장애: `incident\_key`(제공사:장애ID) 기준으로 갱신 → 같은 장애가 매시간 중복 저장되지 않음
* 요금: `price\_key`(날짜|제공사|인스턴스|과금방식) 기준 → 하루 여러 번 실행돼도 하루 1행

### ⑤ 같은 사양으로 공정하게 비교

요금은 **서울 리전 · Linux · 4 vCPU / 16GB** 기준으로 맞췄습니다.

|제공사|비교 인스턴스|계산 방식|
|-|-|-|
|Azure|D4s\_v5 (Intel), D4as\_v5 (AMD)|Retail Prices API의 인스턴스 시간당 정가|
|GCP|n2-standard-4, e2-standard-4|vCPU 단가 × 4 + 메모리(GiB) 단가 × 16|

온디맨드와 스팟을 각각 수집하고, 월 환산은 730시간 기준입니다.

### ⑥ 실패해도 멈추지 않고, 실패하면 알려주기

* 데이터 소스 하나가 실패해도 나머지 2사는 정상 처리 (`Continue on Fail`)
* 워크플로우가 실패하면 **Error Workflow**가 텔레그램으로 즉시 알림
* API 키·토큰은 n8n Credential로 분리해, 워크플로우 JSON을 공개해도 비밀값이 노출되지 않음

\---

## 4\. 결과 및 인사이트

> 서울 리전 · Linux · 4 vCPU / 16GB · 2026년 10월 기준 공개 정가

|구분|내용|
|-|-|
|💰 **스팟 vs 온디맨드**|같은 인스턴스(Azure D4as\_v5)도 스팟은 월 **$28.6**, 온디맨드는 **$154.76** → 약 **81% 절감**|
|💰 **온디맨드 최저가**|GCP **e2-standard-4** 월 $125.51 — 최고가인 GCP n2-standard-4($182.09)보다 약 **31% 저렴**|
|💰 **CPU 계열 선택**|Azure는 AMD 계열 D4as\_v5($154.76)가 Intel 계열 D4s\_v5($172.28)보다 약 **10% 저렴**|
|📉 **스팟 요금 변동**|GCP 스팟 요금이 10/13, 10/20 두 차례 약 **5%씩 인하** — 스팟은 정가와 달리 주기적으로 바뀌어 지속 추적이 필요|
|⚠️ **장애**|수집 기간 중 공개 장애 GCP 6건, AWS 2건, Azure 0건|
|⚠️ **장기 장애**|AWS 2건은 2026년 3월부터 이어진 중동 리전(me-central-1, me-south-1) 장애로, 아직 종료되지 않음|

**시사점**: 같은 사양이라도 *제공사보다 머신 계열·CPU 종류·과금 방식 선택*이 비용에 더 큰 영향을 줍니다.
중단을 허용할 수 있는 배치·개발 워크로드는 스팟으로, 상시 서비스는 비용 효율적인 계열(e2, AMD)부터 검토하는 것이 합리적입니다.

\---

## 5\. 운영하면서 해결한 문제

|문제|원인|해결|
|-|-|-|
|7일 후 구글 시트 연동이 매시간 실패|OAuth 앱이 **테스트 모드**라 갱신 토큰이 7일 만에 만료|에러 알림으로 즉시 감지 → 앱을 프로덕션으로 게시 후 재인증|
|AWS 데이터가 깨져서 파싱 실패|응답이 **UTF-16** 인코딩|바이너리로 받아 BOM을 확인하고 직접 디코딩|
|GCP 요금 조회 시 화면 멈춤|응답이 수만 건 규모로 큼|`fields` 파라미터로 필요한 필드만 요청 + 페이지네이션 적용|
|Azure 요금이 중복으로 저장|같은 SKU에 **Cloud Services** 상품 가격이 섞여 있음|상품명이 `Virtual Machines`인 것만 남기고, 인스턴스×과금방식으로 중복 제거|
|주간 리포트 요금 표에서 Azure가 누락|리포트가 '전체 최신 날짜' 하나만 사용 → 그날 Azure 수집이 실패하자 Azure 행이 빠짐|제공사별로 각자의 최신 날짜를 사용하도록 수정 (v1.1)|
|주간 리포트가 엉뚱한 시간에 발송될 위험|기본 시간대가 America/New\_York|모든 워크플로우와 인스턴스 시간대를 Asia/Seoul로 통일|

\---

## 6\. 한계와 다음 단계

**한계**

* 장애 지속시간은 각 사 상태 페이지에 **게시된 시작\~종료 시각** 기준이며, 실제 서비스 영향 시간과 다를 수 있음
* AWS·Azure는 진행 중 장애만 공개하므로, 수집이 멈춘 사이에 시작하고 끝난 장애는 누락될 수 있음
* 요금은 **공개 정가** 기준이며 약정 할인(RI/CUD)·볼륨 할인은 반영하지 않음
* AWS 요금은 공개 가격 파일이 수 GB 규모라 이번 범위에서 제외

**다음 단계**

* AWS Price List API 연동으로 3사 요금 비교 완성
* 텔레그램 명령어(`/status gcp`)로 현재 상태를 즉시 조회
* AI 노드를 추가해 장애 공지 원문을 한국어로 요약 (규칙 기반 버전과 비교)

\---

## 7\. 폴더 구조 \& 재현 방법

```
├── README.md
├── workflows/
│   ├── 00\_error-alert\_v1.0.json
│   ├── 01\_multicloud-incident-collector\_v1.0.json
│   ├── 02\_seoul-vm-price-tracker\_v1.1.json
│   └── 03\_weekly-report\_v1.1.json
└── screenshots/
    ├── dashboard.png
    ├── 01\_collector.png
    ├── 02\_price-tracker.png
    ├── 03\_weekly-report.png
    ├── executions.png
    ├── telegram-error-alert.png
    ├── weekly-report-email.png
    └── weekly-report-email-2.png
```

1. n8n에서 `workflows/` 폴더의 JSON을 **Import from File**로 불러오기
2. Credential 연결: Google Sheets OAuth2, Gmail OAuth2, Telegram Bot, GCP API Key (Query Auth)
3. 구글 시트에 `incidents`, `prices` 탭을 만들고 시트 노드에 URL 지정
4. 각 워크플로우 Settings에서 Timezone `Asia/Seoul`, Error Workflow `00\_error-alert` 지정 후 Publish

\---

## 8\. 스크린샷

|장애 수집기|요금 트래커|
|-|-|
|!\[](screenshots/01\_collector.png)|!\[](screenshots/02\_price-tracker.png)|

|주간 리포트 워크플로우|주간 리포트 메일|
|-|-|
|!\[](screenshots/03\_weekly-report.png)|!\[](screenshots/weekly-report-email.png)<br>!\[](screenshots/weekly-report-email-2.png)|

|자동 실행 기록|에러 알림 (OAuth 토큰 만료 감지)|
|-|-|
|!\[](screenshots/executions.png)|!\[](screenshots/telegram-error-alert.png)|



