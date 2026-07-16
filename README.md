# 노코드 자동화 도구 비교 분석 보고서: Make vs Zapier

> 작성일: 2026-07-16  
> 작성자: 신의령

---

## 1. 개요

### 1.1 비교 목적

동일한 자동화 워크플로우를 Make와 Zapier에서 각각 구현하여, 과제 수행 당시 무료 플랜 및 무료 체험 환경을 기준으로 기능, 사용성, 분기 방식, 실행량, 디버깅 편의성을 실제 구현 경험에 따라 비교 분석한다.

### 1.2 구현한 워크플로우

**퀴즈 응시 결과 자동 분류 및 알림**

```text
[Trigger] Google Form 제출
    ↓
Google Sheets 응답 탭에 새 행 추가 감지
    ↓
[조건 판단] 점수 ≥ 80 ?
    ├─ Yes → 처리시각 생성·변환
    │        → 합격자 시트에 이름·점수·처리시각 기록
    │        → Discord 합격 알림 전송
    │
    └─ No  → 처리시각 생성·변환
             → 불합격자 시트에 이름·점수·처리시각 기록
             → Discord 불합격 알림 전송
```

- 논리 구성: Trigger 1개 + 조건 분기 1개 + 처리시각 생성·변환 + 결과 기록/알림 Action 4개
- 연동 서비스: Google Forms, Google Sheets, Discord
- 결과 시트 컬럼: `이름 / 점수 / 처리시각`
- Discord 메시지 포함 정보: 이름, 점수, 처리 결과, 처리시각

처리시각은 자동화가 실제로 데이터를 처리한 시점을 기록하기 위해 추가하였다. 특히 Zapier에서는 기본 현재 시각이 UTC로 입력되어 한국 시간과 9시간 차이가 발생했기 때문에, `Formatter by Zapier`를 사용하여 UTC 시간을 `Asia/Seoul`로 변환한 뒤 Google Sheets와 Discord에 전달했다.

---

## 2. 구현 방식 비교

### 2.1 전체 구조 차이

| 항목 | Make | Zapier |
|---|---|---|
| 워크플로우 개수 | 시나리오 1개 | Zap 2개: 합격 처리용, 불합격 처리용 |
| 분기 처리 방식 | Router에서 조건별 경로를 시각적으로 분리 | Filter에서 조건 미충족 시 실행 중단 |
| 무료 환경에서의 우회 방식 | 하나의 시나리오 안에서 합격·불합격 분기 구현 | 합격 Zap과 불합격 Zap을 별도로 생성 |
| 처리시각 반영 | 결과 기록 시 처리시각 값을 함께 매핑 | Formatter로 UTC → Asia/Seoul 변환 후 매핑 |
| 실제 Zapier 스텝 | 해당 없음 | 각 Zap마다 Trigger → Filter → Formatter → Sheets → Discord |

**체감 소감:**  
Make는 Router를 사용해 한 화면에서 합격과 불합격 흐름을 함께 확인할 수 있어 전체 구조 파악과 수정이 편했다. 반면 Zapier는 동일한 로직을 합격용과 불합격용 Zap으로 분리해야 했기 때문에 조건, 시트 탭, Discord 문구를 각각 관리해야 했다. 특히 처리시각 변환 로직도 두 Zap에 동일하게 유지해야 하므로, 이후 수정이 생기면 두 Zap을 함께 점검해야 한다는 번거로움이 있었다.

### 2.2 Zapier의 실제 5단계 구성

#### 합격 처리 Zap

```text
1. Google Sheets — New Spreadsheet Row
2. Filter by Zapier — 점수 (Number) Greater than 79
3. Formatter by Zapier — Current time: UTC (ISO) → Asia/Seoul
4. Google Sheets — 합격자 탭에 새 행 생성
5. Discord — 합격 알림 전송
```

#### 불합격 처리 Zap

```text
1. Google Sheets — New Spreadsheet Row
2. Filter by Zapier — 점수 (Number) Less than 80
3. Formatter by Zapier — Current time: UTC (ISO) → Asia/Seoul
4. Google Sheets — 불합격자 탭에 새 행 생성
5. Discord — 불합격 안내 전송
```

### 2.3 Discord 연동 방식 차이

| 항목 | Make | Zapier |
|---|---|---|
| 사용한 방식 | HTTP 모듈 + Discord Webhook URL | Discord 공식 앱 연동 |
| 설정 방식 | Webhook URL과 JSON Body를 직접 작성 | Discord 봇 권한 승인 후 채널 선택 |
| 설정 난이도 | HTTP 구조와 JSON 형식을 이해해야 함 | 채널과 메시지 필드 중심으로 설정 가능 |
| 유연성 | HTTP 요청을 자유롭게 구성 가능 | 공식 앱 필드 안에서 비교적 간단하게 설정 |
| 보안 주의점 | Webhook URL이 노출되지 않도록 마스킹 필요 | 계정·서버 권한 화면의 개인정보 노출 주의 |

### 2.4 처리시각 구현 차이

이번 구현에서는 단순히 이름과 점수만 저장하지 않고, 자동화가 결과를 처리한 시간을 함께 기록했다.

Zapier에서 처음 `Current time` 값을 Google Sheets에 직접 입력했을 때 한국 시간 오후 4시가 오전 7시로 기록되었다. 이는 기본 시간이 UTC 기준이었기 때문이다. 이를 해결하기 위해 다음 Formatter 설정을 추가했다.

```text
Input: Current time: UTC (ISO)
From Timezone: UTC
To Timezone: Asia/Seoul
To Format: YYYY-MM-DD HH:mm:ss Z
```

변환 결과 예시:

```text
2026-07-16 16:44:52 +0900
```

Formatter의 `Output`을 Google Sheets의 `처리시각` 컬럼과 Discord 메시지의 처리시각 항목에 연결하여 한국 표준시로 기록되도록 수정했다.

### 2.5 스크린샷 증거

캡처 파일은 Make와 Zapier를 합쳐 하나의 연속 번호 체계로 정리하였다. 모든 이미지는 GitHub 저장소의 `images` 폴더에 저장한다.

#### 무료 플랜 및 전체 구조

| 항목 | 이미지 |
|---|---|
| Make 무료 플랜 대시보드 | ![Make 무료 플랜 대시보드](images/01_Make_대시보드_무료플랜.png) |
| Zapier Pro 무료 체험 대시보드 | ![Zapier Pro 무료 체험 대시보드](images/02_Zapier_대시보드_Pro무료체험.png) |
| Make 전체 시나리오와 Router 분기 | ![Make 전체 시나리오](images/03_Make_전체시나리오_Router분기.png) |
| Zapier 합격 처리 Zap 전체 구성 | ![Zapier 합격 Zap](images/14_Zapier_합격Zap_전체구성.png) |
| Zapier 불합격 처리 Zap 전체 구성 | ![Zapier 불합격 Zap](images/20_Zapier_불합격Zap_전체구성.png) |
| Zapier 두 Zap 활성화 목록 | ![Zapier 두 Zap 활성화](images/24_Zapier_두Zap_Live목록.png) |

#### 조건 설정과 시간대 변환

| 항목 | 이미지 |
|---|---|
| Make Router 합격 필터 | ![Make 합격 필터](images/04_Make_Router_합격필터_80점이상.png) |
| Make Router 불합격 필터 | ![Make 불합격 필터](images/05_Make_Router_불합격필터_80점미만.png) |
| Zapier 합격 필터 | ![Zapier 합격 필터](images/16_Zapier_합격필터_80점이상.png) |
| Zapier 시간대 변환 Formatter | ![Zapier 시간대 변환](images/17_Zapier_시간대변환_Formatter설정.png) |
| Zapier 불합격 필터 | ![Zapier 불합격 필터](images/21_Zapier_불합격필터_80점미만.png) |

#### 실행 결과와 로그

| 항목 | 이미지 |
|---|---|
| Make 합격 경로 실행 성공 | ![Make 합격 실행](images/06_Make_합격경로_실행성공.png) |
| Make 불합격 경로 실행 성공 | ![Make 불합격 실행](images/07_Make_불합격경로_실행성공.png) |
| Make 실행 History | ![Make History](images/11_Make_History_전체실행목록.png) |
| Zapier 합격자 시트 결과 | ![Zapier 합격자 결과](images/25_Zapier_최종결과_합격자시트.png) |
| Zapier 불합격자 시트 결과 | ![Zapier 불합격자 결과](images/26_Zapier_최종결과_불합격자시트.png) |
| Zapier Discord 최종 알림 | ![Zapier Discord 알림](images/27_Zapier_최종결과_Discord알림.png) |
| Zapier 실행 History | ![Zapier History](images/28_Zapier_History_전체실행목록.png) |
| Zapier Filtered 상세 로그 | ![Zapier Filtered 로그](images/29_Zapier_History_Filtered_상세로그.png) |
| Zapier Task 사용량 | ![Zapier Task 사용량](images/30_Zapier_TaskUsage_사용량현황.png) |

> 모든 스크린샷에서 Webhook URL, 이메일 주소, 계정 정보 등 민감정보는 마스킹하거나 화면에서 제외하였다.

---

## 3. 사용성 비교

### 3.1 UI/UX

| 항목 | Make | Zapier |
|---|---|---|
| 편집 방식 | 캔버스 기반으로 모듈과 Router를 자유롭게 배치 | 위에서 아래로 이어지는 세로 스텝 방식 |
| 전체 흐름 파악 | 분기 구조가 한 화면에 보여 직관적 | 각 Zap은 단순하지만 전체 분기를 보려면 두 Zap을 함께 확인해야 함 |
| 초보자 접근성 | 처음에는 용어와 데이터 번들 구조가 낯설 수 있음 | Setup → Configure → Test 순서가 명확함 |
| 데이터 매핑 | 모듈 출력 필드를 선택해 연결 | 이전 Step 데이터와 Variables를 검색해 삽입 |
| 테스트 방식 | `Run once`로 시나리오 전체 실행 | 각 Step의 `Test step`을 순차적으로 실행 |
| 처리시각 수정 | 매핑 또는 날짜 함수 활용 | Formatter를 추가해 시간대 변환 |

### 3.2 실행 로그와 디버깅

- **Make:** History에서 실행별로 모듈의 입력과 출력 Bundle을 확인할 수 있어, 어느 모듈에서 값이 잘못 전달됐는지 추적하기 편했다.
- **Zapier:** Zap History에서 `Successful`, `Filtered` 상태가 명확하게 구분됐다. Filtered 실행을 열면 입력 점수와 함께 `This filter successfully stopped your run.` 문구를 확인할 수 있었고, 이후 Formatter·Google Sheets·Discord 단계가 실행되지 않았다는 점도 확인할 수 있었다.
- **처리시각 디버깅:** Zapier에서 처리시각이 9시간 차이 나는 문제를 Formatter 단계의 입력 시간대와 출력 시간대를 명시해 해결했다.

### 3.3 구현 중 겪은 문제와 해결

| 문제 | 원인 | 해결 |
|---|---|---|
| 점수 조건 비교가 잘못될 가능성 | Text 연산자를 사용하면 숫자가 문자열 방식으로 비교될 수 있음 | Filter 조건을 `(Number)` 연산자로 설정 |
| Zapier에서 한국 시간보다 9시간 이전으로 기록됨 | `Current time`이 UTC 기준으로 처리됨 | Formatter에서 UTC → Asia/Seoul로 변환 |
| Zapier에서 하나의 Zap으로 양쪽 분기를 구현하기 어려움 | Paths가 무료 환경에서 제한됨 | 합격 Zap과 불합격 Zap을 별도로 생성 |
| Zap 복제 후 합격 설정이 남아 있음 | Duplicate 시 기존 Filter, Worksheet, Discord 문구가 그대로 복제됨 | 불합격 Zap에서 Filter, Worksheet, Discord 문구를 각각 수정 |
| Worksheet 변경 후 매핑 초기화 가능성 | 시트 탭 변경 시 필드 구조를 다시 불러옴 | 이름, 점수, 처리시각 매핑을 다시 확인 |
| 테스트 데이터가 조건에 맞지 않아 Filter 테스트 실패 | 합격용 95점 데이터가 불합격 Zap에 남아 있었음 | Trigger 테스트 레코드를 80점 미만 데이터로 교체 |

---

## 4. 무료 플랜 및 실행량 비교

> 아래 내용은 과제 수행 당시 확인한 플랜 화면과 실제 실행 결과를 기준으로 작성했다.

| 항목 | Make Free | Zapier 무료 환경/체험 환경 |
|---|---|---|
| 월 실행 한도 | 1,000 Operations | Free 기준 100 Tasks |
| 과금 단위 | 모듈 1회 실행마다 Operation 사용 | 성공적으로 실행된 Action마다 Task 사용 |
| Trigger·조건 처리 | 실행된 모듈 기준으로 Operation 반영 | Trigger와 Filter는 Billable Task에서 제외 |
| 처리시각 변환 | 구현 방식에 따라 Operation 영향 가능 | Formatter 단계는 존재하지만 실제 성공 실행은 2 Tasks로 기록됨 |
| 이번 프로젝트 성공 1건 | 시나리오의 실제 실행 모듈 수에 따라 계산 | Google Sheets 기록 1 + Discord 전송 1 = 2 Tasks |
| Filtered 실행 | Router/필터까지 실행된 모듈 기준 사용 | 후속 Action이 실행되지 않아 0 Tasks |
| 분기 구현 | Router 사용 가능 | Paths 대신 Zap 2개로 우회 |
| HTTP/Webhook | HTTP 모듈 활용 가능 | Webhooks by Zapier는 프리미엄 앱 |
| 최소 폴링 간격 | 15분 | 15분 |

### 4.1 실제 Zapier Task 사용량

최종 테스트에서는 합격 응답 1건과 불합격 응답 1건을 처리했다.

```text
합격 처리 Zap 성공: 2 Tasks
불합격 처리 Zap 성공: 2 Tasks
반대 조건의 Filtered 실행: 각각 0 Tasks
총 Billable Zap Tasks: 4
```

Zapier Task Usage 화면에서도 두 Zap이 각각 2 Tasks를 사용하여 총 4 Billable Tasks가 발생한 것으로 확인됐다.

### 4.2 실행량 분석

이번 워크플로우에서 Zapier는 성공한 응답 1건당 2 Tasks를 사용했다. 따라서 월 100 Tasks를 단순 적용하면 약 50건의 최종 처리까지 가능하다. 반면 Make는 모듈 단위로 Operation이 계산되므로 시나리오가 복잡해질수록 한 건당 소모량이 증가할 수 있지만, 월 제공량 자체는 더 크다.

단순한 실행 횟수만 보면 Make가 여유롭지만, 실제 효율은 시나리오에서 몇 개의 모듈이 실행되는지와 Router 이후 어느 경로가 동작하는지까지 함께 고려해야 한다.

---

## 5. 종합 평가

### 5.1 항목별 평가

| 평가 항목 | Make | Zapier | 비고 |
|---|---:|---:|---|
| 무료 플랜 실용성 | 5.0 | 3.0 | Make는 Router와 HTTP 활용이 가능하고 실행 한도가 큼 |
| 분기/로직 표현력 | 5.0 | 2.5 | Make는 한 시나리오에서 분기, Zapier는 두 Zap으로 분리 |
| 초보자 진입 장벽 | 3.5 | 4.5 | Zapier의 단계형 UI가 처음 사용하기 쉬움 |
| 연동 앱 설정 편의성 | 4.0 | 4.5 | Zapier Discord 공식 앱 설정이 간단했음 |
| 디버깅/로그 | 4.5 | 4.0 | Make는 모듈별 데이터, Zapier는 상태 구분이 명확 |
| 유지보수 편의성 | 4.5 | 3.0 | Zapier는 두 Zap과 Formatter 설정을 함께 관리해야 함 |
| **총점** | **26.5 / 30** | **21.5 / 30** | 이번 과제 기준 |

### 5.2 상황별 추천

- **Make 추천:** 합격/불합격처럼 여러 조건 분기가 필요하거나, HTTP·Webhook을 자유롭게 사용해야 하며, 무료 환경에서 비교적 많은 실행량이 필요한 경우
- **Zapier 추천:** 로직이 단순하고, 공식 앱 연결을 이용해 빠르게 자동화를 만들고 싶은 경우
- **시간대 처리가 중요한 경우:** 어느 도구를 사용하든 서비스별 기본 시간대를 반드시 확인해야 하며, 기록 결과가 UTC로 저장되는지 테스트해야 한다.

### 5.3 결론

이번 미션에서는 무료 환경에서 Router와 HTTP 기능을 활용해 하나의 시나리오로 전체 흐름을 구성할 수 있었던 Make가 더 적합했다. 합격과 불합격 경로를 한 화면에서 확인할 수 있어 구조 이해와 유지보수가 편했고, 실행 한도도 상대적으로 여유로웠다.

Zapier는 Setup → Configure → Test로 이어지는 단계형 UI와 Discord 공식 앱 연동이 직관적이라는 장점이 있었다. 그러나 무료 환경에서는 합격과 불합격 분기를 위해 두 개의 Zap을 별도로 관리해야 했고, 처리시각의 UTC 문제를 해결하기 위해 각 Zap에 Formatter 단계를 추가해야 했다. 그 결과 각 Zap은 `Trigger → Filter → Formatter → Google Sheets → Discord`의 5단계 구조가 되었다.

최종적으로 이번 과제처럼 조건 분기와 Webhook 활용이 중요한 워크플로우에는 Make가 더 효율적이었다. 반면 단일 조건과 단순한 공식 앱 연동이 중심인 업무에서는 Zapier가 더 빠르게 구축할 수 있는 도구라고 판단했다.

---

## 6. 부록

### 6.1 캡처 파일 전체 목록

아래 파일명은 Make와 Zapier 전체 캡처를 하나의 연속 번호로 재정리한 최종 이름이다.

| 번호 | 최종 파일명 | 내용 |
|---:|---|---|
| 01 | `01_Make_대시보드_무료플랜.png` | Make 무료 플랜 대시보드 |
| 02 | `02_Zapier_대시보드_Pro무료체험.png` | Zapier Pro 무료 체험 대시보드 |
| 03 | `03_Make_전체시나리오_Router분기.png` | Make 전체 시나리오와 Router 분기 |
| 04 | `04_Make_Router_합격필터_80점이상.png` | Make 합격 경로 필터 |
| 05 | `05_Make_Router_불합격필터_80점미만.png` | Make 불합격 경로 필터 |
| 06 | `06_Make_합격경로_실행성공.png` | Make 합격 경로 실행 성공 |
| 07 | `07_Make_불합격경로_실행성공.png` | Make 불합격 경로 실행 성공 |
| 08 | `08_Make_GoogleSheets_합격자탭_기록결과.png` | Make 합격자 탭 기록 결과 |
| 09 | `09_Make_GoogleSheets_불합격자탭_기록결과.png` | Make 불합격자 탭 기록 결과 |
| 10 | `10_Make_Discord_합격불합격_알림결과.png` | Make Discord 합격·불합격 알림 |
| 11 | `11_Make_History_전체실행목록.png` | Make 실행 History 전체 |
| 12 | `12_Make_History_합격경로_상세로그.png` | Make 합격 경로 상세 로그 |
| 13 | `13_Make_History_불합격경로_상세로그.png` | Make 불합격 경로 상세 로그 |
| 14 | `14_Zapier_합격Zap_전체구성.png` | Zapier 합격 처리 Zap 전체 구성 |
| 15 | `15_Zapier_트리거_GoogleSheets설정.png` | Zapier Google Sheets Trigger 설정 |
| 16 | `16_Zapier_합격필터_80점이상.png` | Zapier 합격 필터 |
| 17 | `17_Zapier_시간대변환_Formatter설정.png` | Zapier UTC → Asia/Seoul 변환 |
| 18 | `18_Zapier_합격자시트_Action매핑.png` | Zapier 합격자 시트 Action 매핑 |
| 19 | `19_Zapier_합격Discord_Action설정.png` | Zapier 합격 Discord Action 설정 |
| 20 | `20_Zapier_불합격Zap_전체구성.png` | Zapier 불합격 처리 Zap 전체 구성 |
| 21 | `21_Zapier_불합격필터_80점미만.png` | Zapier 불합격 필터 |
| 22 | `22_Zapier_불합격자시트_Action매핑.png` | Zapier 불합격자 시트 Action 매핑 |
| 23 | `23_Zapier_불합격Discord_Action설정.png` | Zapier 불합격 Discord Action 설정 |
| 24 | `24_Zapier_두Zap_Live목록.png` | 합격·불합격 Zap 활성화 목록 |
| 25 | `25_Zapier_최종결과_합격자시트.png` | Zapier 합격자 시트 최종 결과 |
| 26 | `26_Zapier_최종결과_불합격자시트.png` | Zapier 불합격자 시트 최종 결과 |
| 27 | `27_Zapier_최종결과_Discord알림.png` | Zapier Discord 합격·불합격 최종 알림 |
| 28 | `28_Zapier_History_전체실행목록.png` | Zapier 성공·Filtered 실행 목록 |
| 29 | `29_Zapier_History_Filtered_상세로그.png` | Zapier Filtered 상세 로그 |
| 30 | `30_Zapier_TaskUsage_사용량현황.png` | Zapier Task 사용량 현황 |

- 프로젝트 2: 채용공고 지원 일정 자동 등록 및 알림 — 별도 문서 참조
- 보너스 과제: AI 연동 또는 실패 알림 — 별도 문서 참조
- 처리시각 표준 형식: `YYYY-MM-DD HH:mm:ss Z`
- Zapier 시간대 변환: `UTC → Asia/Seoul`
