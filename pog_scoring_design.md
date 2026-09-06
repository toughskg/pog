# POG 스코어링 설계서

## 1. 목적

본 문서는 POG 시스템의 스코어링 영역을 정의한다. 스코어링은 진열단위 프로젝트에서 상품 라이브러리를 구성하고, MD가 선택한 지표/기간/가중치를 기준으로 상품별 진열 우선순위 점수를 산출한다.

스코어링 결과는 진열제안의 입력으로 사용되며, 시스템은 최종 판단이 아니라 객관적 기준 초안을 제공한다.

## 2. 업무 흐름

```text
표준진열대장에 상품 등록
→ 진열단위 프로젝트 생성
→ 스코어링 지표/기간/가중치 설정
→ 진열단위 내 표준진열대장 상품 취합
→ 상품 라이브러리 생성
→ 라이브러리 상품 추가/삭제
→ 스코어링 실행
→ 상품별 스코어 갱신
→ 진열제안 입력값 제공
```

## 3. 핵심 개념

| 개념 | 설명 |
|---|---|
| 진열단위 | 여러 타입의 표준진열대장을 포함하는 POG 업무 단위 |
| 프로젝트 | 특정 진열단위에서 상품 구성과 스코어링을 수행하는 작업 단위 |
| 상품 라이브러리 | 프로젝트 스코어링 대상 상품 집합 |
| 지표 정책 | MD가 선택한 지표, 매출 기간, 가중치, 점수화 방식 |
| 스코어링 실행 | 라이브러리 상품의 지표값을 갱신하고 최종 점수를 산출하는 실행 단위 |

## 4. 상품 라이브러리

진열단위에 포함된 모든 표준진열대장의 상품을 취합하여 초기 라이브러리를 생성한다. MD는 라이브러리에 상품을 추가하거나 제외할 수 있다.

### 4.1 project_product_library

| 컬럼 | 설명 |
|---|---|
| library_id | 라이브러리 ID |
| project_id | 프로젝트 ID |
| display_unit_id | 진열단위 ID |
| product_id | 상품 코드 |
| product_name | 상품명 |
| source_type | STANDARD_PLANOGRAM / MANUAL / IMPORT |
| include_yn | 스코어링 포함 여부 |
| created_by | 생성자 |
| created_at | 생성일시 |

## 5. 주별 집계 자료

스코어링은 점포-상품 단위 일별 마감 자료를 주별 집계한 `store_product_weekly_metric`을 기준으로 수행한다. 일별 자료는 전일 마감 기준으로 인터페이스하여 누적 적재하고, 주별 자료는 스코어링 및 시즌 POG 참조 기준 자료로 사용한다.

### 5.1 store_product_daily_metric

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
| rfmp_grade | RFMP 고객등급 |
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

### 5.2 store_product_weekly_metric

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
| rfmp_grade | 주간 RFMP 고객등급 |
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

주별 집계 테이블에는 주간 지표값과 기간 재집계에 필요한 구성값을 함께 저장한다. 상품이익률은 주간 이익률만 저장하지 않고 순매출액과 상품이익액을 함께 보관해야 한다.

## 6. 지표와 가중치

| No | 지표명 | 정의 | 그룹 | 가중치(%) |
|---:|---|---|---|---:|
| 1 | 순매출액 | 세금과 에누리액을 제외한 순수 매출액 | A | 20 |
| 2 | 판매수량 | 상품 판매 수량 | A | 15 |
| 3 | 상품이익률 | 순매출액 대비 상품 이익액 비중 | A | 15 |
| 4 | 행사순매출액 | 진성행사상품 순매출액 | A | 10 |
| 5 | 결품수 | 전산 및 점포에서 품절로 확인되는 상품 수량 | B | 10 |
| 6 | 재고보유일수 | 재고 판매하기까지의 평균 보유 일수 | C | 5 |
| 7 | 폐기수량 | 폐기 등록된 상품의 폐기 처리 수량 | B | 7 |
| 8 | 센터반품수량 | 점포 기준 파트너사 및 센터 반품 수량 | B | 5 |
| 9 | 매출원가 | 매출원가를 구성하는 금액의 총합 | D | 3 |
| 10 | 원거래 반품건수 | 영수증 반품 건수 | B | 3 |
| 11 | 원거래 반품액 | 영수증 반품 금액 | B | 3 |
| 12 | 센터입고 수량 | 파트너사로부터 센터로 입고된 상품 수량 | D | 2 |
| 13 | RFMP 고객 등급 | RFMP 기준 고객 등급 | E | 2 |
| 합계 |  |  |  | 100 |

프로젝트에서 MD는 위 기본 지표 중 사용할 지표를 선택하고, 매출 기간을 주 또는 월 단위로 지정하며, 가중치를 조정할 수 있다.

## 7. 점수 계산 공식

```text
최종 스코어 = Σ(지표별 기준점수 × 지표별 가중치 / 100)
```

각 지표는 그룹 A~E의 방식으로 0~100점 기준점수로 변환된다.

## 8. 그룹별 점수화 방식

### 8.1 그룹 A: 높은 값이 좋은 지표

```text
정규화 점수(%) = (value - min) / (max - min) × 100
```

| 정규화 점수 구간 | 기준점수 |
|---:|---:|
| 80% 이상 | 100 |
| 60% 이상 ~ 80% 미만 | 80 |
| 40% 이상 ~ 60% 미만 | 60 |
| 20% 이상 ~ 40% 미만 | 40 |
| 20% 미만 | 20 |

대상 지표: 순매출액, 판매수량, 상품이익률, 행사순매출액.

### 8.2 그룹 B: 낮은 값이 좋은 지표

```text
역방향 정규화 점수(%) = (max - value) / (max - min) × 100
```

| 역방향 정규화 점수 구간 | 기준점수 |
|---:|---:|
| 80% 이상 | 100 |
| 60% 이상 ~ 80% 미만 | 80 |
| 40% 이상 ~ 60% 미만 | 60 |
| 20% 이상 ~ 40% 미만 | 40 |
| 20% 미만 | 20 |

대상 지표: 결품수, 폐기수량, 센터반품수량, 원거래 반품건수, 원거래 반품액.

### 8.3 그룹 C: 재고보유일수

기간 재고보유일수는 다음 방식으로 계산한다.

```text
기간 재고보유일수 = Σ inventory_holding_days_sum / Σ inventory_holding_days_count
초과일수 = 기간 재고보유일수 - 카테고리별 표준 보유일수
```

| 초과일수 구간 | 기준점수 |
|---|---:|
| 기준일수 이하 | 100 |
| 1일 ~ 15일 | 90 |
| 16일 ~ 20일 | 80 |
| 21일 ~ 35일 | 70 |
| 36일 ~ 60일 | 60 |
| 60일 초과 | 50 |

### 8.4 그룹 D: ABC 누적 구성비

센터입고 수량과 매출원가는 기간 합계값을 내림차순 정렬한 뒤 누적 구성비 기준으로 점수를 부여한다.

| 누적 구성비 | 등급 | 기준점수 |
|---:|---|---:|
| 0% 초과 ~ 70% 이하 | A | 100 |
| 70% 초과 ~ 90% 이하 | B | 80 |
| 90% 초과 ~ 100% 이하 | C | 60 |
| 값 없음 또는 0 | 미분류 | 0 |

### 8.5 그룹 E: RFMP 고객 등급

| RFMP 등급 | 기준점수 |
|---|---:|
| MVG | 100 |
| VIP | 80 |
| GOLD | 60 |
| ACE | 40 |
| 등급 없음 또는 매핑 불가 | 0 |

## 9. 기간 집계 방식

| 유형 | 설명 | 대표 지표 |
|---|---|---|
| SUM | 선택 주차 합계 | 순매출액, 판매수량, 센터입고 수량, 매출원가 |
| RATIO_OF_SUMS | 분자 합계 / 분모 합계 | 상품이익률 |
| SUM_DIVIDE_COUNT | 값 합계 / 측정 건수 합계 | 재고보유일수 |
| AVERAGE | 선택 주차 단순 평균 | 보조 지표 |
| LAST_VALUE | 마지막 주차 값 | RFMP 고객 등급 |

## 10. 시즌 POG 스코어링

시즌 POG는 별도 집계 테이블을 만들지 않고 일반 주별 집계 테이블을 사용한다. 여름/겨울 시즌 POG는 참조 기간 유형을 `SAME_SEASON_LAST_YEAR`로 설정하여 전년 동시즌 주차를 조회한다.

```text
일반 POG = 최근 주차 또는 MD 선택 주차 기준
시즌 POG = 전년 동시즌 주차 기준
```

신상품이나 전년 실적이 없는 상품은 시스템 점수를 강제로 낮게 주지 않고 `미평가` 또는 `MD 검토 필요` 상태로 표시한다. MD 전략은 진열제안 및 수동 조정 단계에서 반영한다.

## 11. 마이그레이션 정책

2027년 2월 1일 오픈 기준 시즌 POG를 정상 제시하려면 최소 과거 1년치 주별 집계 자료가 필요하다.

```text
필수: 2026년 2월 1일 ~ 2027년 1월 31일
권장: 2026년 1월 1일 ~ 2027년 1월 31일
가능하면: 2025년 12월 1일 ~ 2027년 1월 31일
```

필수 이관 대상은 `store_product_weekly_metric`이며, 재계산과 감사가 필요하면 `store_product_daily_metric`도 이관한다.

## 12. 주요 데이터 모델

### 12.1 scoring_project

| 컬럼 | 설명 |
|---|---|
| project_id | 프로젝트 ID |
| display_unit_id | 진열단위 ID |
| project_name | 프로젝트명 |
| status | DRAFT / SCORED / PROPOSED / CONFIRMED / DELETED |
| created_by | 생성자 |
| created_at | 생성일시 |

### 12.2 pog_scoring_policy

| 컬럼 | 설명 |
|---|---|
| policy_id | 정책 ID |
| project_id | 프로젝트 ID |
| reference_period_type | RECENT_WEEKS / MANUAL_WEEK_RANGE / SAME_SEASON_LAST_YEAR |
| period_unit | WEEK / MONTH |
| from_period_id | 시작 기간 |
| to_period_id | 종료 기간 |
| status | DRAFT / ACTIVE / CLOSED |

### 12.3 pog_scoring_policy_metric

| 컬럼 | 설명 |
|---|---|
| policy_id | 정책 ID |
| metric_code | 지표 코드 |
| use_yn | 사용 여부 |
| weight | 가중치 |
| scoring_group | A / B / C / D / E |
| period_aggregation_type | 기간 집계 방식 |
| scoring_method | 점수화 방식 |
| score_rule_set_id | 기준점수 룰셋 ID |
| normalization_scope | 정규화 범위 |

### 12.4 product_metric_score_detail

| 컬럼 | 설명 |
|---|---|
| scoring_run_id | 실행 ID |
| project_id | 프로젝트 ID |
| product_id | 상품 ID |
| metric_code | 지표 코드 |
| period_value | 기간 집계 지표값 |
| scoring_group | 스코어링 그룹 |
| normalized_score | 정규화 점수 |
| base_score | 기준점수 |
| weight | 가중치 |
| weighted_score | 가중 점수 |

### 12.5 score_rule_set

A~E 그룹별 기준점수 룰셋 마스터.

| 컬럼 | 설명 |
|---|---|
| score_rule_set_id | 룰셋 ID |
| rule_set_name | 룰셋명 |
| scoring_group | A / B / C / D / E |
| scoring_method | 점수화 방식 |
| enabled | 사용 여부 |
| created_by | 생성자 |
| created_at | 생성일시 |

### 12.6 score_rule_detail

| 컬럼 | 설명 |
|---|---|
| score_rule_set_id | 룰셋 ID |
| rule_order | 적용 순서 |
| from_value | 구간 시작값 |
| to_value | 구간 종료값 |
| grade_code | 등급 코드 |
| base_score | 기준점수 |
| description | 설명 |

그룹별 `from_value`, `to_value`의 의미는 다르다.

| 그룹 | 구간값 의미 |
|---|---|
| A | 정방향 정규화 점수 구간 |
| B | 역방향 정규화 점수 구간 |
| C | 카테고리 기준보유일수 대비 초과일수 구간 |
| D | 누적 구성비 구간 |
| E | RFMP 확정 등급 |

### 12.7 category_inventory_standard

재고보유일수 기준점수 산출을 위한 카테고리별 표준 보유일수.

| 컬럼 | 설명 |
|---|---|
| category_id | 카테고리 ID |
| shelf_type_id | 진열대 유형 ID |
| standard_holding_days | 표준 보유일수 |
| effective_start_date | 적용 시작일 |
| effective_end_date | 적용 종료일 |
| enabled | 사용 여부 |
| created_by | 생성자 |
| created_at | 생성일시 |

### 12.8 scoring_run

스코어링 실행 이력.

| 컬럼 | 설명 |
|---|---|
| scoring_run_id | 실행 ID |
| project_id | 프로젝트 ID |
| policy_id | 정책 ID |
| library_id | 라이브러리 ID |
| status | READY / RUNNING / SUCCESS / FAILED |
| started_by | 실행자 |
| started_at | 시작일시 |
| ended_at | 종료일시 |
| message | 실행 메시지 |

## 13. 배치 처리

### 13.1 일 배치

매일 영업 종료 후 전일 마감 기준 점포-상품 일별 집계 자료를 인터페이스하여 적재한다.

처리 순서:

1. 전일 마감 인터페이스 파일 또는 API 수신
2. staging 테이블 적재
3. 필수값, 숫자값, 코드값 검증
4. 오류 데이터 분리 및 배치 이력 저장
5. 점포-상품 일별 확정 테이블 upsert
6. 적재 완료 이력 저장

### 13.2 주 배치

주별 집계 기준일에 점포-상품 일별 자료를 주차 단위로 집계한다.

처리 순서:

1. 집계 대상 주차 산정
2. 해당 주차의 일별 자료 조회
3. 점포-상품 단위 금액, 수량, 건수 합계 생성
4. 비율 지표 계산에 필요한 분자, 분모 구성값 저장
5. 일수 지표 계산에 필요한 합계 및 측정 건수 저장
6. 점포-상품 주별 집계 테이블 upsert
7. 집계 완료 이력 저장

### 13.3 스코어링 실행

프로젝트의 스코어링 생성 버튼을 클릭하면 시스템은 라이브러리 상품을 대상으로 지표값을 갱신한다.

처리 순서:

1. 프로젝트 및 라이브러리 조회
2. 사용 지표, 기간, 가중치, 룰셋 검증
3. 주별 집계 자료 조회
4. 지표별 기간값 생성
5. 비교군별 정규화 또는 룰셋 점수화
6. 가중치 적용
7. 상품별 최종 스코어 저장
8. 진열제안 입력값 갱신

## 14. 주요 API

| API | 설명 |
|---|---|
| `POST /api/v1/pog/projects` | 프로젝트 생성 |
| `POST /api/v1/pog/projects/{projectId}/library/build` | 표준진열대장 상품 취합 후 라이브러리 생성 |
| `PUT /api/v1/pog/projects/{projectId}/library/items` | 라이브러리 상품 추가/삭제 |
| `PUT /api/v1/pog/projects/{projectId}/scoring-policy` | 스코어링 지표/기간/가중치 설정 |
| `POST /api/v1/pog/projects/{projectId}/score` | 스코어링 실행 |
| `GET /api/v1/pog/projects/{projectId}/scores` | 스코어 결과 조회 |
