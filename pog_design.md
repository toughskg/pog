# POG 진열 스코어링 서버 엔진 설계서

## 1. 목적

POG 시스템에서 상품별 진열 우선순위를 자동 산정하기 위해 판매성, 수익성, 고객가치, 공급안정성, 프로모션, 재고효율, 운영품질, 규모지표를 종합하여 상품별 진열 스코어를 계산하는 서버 엔진을 개발한다.

엔진은 점포-상품 단위의 일별 마감 자료를 입력받아 주별 집계 자료를 생성하고, MD가 설정한 진열대 유형별 지표, 가중치, 참조 주차 범위를 기준으로 표준 POG와 점포별 POG 스코어링을 수행한다. 이후 MD 수동 조정을 거쳐 최종 표준 POG 및 점포별 POG를 생성한다.

본 설계의 핵심 대상은 다음 두 가지이다.

- 전점 기준 표준 POG 스코어링
- 표준 POG 상품을 기준으로 한 점포별 POG 스코어링

## 2. 스코어링 기준

| 평가영역 | 지표 | 가중치(%) | 비고 |
|---|---|---:|---|
| 판매성 | 순매출액 | 20 | 가장 중요한 진열 우선순위 지표 |
| 판매성 | 판매수량 | 15 | 구매빈도 및 수요 반영 |
| 수익성 | 상품이익률 | 15 | 진열 효율 및 수익성 확보 |
| 고객가치 | RFM 고객등급 | 10 | 충성고객 선호도 반영 |
| 공급안정성 | 센터입고 수량 | 10 | 공급 지속 가능성 |
| 프로모션 | 행사순매출액 | 5 | 행사 민감도 반영 |
| 재고효율 | 재고보유일수 | 7 | 재고 회전성 평가 |
| 운영품질 | 결품수 | 5 | 결품이 적을수록 고득점 |
| 운영품질 | 폐기수량 | 3 | 폐기 적을수록 고득점 |
| 운영품질 | 센터반품수량 | 3 | 반품 적을수록 고득점 |
| 운영품질 | 원거래 반품건수 | 3 | 고객 불만족 지표 |
| 운영품질 | 원거래 반품액 | 2 | 반품 영향도 |
| 규모지표 | 매출원가 | 2 | 상품 규모 참고지표 |
| 합계 |  | 100 |  |

## 3. 핵심 계산 방식

### 3.1 기본 공식

```text
최종 진열 스코어 = Σ(지표별 정규화 점수 × 지표별 가중치)
```

각 지표는 0~100점으로 정규화한 뒤 가중치를 적용한다.

```text
가중 점수 = 정규화 점수 × 가중치 / 100
```

최종 점수는 0~100 범위로 산출한다.

### 3.2 POG 생성 단계

POG 생성은 표준 POG와 점포별 POG를 분리하여 처리한다.

```text
점포-상품 일별 마감 자료
→ 점포-상품 주별 집계 자료
→ MD 표준 POG 스코어링 정책 설정
→ 전점 기준 표준 POG 스코어링
→ MD 수동 조정
→ 최종 표준 POG 확정
→ MD 점포별 POG 생성 요청
→ 점포별 POG 스코어링 정책 적용
→ 점포별 POG 생성
```

표준 POG는 전점 기준의 상품 경쟁력을 평가하는 목적이고, 점포별 POG는 확정된 표준 POG 상품 목록 안에서 점포별 실적 특성을 반영하여 진열 우선순위를 재산정하는 목적이다.

### 3.3 주차 기반 참조 기간

MD는 스코어링 정책을 설정할 때 참조 기간을 주차 단위로 선택한다.

예시:

```text
참조 시작 주차: 2026-W31
참조 종료 주차: 2026-W34
```

시스템은 선택된 주차 범위의 주별 집계 자료를 조회하고, 지표별 기간 집계 방식에 따라 스코어링 입력값을 생성한다.

## 4. 지표 방향성

모든 지표가 높을수록 좋은 것은 아니므로 지표별 방향성을 분리한다.

| 지표 | 방향성 | 설명 |
|---|---|---|
| 순매출액 | 높을수록 좋음 | 매출 기여도 |
| 판매수량 | 높을수록 좋음 | 수요 및 구매빈도 |
| 상품이익률 | 높을수록 좋음 | 수익성 |
| RFM 고객등급 | 높을수록 좋음 | 우수 고객 선호도 |
| 센터입고 수량 | 높을수록 좋음 | 공급 지속 가능성 |
| 행사순매출액 | 높을수록 좋음 | 프로모션 반응도 |
| 재고보유일수 | 낮을수록 좋음 | 재고 회전성 |
| 결품수 | 낮을수록 좋음 | 운영 안정성 |
| 폐기수량 | 낮을수록 좋음 | 손실 최소화 |
| 센터반품수량 | 낮을수록 좋음 | 반품 리스크 |
| 원거래 반품건수 | 낮을수록 좋음 | 고객 불만족 지표 |
| 원거래 반품액 | 낮을수록 좋음 | 반품 영향도 |
| 매출원가 | 정책 선택 | 상품 규모 참고지표 |

매출원가는 상품 규모를 나타내는 참고지표이므로 기본 정책은 "높을수록 좋음"으로 두되, 카테고리별 정책 설정이 가능하도록 설계한다.

## 5. 정규화 방식

### 5.1 높을수록 좋은 지표

```text
score = (value - min) / (max - min) × 100
```

### 5.2 낮을수록 좋은 지표

```text
score = (max - value) / (max - min) × 100
```

### 5.3 예외 처리

| 상황 | 처리 |
|---|---|
| max = min | 기본 50점 부여 |
| 값 없음 | 지표별 기본값 또는 제외 정책 적용 |
| 음수 값 | 지표 정의에 따라 0 보정 또는 허용 |
| 이상치 | percentile cap 적용 가능 |
| 신규 상품 | 별도 cold-start 보정 적용 |

## 6. 서버 아키텍처

```mermaid
flowchart TD
    A["전일 마감 인터페이스"] --> B["Daily Metric Staging"]
    B --> C["Daily Snapshot Loader"]
    C --> D["Store-Product Daily Metric"]
    D --> E["Weekly Aggregation Batch"]
    E --> F["Store-Product Weekly Metric"]
    F --> G["Period Metric Aggregator"]
    G --> H["Scoring Engine"]
    I["MD Policy Setting"] --> H
    H --> J["Standard POG Result"]
    J --> K["MD Manual Adjustment"]
    K --> L["Confirmed Standard POG"]
    L --> M["Store POG Generation API"]
    F --> M
    N["Store POG Policy"] --> M
    M --> O["Store POG Result"]
```

## 7. 주요 컴포넌트

### 7.1 Scoring API

스코어링 요청을 수신하고 표준 POG 또는 점포별 POG 계산을 수행한다.

주요 역할:

- 표준 POG 스코어링 실행
- 점포별 POG 스코어링 실행
- 진열대 유형별 정책 조회
- 참조 주차 범위 기준 기간 지표 생성
- 스코어링 결과 및 상세 점수 저장

### 7.2 Validation Layer

입력 데이터의 유효성을 검증한다.

검증 항목:

- 필수 상품 ID 존재 여부
- 지표값 숫자 타입 여부
- 음수 허용 여부
- 기준 기간 유효성
- 카테고리 코드 존재 여부
- 진열대 유형 코드 존재 여부
- 스코어링 정책의 가중치 합계 유효성
- 참조 주차 범위의 주별 집계 자료 존재 여부

### 7.3 Daily Metric Loader

전일 마감 기준으로 인터페이스된 점포-상품 단위 일별 집계 자료를 적재한다.

주요 원칙:

- 일별 자료는 덮어쓰지 않고 기준일 단위로 누적한다.
- 동일 `base_date`, `store_id`, `product_id` 자료가 재수신될 수 있으므로 upsert 방식으로 적재한다.
- 원본 수신 자료는 staging 테이블에 먼저 적재하고, 검증 완료 후 확정 테이블로 반영한다.
- 인터페이스 배치 ID를 저장하여 수신, 검증, 반영 이력을 추적한다.

### 7.4 Weekly Aggregation Batch

일별 자료를 기준으로 점포-상품 단위의 주별 집계 자료를 생성한다.

주별 집계 자료에는 스코어링에 바로 사용할 수 있는 주간 지표값과, 여러 주차를 다시 집계할 때 필요한 원천 구성값을 함께 저장한다. 예를 들어 상품이익률은 주간 이익률만 저장하지 않고 주간 순매출액, 상품이익액도 함께 보관한다.

### 7.5 Period Metric Aggregator

MD가 선택한 참조 주차 범위에 따라 주별 집계 자료를 기간 지표값으로 변환한다.

금액, 수량, 건수 지표는 선택 주차의 합계를 사용하고, 비율 지표는 분자 합계와 분모 합계를 이용해 재계산한다. 일수 및 등급 지표는 지표 정책에 따라 가중 평균, 단순 평균, 마지막 주차 값 중 하나를 사용한다.

### 7.6 Metric Normalizer

원천 지표값을 0~100 점수로 변환한다.

정규화 기준은 다음 단위로 관리할 수 있다.

- 전체 상품 기준
- 카테고리 기준
- 점포 기준
- POG 대상 상품군 기준
- 기간 기준

표준 POG 스코어링은 전점 기준 기간 집계값을 대상으로 정규화하고, 점포별 POG 스코어링은 동일 점포 내 표준 POG 대상 상품을 기준으로 정규화한다.

### 7.7 Weight Engine

정규화된 지표 점수에 가중치를 적용한다.

가중치는 진열대 유형별, 정책 유형별로 MD가 설정한다.

정책 유형:

- `STANDARD`: 전점 기준 표준 POG 생성용 정책
- `STORE`: 점포별 POG 생성용 정책

같은 진열대 유형이라도 표준 POG용 지표와 점포별 POG용 지표를 다르게 설정할 수 있다.

### 7.8 Score Aggregator

지표별 가중 점수를 합산해 최종 진열 스코어를 만든다.

결과 예시:

```json
{
  "productId": "P000123",
  "categoryId": "C010",
  "score": 82.45,
  "rank": 3,
  "grade": "A",
  "calculatedAt": "2026-08-17T10:30:00+09:00"
}
```

### 7.9 Manual Adjustment Manager

표준 POG 스코어링 결과에 대해 MD가 수동으로 순위, 진열 여부, 페이싱 수량 등을 조정할 수 있도록 한다.

시스템 산출값과 MD 조정값은 분리 저장한다. 최종 POG에는 MD 조정이 반영된 최종 순위와 최종 상품 목록을 사용한다.

## 8. API 설계

### 8.1 표준 POG 스코어링 실행

```http
POST /api/v1/pog/standard-pogs/scoring-runs
```

Request:

```json
{
  "shelfTypeId": "SHELF-FROZEN-01",
  "policyId": "POLICY-STD-001",
  "fromWeekId": "2026-W31",
  "toWeekId": "2026-W34",
  "createdBy": "MD001"
}
```

Response:

```json
{
  "standardPogId": "STD-POG-202608-001",
  "shelfTypeId": "SHELF-FROZEN-01",
  "status": "DRAFT",
  "itemCount": 120,
  "calculatedAt": "2026-08-17T10:30:00+09:00"
}
```

### 8.2 표준 POG 수동 조정 저장

```http
PUT /api/v1/pog/standard-pogs/{standardPogId}/items
```

Request:

```json
{
  "items": [
    {
      "productId": "P000123",
      "finalRank": 1,
      "displayYn": true,
      "facingQuantity": 3,
      "adjustReason": "MD 전략상품 우선 진열"
    }
  ]
}
```

### 8.3 표준 POG 확정

```http
POST /api/v1/pog/standard-pogs/{standardPogId}/confirm
```

### 8.4 점포별 POG 생성

```http
POST /api/v1/pog/store-pogs
```

Request:

```json
{
  "storePogName": "2026년 8월 냉동 진열대 점포별 POG",
  "standardPogId": "STD-POG-202608-001",
  "shelfTypeId": "SHELF-FROZEN-01",
  "policyId": "POLICY-STORE-001",
  "fromWeekId": "2026-W31",
  "toWeekId": "2026-W34",
  "storeIds": ["S001", "S002", "S003"],
  "createdBy": "MD001"
}
```

Response:

```json
{
  "storePogBatchId": "STORE-POG-BATCH-202608-001",
  "requestedStoreCount": 3,
  "status": "PROCESSING"
}
```

### 8.5 스코어링 결과 조회

```http
GET /api/v1/pog/scoring-runs/{scoringRunId}/items
```

Response:

```json
{
  "scoringRunId": "RUN-001",
  "items": [
    {
      "rank": 1,
      "productId": "P000001",
      "score": 91.2,
      "metricScores": [
        {
          "metricCode": "netSalesAmount",
          "periodValue": 15200000,
          "normalizedScore": 95.4,
          "weight": 20,
          "weightedScore": 19.08
        }
      ]
    }
  ]
}
```

## 9. 데이터 모델

### 9.1 metric_definition

지표 정의 테이블.

| 컬럼 | 설명 |
|---|---|
| metric_code | 지표 코드 |
| metric_name | 지표명 |
| evaluation_area | 평가영역 |
| default_direction | 기본 방향성 |
| default_period_aggregation_type | 기본 기간 집계 방식 |
| numerator_metric_code | 비율 계산용 분자 지표 코드 |
| denominator_metric_code | 비율 계산용 분모 지표 코드 |
| enabled | 사용 여부 |

### 9.2 store_product_daily_metric

점포-상품 단위 일별 마감 자료.

| 컬럼 | 설명 |
|---|---|
| base_date | 기준일 |
| store_id | 점포 ID |
| product_id | 상품 ID |
| category_id | 카테고리 ID |
| net_sales_amount | 순매출액 |
| sales_quantity | 판매수량 |
| profit_amount | 상품이익액 |
| profit_rate | 상품이익률 |
| rfm_grade | RFM 고객등급 |
| center_inbound_quantity | 센터입고 수량 |
| promotion_net_sales_amount | 행사순매출액 |
| inventory_holding_days | 재고보유일수 |
| stockout_count | 결품수 |
| disposal_quantity | 폐기수량 |
| center_return_quantity | 센터반품수량 |
| original_transaction_return_count | 원거래 반품건수 |
| original_transaction_return_amount | 원거래 반품액 |
| cost_of_sales | 매출원가 |
| interface_batch_id | 인터페이스 배치 ID |
| created_at | 생성일시 |
| updated_at | 수정일시 |

기본키는 `base_date`, `store_id`, `product_id` 조합을 권장한다.

### 9.3 store_product_weekly_metric

점포-상품 단위 주별 집계 자료.

| 컬럼 | 설명 |
|---|---|
| week_id | 주차 ID |
| week_start_date | 주 시작일 |
| week_end_date | 주 종료일 |
| store_id | 점포 ID |
| product_id | 상품 ID |
| category_id | 카테고리 ID |
| net_sales_amount | 주간 순매출액 합계 |
| sales_quantity | 주간 판매수량 합계 |
| profit_amount | 주간 상품이익액 합계 |
| profit_rate | 주간 상품이익률 |
| rfm_grade | 주간 RFM 고객등급 |
| center_inbound_quantity | 주간 센터입고 수량 합계 |
| promotion_net_sales_amount | 주간 행사순매출액 합계 |
| inventory_holding_days_sum | 재고보유일수 합계 |
| inventory_holding_days_count | 재고보유일수 측정 건수 |
| inventory_holding_days_avg | 주간 재고보유일수 평균 |
| stockout_count | 주간 결품수 합계 |
| disposal_quantity | 주간 폐기수량 합계 |
| center_return_quantity | 주간 센터반품수량 합계 |
| original_transaction_return_count | 주간 원거래 반품건수 합계 |
| original_transaction_return_amount | 주간 원거래 반품액 합계 |
| cost_of_sales | 주간 매출원가 합계 |
| aggregated_at | 집계일시 |

주별 집계 테이블에는 주간 지표값과 기간 재집계에 필요한 구성값을 함께 저장한다.

### 9.4 pog_scoring_policy

MD가 설정하는 스코어링 정책 마스터.

| 컬럼 | 설명 |
|---|---|
| policy_id | 정책 ID |
| policy_name | 정책명 |
| policy_type | STANDARD / STORE |
| shelf_type_id | 진열대 유형 ID |
| from_week_id | 참조 시작 주차 |
| to_week_id | 참조 종료 주차 |
| status | DRAFT / ACTIVE / CLOSED |
| created_by | 생성 MD |
| created_at | 생성일시 |
| updated_at | 수정일시 |

### 9.5 pog_scoring_policy_metric

정책별 사용 지표와 가중치.

| 컬럼 | 설명 |
|---|---|
| policy_id | 정책 ID |
| metric_code | 지표 코드 |
| use_yn | 사용 여부 |
| weight | 가중치 |
| direction | HIGHER_IS_BETTER / LOWER_IS_BETTER |
| period_aggregation_type | 기간 집계 방식 |
| normalization_scope | 정규화 범위 |
| display_order | 화면 표시 순서 |

가중치 합계는 사용 지표 기준으로 100이 되도록 검증한다.

### 9.6 standard_pog_result

표준 POG 생성 결과 마스터.

| 컬럼 | 설명 |
|---|---|
| standard_pog_id | 표준 POG ID |
| shelf_type_id | 진열대 유형 ID |
| policy_id | 사용한 표준 POG 정책 |
| from_week_id | 참조 시작 주차 |
| to_week_id | 참조 종료 주차 |
| status | DRAFT / CONFIRMED |
| created_by | 생성 MD |
| confirmed_at | 확정일시 |

### 9.7 standard_pog_result_item

표준 POG 상품별 결과.

| 컬럼 | 설명 |
|---|---|
| standard_pog_id | 표준 POG ID |
| product_id | 상품 ID |
| system_score | 시스템 산출 점수 |
| system_rank | 시스템 산출 순위 |
| md_rank | MD 조정 순위 |
| final_rank | 최종 순위 |
| display_yn | 진열 여부 |
| facing_quantity | 페이싱 수량 |
| manual_adjust_yn | 수동 조정 여부 |
| adjust_reason | 조정 사유 |

### 9.8 store_pog_result

점포별 POG 생성 결과 마스터.

| 컬럼 | 설명 |
|---|---|
| store_pog_id | 점포별 POG ID |
| store_pog_name | 점포별 POG 생성명 |
| standard_pog_id | 기준 표준 POG ID |
| store_id | 점포 ID |
| shelf_type_id | 진열대 유형 ID |
| policy_id | 사용한 점포별 POG 정책 |
| from_week_id | 참조 시작 주차 |
| to_week_id | 참조 종료 주차 |
| status | CREATED / CONFIRMED |
| created_at | 생성일시 |

### 9.9 store_pog_result_item

점포별 POG 상품별 결과.

| 컬럼 | 설명 |
|---|---|
| store_pog_id | 점포별 POG ID |
| store_id | 점포 ID |
| product_id | 상품 ID |
| standard_rank | 표준 POG 최종 순위 |
| store_score | 점포별 스코어 |
| store_rank | 점포별 산출 순위 |
| final_rank | 최종 진열 순위 |

### 9.10 product_metric_score_detail

지표별 상세 점수.

| 컬럼 | 설명 |
|---|---|
| scoring_run_id | 스코어링 실행 ID |
| product_id | 상품 ID |
| metric_code | 지표 코드 |
| period_value | 기간 집계 지표값 |
| normalized_score | 정규화 점수 |
| weight | 가중치 |
| weighted_score | 가중 점수 |

## 10. 주차 기간 집계 정책

주별 집계 자료 생성 시 지표값을 주 단위로 보관하되, MD가 여러 주차를 참조 기간으로 선택한 경우 모든 지표를 단순 평균하지 않는다.

기본 원칙은 다음과 같다.

```text
금액, 수량, 건수 계열 = 선택 주차 합계
비율 계열 = 분자 합계 / 분모 합계
일수, 등급 계열 = 가중 평균, 단순 평균, 마지막 주차 값 중 정책 선택
```

### 10.1 지표별 권장 기간 집계 방식

| 지표 | 주별 보관 방식 | 기간 선택 시 사용 방식 |
|---|---|---|
| 순매출액 | 주간 합계 | 선택 주차 합계 |
| 판매수량 | 주간 합계 | 선택 주차 합계 |
| 상품이익률 | 주간 이익률, 상품이익액, 순매출액 | 상품이익액 합계 / 순매출액 합계 |
| RFM 고객등급 | 주간 등급 | 마지막 주차 값 또는 가중 평균 |
| 센터입고 수량 | 주간 합계 | 선택 주차 합계 |
| 행사순매출액 | 주간 합계 | 선택 주차 합계 |
| 재고보유일수 | 주간 평균, 측정일수 | 가중 평균 |
| 결품수 | 주간 합계 | 선택 주차 합계 |
| 폐기수량 | 주간 합계 | 선택 주차 합계 |
| 센터반품수량 | 주간 합계 | 선택 주차 합계 |
| 원거래 반품건수 | 주간 합계 | 선택 주차 합계 |
| 원거래 반품액 | 주간 합계 | 선택 주차 합계 |
| 매출원가 | 주간 합계 | 선택 주차 합계 |

### 10.2 비율 지표 재계산 예시

상품이익률은 주차별 이익률의 단순 평균을 사용하지 않고 기간 합산값으로 재계산한다.

```text
기간 상품이익률 = 선택 주차 상품이익액 합계 / 선택 주차 순매출액 합계
```

### 10.3 일수 지표 재계산 예시

재고보유일수는 측정 건수를 고려한 가중 평균을 사용한다.

```text
기간 재고보유일수 =
선택 주차 재고보유일수 합계 / 선택 주차 재고보유일수 측정 건수 합계
```

이 방식은 특정 주차의 데이터 건수가 적거나 결측이 있는 경우에도 기간 지표의 왜곡을 줄일 수 있다.

## 11. 등급 정책

최종 스코어에 따라 진열 추천 등급을 부여한다.

| 점수 | 등급 | 의미 |
|---:|---|---|
| 90 이상 | S | 최우선 진열 |
| 80 이상 | A | 우선 진열 |
| 70 이상 | B | 일반 진열 |
| 60 이상 | C | 제한 진열 |
| 60 미만 | D | 비추천 또는 검토 필요 |

## 12. 배치 처리 설계

### 12.1 일 배치

매일 영업 종료 후 전일 마감 기준 점포-상품 일별 집계 자료를 인터페이스하여 적재한다.

처리 순서:

1. 전일 마감 인터페이스 파일 또는 API 수신
2. staging 테이블 적재
3. 필수값, 숫자값, 코드값 검증
4. 오류 데이터 분리 및 배치 이력 저장
5. 점포-상품 일별 확정 테이블 upsert
6. 적재 완료 이력 저장

### 12.2 주 배치

주별 집계 기준일에 점포-상품 일별 자료를 주차 단위로 집계한다.

처리 순서:

1. 집계 대상 주차 산정
2. 해당 주차의 일별 자료 조회
3. 점포-상품 단위 금액, 수량, 건수 합계 생성
4. 비율 지표 계산에 필요한 분자, 분모 구성값 저장
5. 일수 지표 계산에 필요한 합계 및 측정 건수 저장
6. 점포-상품 주별 집계 테이블 upsert
7. 집계 완료 이력 저장

### 12.3 표준 POG 스코어링 배치

MD가 표준 POG 스코어링을 실행하면 시스템은 선택한 진열대 유형, 정책, 참조 주차 범위로 전점 기준 기간 지표를 생성한다.

처리 순서:

1. 표준 POG 정책 조회
2. 사용 지표 및 가중치 검증
3. 참조 주차 범위의 점포-상품 주별 자료 조회
4. 전점 기준 상품별 기간 지표 생성
5. 사용 지표별 정규화
6. 가중치 적용
7. 상품별 스코어 및 순위 산출
8. 표준 POG 초안 저장

### 12.4 점포별 POG 스코어링 배치

MD가 점포별 POG 생성 버튼을 클릭하면 시스템은 확정된 표준 POG 상품 목록을 기준으로 점포별 스코어링을 수행한다.

처리 순서:

1. 확정 표준 POG 조회
2. 표준 POG 포함 상품 목록 조회
3. 점포별 POG 정책 조회
4. 대상 점포 및 참조 주차 범위 확인
5. 점포별 주별 집계 자료 조회
6. 표준 POG 상품만 대상으로 점포별 기간 지표 생성
7. 점포별 정규화 및 가중치 적용
8. 점포별 스코어 및 순위 산출
9. 점포별 POG 결과 저장

## 13. 운영 설정

가중치와 방향성은 코드에 고정하지 않고 MD가 화면에서 설정할 수 있도록 한다.

운영 설정의 기본 단위는 다음과 같다.

- 진열대 유형
- 정책 유형: 표준 POG / 점포별 POG
- 참조 시작 주차 및 종료 주차
- 사용 지표
- 지표별 가중치
- 지표별 방향성
- 지표별 기간 집계 방식
- 정규화 범위

예시:

```json
{
  "policyType": "STANDARD",
  "shelfTypeId": "SHELF-FROZEN-01",
  "fromWeekId": "2026-W31",
  "toWeekId": "2026-W34",
  "metrics": [
    {
      "code": "netSalesAmount",
      "name": "순매출액",
      "area": "판매성",
      "weight": 20,
      "direction": "HIGHER_IS_BETTER",
      "periodAggregationType": "SUM"
    },
    {
      "code": "inventoryHoldingDays",
      "name": "재고보유일수",
      "area": "재고효율",
      "weight": 7,
      "direction": "LOWER_IS_BETTER",
      "periodAggregationType": "WEIGHTED_AVERAGE"
    }
  ]
}
```

## 14. 기술 권장안

| 영역 | 권장 |
|---|---|
| Backend | Spring Boot, NestJS, FastAPI 중 택1 |
| DB | PostgreSQL |
| Batch | Spring Batch, Airflow, Kubernetes CronJob |
| Cache | Redis |
| Message Queue | Kafka 또는 RabbitMQ |
| API Format | REST 우선, 필요 시 gRPC |
| Observability | Prometheus, Grafana, ELK/OpenSearch |

POG 시스템이 Java 기반이라면 Spring Boot를 우선 추천한다. 데이터 계산 중심의 독립 마이크로서비스라면 FastAPI도 적합하다.

## 15. 비기능 요구사항

| 항목 | 요구사항 |
|---|---|
| 성능 | 카테고리 1개 기준 수만 개 상품 배치 처리 가능 |
| 확장성 | 지표 추가/삭제 시 코드 변경 최소화 |
| 정확성 | 동일 입력에 대해 동일 결과 보장 |
| 감사성 | 지표별 상세 점수 저장 |
| 운영성 | 가중치 변경 이력 관리 |
| 장애 대응 | 배치 실패 시 재시도 및 이전 결과 유지 |
| 보안 | 내부 API 인증 및 접근 제어 적용 |
| 추적성 | 일별, 주별, 기간 지표, 스코어링 정책, MD 조정 이력 추적 |
| 재현성 | 과거 정책과 참조 주차 기준으로 동일 스코어 재현 가능 |

## 16. 향후 확장

- 점포별 POG 스코어 분리
- 시즌성 지표 추가
- 신상품 보정 로직
- 카테고리별 가중치 커스터마이징
- 진열 면적 및 페이싱 수량 추천
- AI 기반 스코어 보정 모델
- 스코어 변화 추이 모니터링
- A/B 테스트 기반 가중치 최적화
- 진열대 유형별 표준 POG 템플릿 관리
- 점포 클러스터별 POG 생성
- MD 수동 조정 패턴 분석

## 17. 요약

이 서버 엔진은 점포-상품 단위 일별 마감 자료를 누적 적재하고, 주별 집계 자료를 생성한 뒤, MD가 설정한 진열대 유형별 지표, 가중치, 참조 주차 범위를 기준으로 표준 POG와 점포별 POG를 생성하는 시스템이다.

핵심 설계 원칙은 다음과 같다.

- 일별 자료는 점포-상품-일자 기준으로 누적 적재한다.
- 주별 자료는 점포-상품-주차 기준으로 집계하여 보관한다.
- 주별 자료에는 기간 재집계를 위한 원천 구성값도 함께 저장한다.
- MD가 선택한 참조 기간에 대해 지표별 집계 방식을 다르게 적용한다.
- 표준 POG와 점포별 POG의 스코어링 정책을 분리한다.
- 지표 정의와 가중치는 진열대 유형별로 설정화한다.
- 높은 값이 좋은 지표와 낮은 값이 좋은 지표를 구분한다.
- 표준 POG는 전점 기준으로 정규화하고, 점포별 POG는 점포별 대상 상품 기준으로 정규화한다.
- 최종 점수뿐 아니라 지표별 상세 점수를 저장한다.
- 시스템 산출 결과와 MD 수동 조정 결과를 분리 저장한다.
