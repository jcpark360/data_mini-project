# ESS 배터리 수명 조기 예측

상용 LFP/흑연 전지의 초기 100사이클 데이터로 EOL까지의 총 수명(`cycle_life`)을 예측하는 회귀 프로젝트입니다. 핵심 목적은 용량 저하가 뚜렷해지기 전에 수명을 추정하여 셀 선별, 팩 구성, 예방정비 및 교체 계획을 지원하는 것입니다.

## 프로젝트 개요

- 데이터셋: MIT–Stanford Battery Dataset
- Task: Regression
- Target: `cycle_life`
- 학습 데이터: Batch 1 (`2017-05-12`)
- 필수 테스트: Batch 2 (`2018-02-20`)
- 추가 테스트: Batch 3 (`2018-04-12`)
- 평가 지표: MAPE
- 비교 기준: 논문 성능 9.1% MAPE

## 데이터 정제 및 분할

- 원본 데이터는 용량이 커서 저장소에 포함하지 않습니다. 다운로드 경로, 파일명 및 배치 방법은 [`DATASET.md`](DATASET.md)를 참고합니다.
- Target이 없는 셀은 지도학습에서 제외했습니다.
- 공식 `LoadData.m` 기준에 따라 Batch 1의 미완주 셀 5개와 Batch 3의 수집 실패·미완주·노이즈 셀 6개를 제외했습니다.
- Batch 1은 충전 프로토콜 단위로 development와 hold-out을 분리했습니다.
- 같은 충전 프로토콜의 셀이 development와 hold-out에 동시에 포함되지 않습니다.
- Batch 2·3의 Target은 모델과 피처 선택에 사용하지 않고 최종 평가에만 사용했습니다.

|구분|셀 수|역할|
|---|---:|---|
|Batch 1 development|31|그룹 교차검증 및 모델 선택|
|Batch 1 hold-out|10|내부 검증|
|Batch 2|39|필수 외부 테스트|
|Batch 3|40|추가 외부 테스트|

## 핵심 EDA 인사이트

1. `cycle_life` 분포가 배치마다 크게 달라 무작위 셀 분할만으로는 일반화 성능을 평가하기 어렵습니다.
2. `dq_std = std(Q100(V)-Q10(V))`는 세 배치에서 수명과 일관된 음의 상관을 보였습니다.
3. 충전시간·용량·온도 피처는 Batch 1 내부에서는 성능을 높이기도 했지만 배치가 바뀌면 관계가 약해지거나 방향이 달라졌습니다.
4. Batch 2의 `dq_std` 중앙값은 Batch 1보다 높고 수명은 400사이클대에 집중되어 있어 뚜렷한 분포 이동이 존재합니다.

## 피처 엔지니어링과 모델 선택

비교한 피처 세트는 ΔQ 단독, ΔQ 형태, 충전용량 결합, 온도 결합, 충전정책 결합 및 기존 요약통계입니다. 후보 모델은 Ridge, ElasticNet, RandomForest, GradientBoosting이며 Dummy를 기준선으로 사용했습니다.

Batch 1 development에서 충전 정책을 그룹으로 묶은 5-fold CV를 수행했습니다. 최저 CV MAPE는 `dq_std + qc_end` ElasticNet의 7.12%였지만, 표본이 작고 `qc_end`가 배치에 따라 불안정했습니다. 이에 one-standard-error 규칙을 적용하여 최저 모델과 통계적으로 비슷한 범위에서 가장 단순한 모델을 선택했습니다.

최종 모델:

- Feature: `dq_std`
- Model: Ridge Regression
- `alpha`: 0.01
- Target transformation: `log(cycle_life)`
- 결측 처리: 학습 데이터 중앙값
- 전처리: 표준화

## 성능 결과

|구분|MAPE|해석|
|---|---:|---|
|Train (Batch 1 CV)|8.84%|5-fold GroupKFold 평균|
|Valid (Batch 1 Hold-out)|7.09%|충전 정책이 겹치지 않는 내부 검증|
|Test (Batch 2)|24.39%|필수 외부 배치 테스트|
|Gap (Train-Valid)|-1.75%p|내부 과적합 징후 없음|
|Gap (Valid-Test)|+17.31%p|배치 간 일반화 저하|
|Gap (Target-Test)|+15.29%p|논문 목표 9.1% 대비 차이|
|Test (Batch 3)|12.71%|추가 외부 배치 테스트|
|Gap (Batch2-Batch3)|-11.69%p|Batch 3에서 성능 회복|
|Gap (Target-Test, Batch 3)|+3.61%p|논문 목표 대비 차이|

Batch 1 내부 성능은 논문 목표와 유사하지만 Batch 2에서 오차가 크게 증가했습니다. 이는 학습 과적합보다는 Batch 2의 충전정책, 수명 및 핵심 피처 분포 이동에 의한 것으로 해석됩니다. 또한 본 과제의 Batch 1→Batch 2 분할은 원논문의 정확한 41/43 split과 동일하지 않으므로 9.1%는 참고 목표로 비교했습니다.

## 오류 분석

- Batch 2에서는 단수명 셀을 전반적으로 과대예측했습니다. 가장 큰 오류는 실제 393사이클 셀을 약 648사이클로 예측한 경우입니다.
- Batch 3에서는 초장수명 셀을 과소예측했습니다. 실제 1,935사이클 셀의 예측값은 약 1,044사이클이었습니다.
- 전체 MAPE 외에 실제 수명 사분위별 MAPE와 편향, 입력 피처의 학습 범위 이탈 여부를 함께 확인해야 합니다.

## ESS 도메인 해석

- 초기 `dq_std`가 큰 셀을 조기에 식별하여 장수명 셀과 동일한 팩에 혼합되는 것을 줄일 수 있습니다.
- 수명 예측값은 예방정비, 교체 우선순위 및 보증 리스크 관리에 활용할 수 있습니다.
- 새로운 충전정책에서는 오차가 증가할 수 있으므로 예측값만으로 자동 폐기 결정을 내려서는 안 됩니다.
- 실제 배포에는 대상 충전정책을 포함한 추가 학습 데이터, OOD 감지 및 예측구간 보정이 필요합니다.

## 파일 구조

```text
ess_data/
├── .gitignore                       # 원본 데이터·가상환경 제외
├── DATASET.md                       # 데이터 다운로드 및 배치 안내
├── archive/                         # 원본 MAT 데이터(저장소에서 제외)
├── results/
│   ├── figures/                     # EDA·성능·오류 분석 그래프
│   ├── model_performance.csv        # DAY2 성능 및 Gap 표
│   ├── predictions.csv              # 셀별 예측·잔차·APE
│   └── batch1_group_cv_search.csv   # Batch 1 모델 탐색 결과
├── 30-ESSHealth-scratch.ipynb       # 전체 EDA, 모델링, 평가
├── requirements.txt
└── README.md
```

## 실행 방법

```bash
git clone https://github.com/jcpark360/data_mini-project.git
cd data_mini-project
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
jupyter lab 30-ESSHealth-scratch.ipynb
```

실행 전에 [`DATASET.md`](DATASET.md)의 안내에 따라 MAT 파일을 `archive/`에 배치합니다. 노트북을 위에서부터 실행하면 Batch 1·2·3 피처 추출, EDA, 모델 탐색, 최종 평가와 `results/`의 CSV 및 그래프 생성을 재현할 수 있습니다.

## 한계

- 학습 셀이 41개로 적어 복잡한 모델의 안정성이 낮습니다.
- 한 종류의 LFP/흑연 셀과 제한된 급속충전 조건만 포함합니다.
- 배치 간 분포 이동으로 인해 새로운 운전 조건에서 성능이 보장되지 않습니다.
- Batch 3 초장수명 영역은 Batch 1의 학습 범위를 넘어 과소예측이 큽니다.

## 참고문헌

- Severson, K. A. et al. (2019). *Data-driven prediction of battery cycle life before capacity degradation*. Nature Energy, 4, 383–391.
- [논문](https://doi.org/10.1038/s41560-019-0356-8)
- [공식 데이터 처리 코드](https://github.com/rdbraatz/data-driven-prediction-of-battery-cycle-life-before-capacity-degradation)
