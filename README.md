# Telco Customer Churn Prediction  
(통신 고객 이탈 예측 프로젝트)

---

## 1. 프로젝트 개요

본 프로젝트는 **통신사 고객 이탈(Churn)을 사전에 예측**하여,  
이탈 가능성이 높은 고객을 선별하고 **효율적인 유지 전략 수립을 지원**하는 것을 목표로 한다.

단순한 분류 성능 극대화가 아니라,  
**운영 관점에서 이탈 고객을 놓치지 않는 의사결정 정책(Recall 우선)**을 중심으로  
모델 평가 및 임곗값(threshold) 전략을 설계하였다.

---

## 2. 분석 배경 및 목적

- 통신 시장은 이미 포화 단계에 접어들어,  
  **신규 고객 유치보다 기존 고객 유지의 ROI가 높은 구조**이다.
- 고객 이탈 증가는 기존 매출 감소뿐 아니라,  
  **신규 고객 획득 비용(CAC) 증가로 이어져 전체 수익성을 악화**시킨다.

### 분석 목적
- 이탈 가능성이 높은 고객을 사전에 탐지
- **이탈 고객 누락(False Negative)을 최소화**하는 것을 최우선 목표로 설정
- 운영 부담을 고려해 성능과 효율의 균형을 함께 고려

---

## 3. 데이터셋

- 출처: Kaggle – Telco Customer Churn Dataset
- 관측치 수: 7,043명
- 변수 수: 21개
- 타깃 변수: `churn` (Yes = 1, No = 0)
- 전체 이탈 비율: 약 **26.5%**

고객의 인구통계 정보, 서비스 이용 내역, 계약 유형, 요금 정보 등을 포함한다.

---

## 4. 데이터 분할 및 전처리 전략

### 데이터 분할
- Train / Validation / Test = **60 / 20 / 20**
- 타깃 비율 유지를 위해 **Stratified Split** 적용
- Validation set은 **임곗값 정책 설정 전용**,  
  Test set은 **최종 성능 평가에 1회만 사용**

### 변수 설정
- `customer_id`: 식별자이므로 제거
- `tenure_group`: EDA 과정에서 생성된 파생 변수로, `tenure`와 중복 → 제거
- `gender`: 이탈률 차이 및 통계 검정 결과 영향 미미 → 제거

### 전처리 파이프라인
- 수치형 변수: 결측값 대체 후 **StandardScaler**
- 이진 범주형 변수: **OrdinalEncoder (No=0, Yes=1)**
- 다범주형 변수: **One-Hot Encoding (drop='first')**
- 모든 전처리는 **Pipeline + ColumnTransformer** 내부에서 처리하여 데이터 누수 방지

---

## 5. 모델링 전략

### 사용 모델
- Logistic Regression
- Random Forest
- XGBoost
- LightGBM
- CatBoost

모든 모델에 대해:
- 동일한 데이터 분할
- 동일한 전처리 파이프라인
- 동일한 평가 정책을 적용하여 **공정한 비교**를 수행하였다.

### 불균형 데이터 처리
- Logistic / RandomForest: `class_weight='balanced'`
- XGBoost / LightGBM: `scale_pos_weight`
- CatBoost: `auto_class_weights='Balanced'`

---

## 6. 모델 평가 기준 및 임곗값 정책

### 핵심 평가 원칙
- **1차 지표: Recall (≥ 0.80)**
- **2차 지표: F1 Score**
- 보조 지표: Precision, PPR, PR-AUC

### 임곗값(threshold) 정책
- Validation set에서 다음 조건을 만족하는 임곗값 선택:
  1. Recall ≥ 0.80을 만족
  2. 해당 조건을 만족하는 범위 내에서 **F1 Score 최대**
- 선택된 임곗값은 **고정**하여 Test set에 1회 적용
- 하이퍼파라미터 튜닝(CV)과 임곗값 선택을 분리하여 **최적화 편향 방지**

---

## 7. 모델 비교 및 최종 선정

각 모델에 대해 Base / RandomizedSearch / Optuna 탐색을 수행한 뒤,  
**운영 정책(Recall ≥ 0.80)을 만족하는 최고 성능 모델**을 대표값으로 선정하여 비교하였다.

### 최종 선택 모델
- **CatBoost + Optuna**

선정 이유:
- Recall 기준을 안정적으로 충족
- Recall 제약 하에서 **F1 Score 최대**
- PR-AUC가 가장 높아 **이탈 고객 랭킹 품질이 우수**

---

## 8. 최종 성능 (Test Set)

- Recall: **0.797**
- Precision: **0.527**
- F1 Score: **0.635**
- PR-AUC: **0.649**
- Predicted Positive Rate (PPR): **40.1%**

Validation 대비 성능은 소폭 하락했으나,  
과적합이나 임곗값 불안정 징후는 관찰되지 않았다.

---

## 9. 주요 인사이트 및 비즈니스 시사점

### 이탈 위험이 높은 고객 특성
- Fiber optic 서비스 이용
- 월 단위 계약(Month-to-month)
- 상대적으로 높은 월 요금($70+)
- 단기 가입 고객
- 보안 서비스 미가입

### 이탈 위험이 낮은 고객 특성
- 1–2년 장기 계약
- DSL 또는 인터넷 미사용
- 보안 서비스 가입
- 장기 가입 고객

### 비즈니스 가치
- 모델이 선별한 타겟군의 이탈 비율은  
  **전체 평균 대비 약 2배 수준**
- 무작위 타겟팅 대비 **동일 예산으로 더 많은 이탈 고객을 포착 가능**
- 유지 캠페인의 **비용 효율 개선 가능성** 제시

## 프로젝트 구조

```text
Telco_customer_project/
├─ data/
│  └─ Telco-Customer-Churn.csv
├─ notebooks/
│  └─ Telco_customer_churn.ipynb
├─ .gitignore 
└─ README.md

