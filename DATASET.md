# Dataset setup

이 저장소는 대용량 원본 배터리 데이터를 포함하지 않습니다. MATLAB 데이터 파일은 파일당 약 2~3GB이며 GitHub의 일반 파일 크기 제한을 초과하므로 `archive/` 폴더가 `.gitignore`에 등록되어 있습니다.

## 데이터 출처

- 공식 데이터 페이지: <https://data.matr.io/1/>
- 공식 데이터 처리 코드: <https://github.com/rdbraatz/data-driven-prediction-of-battery-cycle-life-before-capacity-degradation>
- 관련 논문: <https://doi.org/10.1038/s41560-019-0356-8>

## 필요한 파일

공식 페이지 또는 교육 과정에서 제공된 데이터에서 다음 파일을 준비합니다.

```text
archive/
├── 2017-05-12_batchdata_updated_struct_errorcorrect.mat
├── 2018-02-20_batchdata_updated_struct_errorcorrect.mat
├── 2018-04-03_varcharge_batchdata_updated_struct_errorcorrect.mat
└── 2018-04-12_batchdata_updated_struct_errorcorrect.mat
```

`2018-04-03_varcharge` 파일은 별도의 2셀 가변 충전 실험입니다. Batch 1·2·3 모델 학습과 평가에는 사용하지 않지만 원본 데이터 구성을 기록하기 위해 목록에 포함했습니다.

## 폴더 준비

프로젝트 루트에서 다음 명령으로 데이터 폴더를 만든 뒤, 다운로드한 MAT 파일을 직접 복사합니다.

```bash
mkdir -p archive
```

설치 후 아래 코드로 파일 존재 여부를 확인할 수 있습니다.

```python
from pathlib import Path

required = [
    '2017-05-12_batchdata_updated_struct_errorcorrect.mat',
    '2018-02-20_batchdata_updated_struct_errorcorrect.mat',
    '2018-04-12_batchdata_updated_struct_errorcorrect.mat',
]

archive = Path('archive')
missing = [name for name in required if not (archive / name).exists()]

if missing:
    raise FileNotFoundError(f'Missing dataset files: {missing}')

print('Required Batch 1, 2, 3 files are ready.')
```

## 분석 코호트

- Batch 1: 공식 `LoadData.m` 기준의 미완주 셀 5개 제외
- Batch 2: `cycle_life`가 없는 셀 제외
- Batch 3: 데이터 수집 실패·미완주·노이즈 셀 6개 제외

정제 후 DAY2 평가에 사용되는 셀 수는 Batch 1 41개, Batch 2 39개, Batch 3 40개입니다.

## 주의사항

- 원본 MAT 파일을 Git 또는 Git LFS에 임의로 추가하지 않습니다.
- 공개 저장소에는 데이터 자체가 아니라 출처와 준비 방법만 제공합니다.
- 데이터 사용 및 재배포 조건은 공식 데이터 페이지의 안내를 따릅니다.
