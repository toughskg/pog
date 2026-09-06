# POG 점별진열대장 생성 설계서

## 1. 목적

본 문서는 확정된 표준 프로젝트와 표준진열대장을 기준으로 점포별 진열대장을 생성하는 영역을 정의한다. 점별진열대장 생성은 표준진열대장과 매핑된 점별진열대장을 일괄 생성하거나, 사용자가 action list를 단계별로 실행하며 실시간으로 조정하는 방식으로 수행한다.

## 2. 업무 흐름

```text
프로젝트 확정
→ 표준진열대장 조회
→ 표준/점별 진열대장 매핑 조회
→ Action List 설정
→ 사용자 수동 실행 또는 배치 실행
→ 점포별 특성 반영
→ 점별진열대장 생성
→ 실행 결과 검토
→ 점별진열대장 확정
```

## 3. Action List 개념

Action List는 표준진열대장을 점별진열대장으로 변환하기 위한 실행 옵션 묶음이다. Action List는 사용자가 수동으로 실행할 수 있고, 확정된 프로젝트에 대해 배치로 실행할 수도 있다.

각 action은 단계별로 실행 가능해야 하며, 사용자는 표준진열대장과 점별진열대장이 실시간으로 변경되는 모습을 확인하면서 옵션 값을 수정할 수 있다.

Action List는 JSON 형식의 데이터 구조로 저장한다. 명령의 실행 단위는 `key1`이며, `key1` 하나가 하나의 설정 화면을 구성한다. `key2`, `key3`, `key4`는 해당 `key1` 설정 화면 안에서 관리되는 세부 옵션이다.

명령 순서는 같은 단계 안에서만 변경할 수 있다. 예를 들어 `Fixtures` 단계의 `Mirror` 명령은 `Fixtures` 단계 안에서 순서를 바꿀 수 있지만 `Products Phase`나 `Positions Phase`로 이동할 수 없다. 단, `common` 명령은 전체 단계의 어느 위치에도 삽입할 수 있다.

## 4. Action List 주요 옵션

Action List 옵션은 80여 가지 이상으로 확장될 수 있으므로, 아래 항목은 예시이며 전체 범위를 제한하지 않는다.

| 옵션 영역 | 예시 |
|---|---|
| 점포 특성 | 점포 유형, 권역, 매출 규모, 면적, 객층 |
| 진열 방향 | 좌→우, 우→좌, 상→하, 하→상 |
| 부족 처리 | 상품 축소, 저스코어 상품 제외, 대체 상품 사용, 빈 공간 허용 |
| 초과 처리 | 면수 축소, 깊이 축소, 하위 상품 제외, 보조 진열대 이동 |
| 브랜드 모음 | 동일 브랜드 연속 진열, 브랜드존 고정 |
| 카테고리 그룹 | 소분류별 묶음, 용도별 묶음, 가격대별 묶음 |
| 계약 위치 | 특정 상품/브랜드 위치 고정 |
| 신상품 | 신상품 우선 진열, 최소 면수 보장 |
| 재고/매출 | 매출 비중별 면수 조정, 재고수량 기준 제외 |
| 예외 처리 | 점포별 미취급, 취급중단, 물류 불가 상품 제외 |

## 5. Action List 명령 카탈로그

Action List에서 실행되는 명령은 별도의 명령 카탈로그로 관리한다. 명령 카탈로그는 단계, key 계층, 입력 타입, 기본값, 값 형식, 지정값, 설명, 효과를 가진다.

원본 명령 정의는 다음 구조를 기준으로 수집한다.

| 컬럼 | 설명 |
|---|---|
| 단계 | 명령이 실행되는 최상위 단계 |
| key1 | 1단계 명령 그룹 |
| key2 | 2단계 옵션 |
| key3 | 3단계 옵션 |
| key4 | 4단계 옵션 |
| 타입 | check / text / select / radio / number 등 입력 타입 |
| 기본값 | 옵션 기본값 |
| 값 형식 | %, in/cm, fixed value 등 값의 단위 또는 형식 |
| 지정값 | 선택 가능한 값 |
| 설명 | 옵션 설명 |
| 효과 | 실행 시 POG에 미치는 효과 |

### 5.1 명령 단계

| 단계 | 주요 명령 그룹 |
|---|---|
| setting | fill out, Warnings, Collision |
| Fixtures | Segment association, Mirror, Filter fixtures, Adjust fixtures, Rename fixtures, Combine fixtures, Fixture linkage, Copy fixed signs |
| Products Phase | Include products, Exclude products, Copy merchandised signs |
| Positions Phase | Orientation, Allow secondary orientation, Drop duplicate positions, Group positions along fixture, Sort positions along fixture, Move to same fixture, Move to same fixture (multiple), Split positions across linked fixtures |
| Bands and Families Phase | Define a band, Define multiple bands, Define product families, Preview |
| Reduce to Fit Phase | Drop Positions, Relax minimum unit facings |
| Fill out Phase | Add positions, Boost minimum unit facings, Allow motion, Allow motion (multiple) |
| Final Phase | Duplicate positions vertically to empty shelves, Add side caps, Copy data to target, Set placement |
| common | Obey next action if ..., By pass this action List, .. also where, Comment, Stop |

### 5.2 단계별 역할

| 단계 | 역할 |
|---|---|
| setting | 전체 실행 전 빈 공간, overflow, warning, collision 처리 정책을 설정 |
| Fixtures | 세그먼트, 선반, 집기, 미러링, fixture 필터링과 조정 처리 |
| Products Phase | 대상 상품 포함/제외와 상품 관련 사인 복사 처리 |
| Positions Phase | 상품 방향, 중복 포지션 제거, 위치 정렬, fixture 간 이동 처리 |
| Bands and Families Phase | 밴드, 복수 밴드, 상품 패밀리 정의 및 미리보기 처리 |
| Reduce to Fit Phase | 공간 부족 시 포지션 제거 또는 최소 면수 완화 처리 |
| Fill out Phase | 남는 공간 채우기, 최소 unit facings 보강, 상품 이동 처리 |
| Final Phase | 빈 선반 보완, 사이드캡 추가, 데이터 복사, 최종 배치 보정 |
| common | 조건부 실행, action list 우회, 추가 필터, 주석, 중지 처리 |

### 5.3 명령 처리 원칙

- Action List는 JSON 형식으로 저장하고 실행한다.
- 명령의 실행 단위는 `key1`이다.
- `key1` 한 개는 하나의 설정 화면으로 구성한다.
- `key2`, `key3`, `key4`는 `key1` 화면의 하위 옵션으로 관리한다.
- 명령은 `단계 > key1 > key2 > key3 > key4` 계층으로 관리한다.
- 각 명령 옵션은 입력 타입과 값 형식에 따라 화면 컴포넌트를 자동 결정할 수 있어야 한다.
- 명령 정의는 코드에 고정하지 않고 DB 또는 설정 파일로 관리한다.
- 명령 실행 순서는 `단계`와 단계 내 `key1` 순서를 함께 사용한다.
- 명령 순서 변경은 동일 단계 안에서만 허용한다.
- 일반 명령은 다른 단계로 이동할 수 없다.
- `common` 명령은 예외적으로 전체 단계의 어느 위치에도 삽입할 수 있다.
- 명령별 설명과 효과는 사용자 화면의 도움말로 제공한다.
- 명령 옵션은 예시 목록에 한정하지 않고 확장 가능해야 한다.

### 5.4 Action List JSON 구조

Action List는 다음과 같은 구조를 가진다.

```json
{
  "actionListId": "AL-001",
  "projectId": "PRJ-001",
  "steps": [
    {
      "phaseCode": "Fixtures",
      "stepOrderInPhase": 1,
      "key1": "Mirror",
      "commandDefinitionId": "CMD-FIX-MIRROR",
      "settings": {
        "enabled": true,
        "Mirror": 2,
        "Desired traffic flow formula": "IF(Width=300,1,0)",
        "Reorder segments, but preserve the contents of each segment": false
      }
    },
    {
      "phaseCode": "Products Phase",
      "stepOrderInPhase": 1,
      "key1": "Include products",
      "commandDefinitionId": "CMD-PRD-INCLUDE",
      "settings": {
        "enabled": true,
        "Where": "FILTER_IN(UNIT MOVEMENT > 5)"
      }
    },
    {
      "phaseCode": "common",
      "insertTargetPhaseCode": "Products Phase",
      "insertBeforeStepOrder": 1,
      "key1": "Obey next action if ...",
      "commandDefinitionId": "CMD-COM-OBEY-NEXT",
      "settings": {
        "condition": "COUNT(UPC) > 0"
      }
    }
  ]
}
```

`common` 명령은 논리상 `phaseCode = common`으로 관리하되, 실제 실행 위치는 `insertTargetPhaseCode`, `insertBeforeStepOrder`, `insertAfterStepOrder` 등으로 지정한다.

## 6. 사용자 수식

Action List는 사용자 수식을 지원한다. 사용자 수식은 DB 집계 함수에 준하는 조건, 그룹화, 랭킹, 문자열 조건 등을 표현할 수 있어야 한다.

사용자 수식은 상품, 세그먼트, 모듈, 선반, 집기, 점포별 집계 지표를 대상으로 실행된다. 수식은 SQL을 직접 입력하는 방식이 아니라 제한된 DSL 또는 expression engine으로 평가한다.

### 6.1 수식 문법 범위

| 유형 | 설명 |
|---|---|
| 조건식 | `=`, `>`, `<`, `>=`, `<=`, `AND`, `OR` |
| 분기식 | `IF(condition, true_value, false_value)` |
| 문자열 | `CONTAINS(field, "text")` |
| 산술식 | `+`, `-`, `*`, `/`, 괄호 |
| 집계식 | `SUM`, `COUNT`, `AVERAGE`, `MINIMUM`, `MAXIMUM` |
| 고유값 집계 | `COUNT_UNIQUE` |
| 필터 | `FILTER_IN(condition)` |
| 순위 | `RANK_BY` |
| 사분면 | `QUADRANT` |
| 누적 | `CUME(value)` |
| 객체 참조 | `Name(product)`, `Width(Segment)`, `Height(product)` 등 |

### 6.2 사용자 수식 예시

아래 수식은 Action List 옵션에서 조건, 계산값, 그룹 기준, 필터 기준으로 사용할 수 있는 예시이다.

```text
IF(Width=300,1,0)=0
Width=300
IF(CONTAINS(Name,"연어") OR CONTAINS(Name,"동원1"),"연어/동원1","동원2/사조/오뚜기")
CONTAINS(Desc 2,"고등어") OR CONTAINS(Desc 2,"꽁치")
SUM(Value 10 (performance))
Value 10 / SUM(Value 10) * 100
SUM(Height(product) * Width(product))
SUM(IF(Category(product)="MyCo", Value 10, 0))
SUM(IF(Name(Segment)="스칸디나", Width(Segment), 0))=100
SUM(Value 10) / COUNT()
COUNT_UNIQUE(Category)
COUNT_UNIQUE(IF(Number(Segment) <= 3, Width(Segment), 0)) > 1 OR Number of Segments <= 3
IF(Value 10(performance) > 250, "Winner", "Loser")
FILTER_IN(Capacity > 0) + SUM(Sales)
FILTER_IN(UNIT MOVEMENT > 5) + COUNT(UPC)
SUM
COUNT
AVERAGE
MINIMUM
MAXIMUM
RANK_BY
QUADRANT
CUME(Value 10)
CUME(Value 10) / SUM(Value 10)
```

### 6.3 지원 함수

| 함수 | 설명 |
|---|---|
| `IF(condition, a, b)` | 조건이 참이면 `a`, 거짓이면 `b` 반환 |
| `CONTAINS(value, text)` | 문자열 포함 여부 판단 |
| `SUM(expr)` | 대상 범위 합계 |
| `COUNT()` | 대상 건수 |
| `AVERAGE(expr)` | 평균 |
| `MINIMUM(expr)` | 최소값 |
| `MAXIMUM(expr)` | 최대값 |
| `COUNT_UNIQUE(expr)` | 고유값 건수 |
| `FILTER_IN(condition)` | 조건에 맞는 대상만 포함 |
| `RANK_BY(expr, direction)` | 표현식 기준 순위 |
| `QUADRANT(expr1, expr2)` | 2개 축 기준 사분면 분류 |
| `CUME(expr)` | 정렬 기준 누적값 또는 누적 구성비 계산용 누계 |

### 6.4 참조 객체와 필드

수식은 다음 객체의 속성과 지표를 참조할 수 있다.

| 객체 | 예시 |
|---|---|
| `product` | 상품명, 카테고리, 브랜드, 상품 폭/높이/깊이, 매출, 판매수량 |
| `Segment` | 세그먼트명, 세그먼트 번호, 폭, 높이, 방향 |
| `Module` | 모듈 번호, 폭, 높이, 깊이 |
| `Shelf` | 선반 번호, 선반 높이, 선반 깊이 |
| `Equipment` | 집기 유형, 용량, 계약 위치 여부 |
| `Store` | 점포 코드, 점포 유형, 권역, 면적 |
| `performance` | 매출, 판매수량, 재고수량, Value 10 등 집계 지표 |

필드명은 사용자가 이해하기 쉬운 화면 표시명과 시스템 내부 영문 필드명을 모두 매핑할 수 있어야 한다. 예를 들어 `Value 10`, `UNIT MOVEMENT`, `Desc 2`처럼 레거시 또는 상용 POG 도구에서 사용하던 명칭도 필드 사전에서 내부 컬럼으로 변환한다.

### 6.5 수식 검증 원칙

사용자 수식은 실행 전에 반드시 검증한다.

검증 항목:

- 괄호 쌍 정상 여부
- 문자열 따옴표 정상 여부
- 지원 함수 여부
- 참조 필드 존재 여부
- 함수 인자 개수와 타입
- 집계 함수 사용 가능 범위
- 0으로 나누기 가능성
- 실행 대상 건수 제한
- 순환 참조 여부

검증 실패 시 action 실행을 중단하고 사용자에게 오류 위치와 원인을 표시한다.

### 6.6 실행 원칙

사용자 수식은 보안과 성능을 위해 직접 SQL로 실행하지 않는다. 시스템은 수식을 파싱하여 AST로 변환하고, 허용된 함수와 필드만 실행한다.

```text
사용자 수식 입력
→ 토큰화
→ 구문 검증
→ 필드/함수 검증
→ 실행 계획 생성
→ 미리보기 실행
→ 적용 또는 rollback
```

수식 실행 결과는 action 단계별로 저장하여, 어떤 수식이 어떤 상품과 위치를 변경했는지 추적할 수 있어야 한다.

## 7. 실행 모드

| 실행 모드 | 설명 |
|---|---|
| STEP_PREVIEW | 특정 action만 미리보기 실행 |
| STEP_APPLY | 특정 action 결과 적용 |
| FULL_PREVIEW | 전체 action list 미리보기 |
| FULL_APPLY | 전체 action list 적용 |
| BATCH_APPLY | 확정 프로젝트 기준 배치 실행 |
| ROLLBACK | 특정 단계 또는 실행 전 상태로 복원 |

## 8. 점별진열대장 생성 로직

```text
1. 확정 프로젝트와 진열제안 결과 조회
2. 표준진열대장과 점별진열대장 매핑 조회
3. 점포별 공간 마스터 및 점별 진열대장 속성 조회
4. Action List 옵션 조회
5. 점포별 미취급/취급중단/공간 제약 반영
6. 사용자 수식 평가
7. 부족/초과 처리
8. 상품 위치, 면수, 깊이 재계산
9. 점별진열대장 저장
10. 실행 로그 및 변경 이력 저장
```

## 9. 주요 데이터 모델

### 9.1 action_list

| 컬럼 | 설명 |
|---|---|
| action_list_id | Action List ID |
| project_id | 프로젝트 ID |
| action_list_name | Action List명 |
| standard_planogram_id | 표준진열대장 ID |
| status | DRAFT / ACTIVE / CLOSED |
| created_by | 생성자 |
| created_at | 생성일시 |

### 9.2 action_step

| 컬럼 | 설명 |
|---|---|
| action_step_id | Action 단계 ID |
| action_list_id | Action List ID |
| phase_code | 명령 단계 |
| step_order | 전체 실행 순서 |
| step_order_in_phase | 단계 내 실행 순서 |
| key1 | 실행 명령 단위 |
| command_definition_id | 실행 명령 정의 ID |
| action_type | Action 유형 |
| option_json | key1 설정 화면의 전체 JSON 옵션값 |
| insert_target_phase_code | common 명령 삽입 대상 단계 |
| insert_before_step_order | common 명령 삽입 기준 이전 순서 |
| insert_after_step_order | common 명령 삽입 기준 이후 순서 |
| formula_text | 사용자 수식 |
| formula_ast_json | 파싱된 수식 구조 |
| formula_validation_status | 수식 검증 상태 |
| formula_validation_message | 수식 검증 메시지 |
| enabled | 사용 여부 |

### 9.3 action_command_definition

Action List에서 사용할 수 있는 명령과 옵션 정의.

| 컬럼 | 설명 |
|---|---|
| command_definition_id | 명령 정의 ID |
| phase_code | 단계 코드 |
| key1 | 1단계 명령 그룹 |
| key2 | 2단계 옵션 |
| key3 | 3단계 옵션 |
| key4 | 4단계 옵션 |
| input_type | check / text / select / radio / number 등 |
| default_value | 기본값 |
| value_format | 값 형식 |
| allowed_values | 지정값 |
| is_common_command | common 명령 여부 |
| movable_scope | SAME_PHASE / ANY_PHASE |
| description | 설명 |
| effect | 효과 |
| display_order | 화면 표시 순서 |
| enabled | 사용 여부 |

### 9.4 action_execution

| 컬럼 | 설명 |
|---|---|
| execution_id | 실행 ID |
| action_list_id | Action List ID |
| execution_mode | 실행 모드 |
| target_store_id | 대상 점포 |
| status | READY / RUNNING / SUCCESS / FAILED / ROLLED_BACK |
| started_by | 실행자 |
| started_at | 시작일시 |
| ended_at | 종료일시 |

### 9.5 action_execution_detail

| 컬럼 | 설명 |
|---|---|
| execution_id | 실행 ID |
| action_step_id | Action 단계 ID |
| product_id | 상품 ID |
| before_value_json | 실행 전 값 |
| after_value_json | 실행 후 값 |
| message | 처리 메시지 |

### 9.6 store_planogram_item

| 컬럼 | 설명 |
|---|---|
| store_planogram_id | 점별진열대장 ID |
| store_id | 점포 ID |
| product_id | 상품 ID |
| module_id | 모듈 ID |
| shelf_id | 선반 ID |
| equipment_id | 집기 ID |
| position_no | 위치 순번 |
| facing_quantity | 면수 |
| display_depth | 진열 깊이 |
| display_quantity | 진열 수량 |
| source_project_id | 생성 기준 프로젝트 |
| source_action_list_id | 생성 기준 Action List |

## 10. 실시간 변경 미리보기

사용자는 Action List의 각 옵션을 단계별로 실행하면서 결과를 화면에서 확인할 수 있어야 한다.

필수 기능:

- 실행 전/후 비교
- 표준진열대장과 점별진열대장 동시 표시
- action 단계별 적용/해제
- 옵션값 변경 후 즉시 재실행
- 특정 단계 rollback
- 미배정 상품, 제외 상품, 초과 상품 사유 표시

## 11. 주요 화면/서버 처리

| 화면 처리 | 설명 |
|---|---|
| Action List 생성 화면 액션 | Action List 생성 |
| Action 명령 카탈로그 조회 화면 처리 | Action 명령 카탈로그 조회 |
| Action 단계 저장 화면 액션 | Action 단계 저장 |
| 미리보기 실행 화면 액션 | 점별진열대장 엔진 preview job 생성 |
| 수동 실행 화면 액션 | 점별진열대장 엔진 apply job 생성 |
| 확정 프로젝트 배치 실행 화면 액션 | 확정 프로젝트 배치 실행 job 생성 |
| 실행 결과 복원 화면 액션 | 실행 결과 복원 |
