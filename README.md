# Amazon Connect Decisions — Supply & Demand Plan Quick Start

> Amazon Connect Decisions(ACD)에서 Supply Plan과 Demand Plan을 처음부터 성공시키기 위한 샘플 데이터 + 설정 가이드입니다.

---

## 목차

1. [개요](#1-개요)
2. [데이터 구조](#2-데이터-구조)
3. [사전 준비](#3-사전-준비)
4. [데이터 업로드 & 매핑](#4-데이터-업로드--매핑)
5. [Supply Plan 생성](#5-supply-plan-생성)
6. [Demand Plan 생성](#6-demand-plan-생성)
7. [데이터 검증 규칙 (필독)](#7-데이터-검증-규칙-필독)
8. [자주 발생하는 에러 & 해결](#8-자주-발생하는-에러--해결)
9. [운영 팁](#9-운영-팁)

---

## 1. 개요

### Amazon Connect Decisions란?

AWS의 AI 기반 공급망 계획 서비스입니다. 과거 판매 이력을 분석해 **수요를 예측(Demand Plan)**하고, 예측된 수요를 기반으로 **언제·얼마나·어디서 조달할지(Supply Plan)**를 자동 생성합니다.

### 이 레포에서 제공하는 것

- ✅ **15개 CDM 호환 샘플 CSV 파일** (`sample-data/` 폴더) — 바로 업로드하면 Plan 생성 가능
- ✅ **Supply Plan 설정 가이드** — Plan Details + Input Parameters
- ✅ **Demand Plan 설정 가이드** — Configuration + Planning Rules
- ✅ **데이터 매핑 방법** — Data Flow 설정 순서
- ✅ **검증 규칙 체크리스트** — 실패 방지를 위한 필수 확인 사항

### 샘플 데이터 스펙

| 항목 | 값 |
|------|-----|
| 회사 | ACD-COMPANY (가상) |
| 제품 수 | 5개 (Skincare 3 + Makeup 2) |
| 사이트 수 | 3개 (DC 2 + Store 1) |
| 벤더 수 | 3개 |
| 판매 이력 | 24개월 (~4,980건) |
| Forecast | 52주 Weekly (780건) |

---

## 2. 데이터 구조

### 파일 목록 & CDM 매핑

| # | 파일명 | CDM Target Table | 행 수 | 필수 | 역할 |
|---|--------|-----------------|------|------|------|
| 1 | `company.csv` | company | 1 | ✅ | 회사 마스터 |
| 2 | `geography.csv` | geography | 1 | ✅ | 지역 정보 |
| 3 | `product.csv` | product | 5 | ✅ | 제품 마스터 |
| 4 | `site.csv` | site | 3 | ✅ | 거점(DC/Store) |
| 5 | `tradingpartner.csv` | trading_partner | 3 | ✅ | 벤더/공급자 |
| 6 | `vendorproduct.csv` | vendor_product | 15 | ✅ | 벤더-제품 매핑 |
| 7 | `vendorleadtime.csv` | vendor_lead_time | 15 | ✅ | 벤더별 리드타임 |
| 8 | `sourcingrules.csv` | sourcing_rules | 15 | ✅ | 조달 규칙 (어디서 살 것인가) |
| 9 | `inventorypolicy.csv` | inventory_policy | 15 | ✅ | 안전재고 정책 |
| 10 | `inventorylevel.csv` | inventory_level | 15 | ✅ | 현재 재고 수준 |
| 11 | `forecast.csv` | forecast | 780 | ✅ (Supply) | 외부 수요 예측 |
| 12 | `inboundorderline.csv` | inbound_order_line | 15 | ✅ | 입고 주문 (PO) |
| 13 | `outboundorderline.csv` | outbound_order_line | 4,980 | ✅ | 출고 주문 (판매 이력) |
| 14 | `supplementary_time_series.csv` | supplementary_time_series | 0 | 선택 | 외부 시그널 (프로모션 등) |
| 15 | `product_alternate.csv` | product_alternate | 0 | 선택 | 대체 제품 |

### 엔티티 관계도

```
company ─────┬──→ product (company_id)
             ├──→ site (company_id)
             ├──→ geography (company_id)
             └──→ tradingpartner (company_id)

product ─────┬──→ forecast (product_id)
             ├──→ inventorylevel (product_id)
             ├──→ inventorypolicy (product_id)
             ├──→ sourcingrules (product_id)
             ├──→ inboundorderline (product_id)
             ├──→ outboundorderline (product_id)
             ├──→ vendorleadtime (product_id)
             └──→ vendorproduct (product_id)

site ────────┬──→ forecast (site_id)
             ├──→ inventorylevel (site_id)
             ├──→ sourcingrules (to_site_id)
             ├──→ inboundorderline (to_site_id)
             ├──→ outboundorderline (ship_from_site_id)
             └──→ vendorleadtime (site_id)

tradingpartner ──→ sourcingrules (tpartner_id)
                 → inboundorderline (tpartner_id)
                 → vendorleadtime (vendor_tpartner_id)
                 → vendorproduct (vendor_tpartner_id)
```

---

## 3. 사전 준비

### 필요한 것

1. AWS 계정 + Amazon Connect Decisions 접근 권한
2. ACD 인스턴스 생성 완료
3. 이 레포의 CSV 파일들 (`sample-data/` 폴더)

### Step 1: ACD 인스턴스 생성

AWS Console에서 Amazon Connect Decisions를 검색하고 인스턴스를 생성합니다.

![AWS Console - Instance](screenshots/01-aws-console-instance.png)

### Step 2: 사용자 등록 (Sign up)

인스턴스 생성 후 이메일로 사용자를 등록합니다. IAM Identity Center와 연동됩니다.

![Sign up with email](screenshots/02-sign-up-email.png)

### Step 3: 초기 설정 (Get Started)

ACD 앱에 처음 접속하면 4단계 온보딩 질문이 나옵니다:

![Get Started Steps](screenshots/03-get-started-steps.png)

| Step | 질문 | 권장 응답 |
|------|------|---------|
| 1 | Select industry | **CPG** (또는 해당 업종) |
| 2 | Define biggest challenge | **Address both inventory optimization and demand forecast accuracy** |
| 3 | Choose demand planning approach | **Bring in existing forecasts** |
| 4 | Choose inventory planning approach | **Bring in existing inventory projections or plans** |

Submit 클릭하면 Home 대시보드로 이동합니다.

### Step 4: Home 대시보드 확인

온보딩 완료 후 대시보드에서 "Connect your data"부터 시작합니다.

![Home Dashboard](screenshots/04-home-dashboard.png)

---

## 4. 데이터 업로드 & 매핑

### Step 5: 파일 업로드

**Data Management → Upload Files for New Data Source**에서 `sample-data/` 폴더의 모든 CSV를 한꺼번에 업로드합니다.

![Upload Files](screenshots/05-upload-files.png)

> 지원 형식: CSV, Parquet | 최대 파일 크기: 5GB | 파일명: 영문+숫자+밑줄만 허용

### Step 6: 데이터 매핑

업로드 후 자동으로 Data Mapping 화면이 나옵니다. Decisions Teammate가 각 소스를 CDM 타겟 테이블에 매핑합니다.

![Data Mapping](screenshots/06-data-mapping.png)

각 테이블의 매핑을 하나씩 진행합니다 (Decisions Teammate에게 요청하면 SQL을 자동 생성해줍니다). 모든 매핑이 **Complete** 상태가 되면 **Accept mappings**를 클릭합니다.

### Step 7: Data Flow 실행 & 완료 확인

Accept Mappings 후 Data Management → Destinations 탭에서 각 플로우의 실행 상태를 확인합니다.

**In progress 상태** — 데이터 수집 진행 중:

![Data Flow In Progress](screenshots/07-data-flow-in-progress.png)

**모든 플로우 Succeeded** — 이 상태가 되어야 Plan 생성 가능:

![Data Flow Succeeded](screenshots/08-data-flow-succeeded.png)

> ⚠️ 하나라도 Failed가 있으면 Plan 생성이 실패합니다. Errors 탭에서 원인을 확인하세요.

### 업로드 순서 (중요!)

Master 데이터 → 관계 데이터 → 상태 데이터 → 트랜잭션 순서로 업로드해야 참조 무결성이 유지됩니다.

```
Phase 1 (Master):
  1. company.csv          → company
  2. geography.csv        → geography
  3. product.csv          → product
  4. site.csv             → site
  5. tradingpartner.csv   → trading_partner

Phase 2 (Relationships):
  6. vendorproduct.csv    → vendor_product
  7. vendorleadtime.csv   → vendor_lead_time
  8. sourcingrules.csv    → sourcing_rules

Phase 3 (State):
  9. inventorypolicy.csv  → inventory_policy
  10. inventorylevel.csv  → inventory_level
  11. forecast.csv        → forecast

Phase 4 (Transactions):
  12. inboundorderline.csv  → inbound_order_line
  13. outboundorderline.csv → outbound_order_line
```

### 매핑 방법 (각 파일별)

**Data Management → Data flows**에서 각 CDM 타겟 테이블에 대해:

1. **해당 테이블의 Data flow** 클릭 (또는 새로 생성)
2. **Source 선택**: 
   - "Add source" → 파일 업로드 또는 S3 경로 지정
   - 소스 데이터셋 이름 확인 (예: `sourcingrules`)
3. **SQL Mapping 생성**:
   - Decisions Teammate에게 요청: `"Create a SQL query from source(s) TO the "sourcing_rules" target table"`
   - 또는 수동으로 SQL 작성
4. **Accept Mappings** 클릭
5. **Run Flow** 실행
6. **Status = Succeeded** 확인

### SQL Mapping 예시

소스와 CDM 컬럼명이 동일하면 Decisions Teammate가 자동 생성합니다:

```sql
-- sourcing_rules 예시 (자동 생성됨)
SELECT
  sourcing_rule_id,
  company_id,
  product_id,
  to_site_id,
  sourcing_rule_type,
  tpartner_id,
  sourcing_ratio,
  eff_start_date,
  eff_end_date
FROM sourcingrules
```

### ⚠️ 매핑 시 주의사항

- **소스 데이터셋을 먼저 선택**한 후 SQL을 생성해야 합니다 (소스 없이 SQL만 넣으면 `sources=[]` 에러)
- SQL 생성 중 **페이지를 벗어나면 저장되지 않습니다** — 완료까지 탭 유지
- 모든 flow가 **Succeeded**인지 확인 후 Plan 생성

---

## 5. Supply Plan 생성

### 5.1 Plan Details (계획 상세)

**Plans → Supply Plan → Create Plan**

![Supply Plan Configuration](screenshots/10-supply-plan-config.png)

| 항목 | 설정값 | 설명 |
|------|--------|------|
| **Plan name** | `Full scope supply plan` | 플랜 식별 이름 |
| **Description** | `Supply plan for all products and sites` | 설명 |
| **Time bucket** | `Weekly` | 계획 단위 (Daily 또는 Weekly) |
| **Plan start day** | `Monday` | 주간 계획 시작 요일 |
| **Horizon** | `52` weeks | 계획 기간 (1~52주) |
| **Plan run** | `One time run` | 1회 실행 (또는 Recurring schedule) |

### 5.2 Input Parameters (입력 파라미터)

| 항목 | 설정값 | 설명 |
|------|--------|------|
| **Demand netting** | ✅ `Forecast` | 수요 소스 선택 (Forecast / Sales order / 둘 다) |
| **Forecast source** | `External` | External = 업로드한 forecast.csv 사용 |
| **Demand time fence** | `14` days | 이 기간 내에서는 forecast를 무시하고 확정 주문만 사용 |
| **Historical period for demand** | `14` days | 평균 과거 수요 계산에 고려할 기간 |
| **Past due supply days** | `7` days | 납기 초과해도 유효 공급으로 인정하는 일수 |
| **Planning time fence** | `0` weeks | 동결 구간 (0 = 전 기간에 새 발주 생성 가능) |
| **Allow open order qty adjustments** | `Disabled` | 기존 PO 수량 변경 허용 여부 |

### 5.3 설정 후 실행

1. **Save** 클릭
2. **Generate Plan** (또는 "Create supply plan") 클릭
3. Status: `In progress` → `Completed` 대기 (보통 5~10분)

### 5.4 파라미터 설명 상세

```
┌─────────────────────────────────────────────────────────────┐
│ Demand Time Fence (14일)                                     │
│ ├── 0~14일: 확정 주문(OPEN PO/SO)만 사용                      │
│ └── 14일 이후: Forecast 기반 수요                             │
├─────────────────────────────────────────────────────────────┤
│ Planning Time Fence (0주)                                    │
│ ├── 0 = 동결 없음 (전 기간에 새 Planned Order 생성)           │
│ └── N주 = 처음 N주는 기존 주문만 유지, 변경 없음              │
├─────────────────────────────────────────────────────────────┤
│ Past Due Supply Days (7일)                                   │
│ └── 납기일로부터 7일 지연된 PO도 아직 들어올 것으로 간주       │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Demand Plan 생성

### 6.1 Configuration (구성)

**Plans → Demand Plan → Create Plan**

![Demand Plan Configuration](screenshots/09-demand-plan-config.png)

| 항목 | 설정값 | 설명 |
|------|--------|------|
| **Time bucket** | `Weekly` | 예측 단위 |
| **Plan horizon** | `12` weeks (또는 `52`) | 예측 기간 |
| **Forecast start date** | `2026/02/01` (과거 날짜) | 과거 설정 시 Backtest 가능 (정확도 검증) |
| **Plan run** | `One time run` | 1회 실행 |

### 6.2 Forecast Granularity (예측 세부 수준)

| 차원 | 설정값 | 설명 |
|------|--------|------|
| **Product** | `Product id` (기본 포함) | 제품별 예측 |
| **Site** | `Ship from site id` | 출하 거점별 예측 — **반드시 선택!** |
| **Channel** | `None` | 데이터에 채널 구분 없으면 None |
| **Customer** | `None` | 데이터에 고객 구분 없으면 None |

> ⚠️ **Site를 선택하지 않으면** 제품 단위로만 예측이 생성되어 Supply Plan(product × site별)과 연결할 수 없습니다.

### 6.3 Advanced Configuration

| 항목 | 설정값 | 설명 |
|------|--------|------|
| **Prediction lead times** | `3` weeks | 예측이 몇 주 앞을 내다봐야 하는지 (벤더 리드타임 고려) |

### 6.4 Forecast Start Date 이해

```
과거 날짜 설정 시:
  [start_date ~ 오늘]  = Backtest 구간 (실제값과 비교 → 정확도 측정)
  [오늘 ~ +52주]       = Forecast 구간 (실제 예측)

미래 날짜 설정 시:
  [start_date ~ +52주] = Forecast만 생성 (Backtest 없음)
```

### 6.5 Demand Plan 필수 데이터

| 데이터 | 최소 요구사항 | 권장 |
|--------|------------|------|
| outboundorderline (판매 이력) | **12개월 이상** | 24~36개월 |
| product | 1개 이상 | - |
| site | 1개 이상 | - |
| company | 1개 | - |

> Supply Plan은 forecast.csv가 핵심 입력이지만, Demand Plan은 **outboundorderline(과거 판매 이력)**이 핵심입니다. 시스템이 이 이력을 학습해서 예측을 생성합니다.

### 6.6 설정 후 실행

1. 설정 완료 → **Save**
2. **Generate Plan** 클릭
3. Status: `In progress` → `Completed` (보통 10~20분)
4. 완료 후 예측 결과 확인 → 필요 시 수동 Override
5. 확정 시 **Publish** 클릭

### 6.7 Publish란?

| 상태 | 의미 |
|------|------|
| In progress | 예측 생성 중 |
| Review | 예측 완료 — 검토/편집 가능 |
| **Publish** | **공식 수요 계획으로 확정** → Supply Plan의 입력으로 사용 가능 |
| Final | 계획 주기 종료 — 편집 불가 |

---

## 7. 데이터 검증 규칙 (필독)

이 규칙을 위반하면 Plan 생성이 **Failed**됩니다.

### 7.1 Sourcing Rules

| 규칙 | 설명 |
|------|------|
| ✅ `sourcing_ratio` 합계 = 100% | 동일 (product_id + to_site_id) 조합의 ratio 합이 정확히 100이어야 함 |
| ✅ 중복 없음 | 동일 (product_id + to_site_id + tpartner_id) 조합은 1건만 |
| ✅ `sourcing_rule_type` 유효값 | `buy`, `transfer`, `manufacture` |
| ✅ 모든 SR에 대응하는 VLT 존재 | (product_id, tpartner_id)가 vendorleadtime에 있어야 함 |

### 7.2 Inventory

| 규칙 | 설명 |
|------|------|
| ✅ `on_hand_inventory` >= 0 | 음수 재고 불가 |
| ✅ `ss_policy` 유효값 | `abs_level`, `sl`, `doc_dem`, `doc_fcst` |
| ✅ min_safety_stock <= target <= max_safety_stock | 안전재고 범위 논리적 |

### 7.3 Orders

| 규칙 | 설명 |
|------|------|
| ✅ `order_date` < `requested_delivery_date` | 주문일이 배송 요청일보다 앞서야 함 |
| ✅ `expected_delivery_date` 필수 | Inbound에서 비어있으면 안 됨 |
| ✅ Inbound status: `OPEN`, `CLOSED` (대문자) | |
| ✅ Outbound status: `closed`, `open` (소문자) | 대소문자 주의! |

### 7.4 Site

| 규칙 | 설명 |
|------|------|
| ✅ `site_type` 유효값 | `Plant`, `DC`, `Store` |

### 7.5 날짜 형식

| 규칙 | 설명 |
|------|------|
| ✅ ISO 8601 형식 | `YYYY-MM-DDTHH:MM:SS.000Z` |
| ✅ 유효기간 무한 표현 | 시작: `1900-01-01T00:00:00Z`, 종료: `9999-12-31T00:00:00Z` |
| ✅ 전체 파일 형식 통일 | 같은 파일 내에서 다른 형식 혼용 금지 |

### 7.6 일반

| 규칙 | 설명 |
|------|------|
| ✅ `planned_lead_time` > 0 | 리드타임은 양수 |
| ✅ FK 참조 무결성 | product_id, site_id, tpartner_id가 마스터에 존재해야 함 |
| ✅ NULL 처리 | 빈 값은 `SCN_RESERVED_NO_VALUE_PROVIDED` 사용 |
| ✅ 인코딩 | UTF-8 (BOM 없음) |

---

## 8. 자주 발생하는 에러 & 해결

| # | 에러 메시지 | 원인 | 해결 |
|---|----------|------|------|
| 1 | `ss_policy must be one of: abs_level, sl, doc_dem, doc_fcst` | MIN_MAX 등 잘못된 값 | abs_level로 변경 |
| 2 | `On-hand inventory is negative` | 음수 재고 | 0 이상으로 보정 |
| 3 | `Order date is later than requested delivery date` | 날짜 컬럼 뒤바뀜 | order_date < delivery_date 확인 |
| 4 | `No delivery date was provided` | expected_delivery_date 비어있음 | submitted_date + lead_time으로 채움 |
| 5 | `Inbound order line routes have no matching sourcing rule` | PO의 (vendor, site, product)가 SR에 없음 | Medium — Plan에서 해당 주문만 제외됨 |
| 6 | `sources=[]` (Data flow 에러) | SQL 매핑 전에 소스 데이터셋 미선택 | Source 먼저 선택 후 SQL 생성 |
| 7 | Demand Plan Failed | outbound 이력 부족 (3개월 미만) | 최소 12개월 이력 필요 |
| 8 | `sourcing_ratio` 합계 초과 | 중복 SR로 인해 합계 > 100% | 중복 제거 + ratio 재정규화 |

---

## 9. 운영 팁

### Plan 실행 순서 (권장)

```
1. Demand Plan 생성 → Publish
2. Supply Plan에서 Forecast source = "Demand Plan" 선택
3. Supply Plan 생성
```

이렇게 하면 Demand Plan이 예측한 수요를 Supply Plan이 직접 사용합니다.

### 데이터 갱신 방법

- ACD는 **Full Replace** 방식 — 파일 전체를 교체 후 Flow 재실행
- 부분 추가(append)가 아닌 전체 덮어쓰기
- 기존 Plan은 불변 — 새 Plan 실행 시에만 변경된 데이터 반영

### 반복 실행 설정

Plan run을 `Recurring schedule`로 변경하면:
- Daily / Weekly / Monthly 자동 실행
- S3에 최신 데이터를 올려놓으면 자동 수집 → 자동 Plan 생성

### 비용 최적화

- 테스트 시 One time run 사용
- Horizon을 필요 이상 길게 잡지 않기 (데이터 없는 구간은 무의미)
- Demand Plan의 판매 이력은 24~36개월이 최적 (그 이상은 노이즈)

---

## License

This sample data is provided as-is for educational purposes. No real company data is included.

---

## References

- [Amazon Connect Decisions - Supply Planning](https://docs.aws.amazon.com/aws-supply-chain/latest/userguide/plans-supply-planning.html)
- [Creating a New Plan](https://docs.aws.amazon.com/aws-supply-chain/latest/userguide/plans-supply-creating-a-new-plan.html)
- [Configuring Plan Parameters](https://docs.aws.amazon.com/aws-supply-chain/latest/userguide/plans-supply-configuring-plan-parameters.html)
- [Common Issues and Solutions](https://docs.aws.amazon.com/aws-supply-chain/latest/userguide/common-issues-and-solutions.html)
