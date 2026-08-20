# 경구약제 이미지 객체 검출 (Pill Object Detection)

헬스케어 시나리오 기반, 알약 사진에서 **알약의 종류(클래스)와 위치(BBox)** 를 검출하는 Object Detection 프로젝트.
초급 프로젝트 — **4팀 아기캥거루**

- **대회**: `[AI13] 경구약제 이미지 객체 검출(Object Detection)` (Kaggle 비공개)
- **평가 지표**: `mAP@[0.75:0.95]` (IoU 0.75 / 0.80 / 0.85 / 0.90 / 0.95 평균)

---

## 팀원 및 역할

| 이름 | 역할 |
|---|---|
| 채윤휘빈센트, 권유진 | Project Manager |
| 김연준, 박단비, 권유진 | Data Engineer |
| 김연주, 김연준, 권유진 | Model Architect |
| 채윤휘빈센트 | Experimentation Lead |
| 권유진 | 계획 및 보고서 작성 총괄 |

---

## 최종 결과

| 항목 | 값 |
|---|---|
| **최고 점수 제출물** | `submission_yolo12m_20260819_101347_92a8b01e0f4b201a_a52986240db61f13.csv` |
| **모델** | YOLO12m (`tune_a`) |
| **Kaggle Score** | **0.46959** |
| Local Validation `mAP@[0.75:0.95]` | 0.9706 (YOLO12m tune_a) |
| 추론 결과 | 842장 / 검출 객체 2,959개 |
| 추론 정책 | confidence 0.001 · Top-K 4 |

### 모델별 Local Validation 결과 (고정 Val 2,433장)

| 모델 | baseline | tune_a | tune_b | 선택 |
|---|---|---|---|---|
| YOLO11s | **0.9720** | 0.9677 | 0.9689 | tune_b |
| YOLO11m | **0.9730** | 0.9695 | 0.9708 | baseline |
| YOLO12m | **0.9730** | 0.9706 | 0.9699 | tune_a |

> 세 모델 모두 전체 Validation에서는 0.97대로 거의 동일했습니다.

---

## 데이터셋

> 용량 문제로 데이터는 GitHub에 포함하지 않았습니다. 아래 Drive 링크에서 받으실 수 있습니다.

| 구분 | 설명 | 용량 | 링크 |
|---|---|---|---|
| **통합 원본** `IntegratedDataset.tar` | 대회 원본 + AI-Hub 추가분 병합본 | 23.6 GB | [다운로드](https://drive.google.com/file/d/1gAokjRDpRa0TEEkQiILL3Spjqvbtbmii/view?usp=drivesdk) |
| **전처리 산출물** `pill_yolo_full_v6_0_preprocessed/` | 학습에 바로 사용 가능한 최종 번들 | 11.2 GB | [열기](https://drive.google.com/drive/folders/1zz7sy24yaOHNZMChwsVpwghGRtWZVOgt) |

### 데이터 구성

대회 제공 데이터만으로는 학습량이 부족하다고 판단해 AI-Hub 경구약제 데이터를 추가 확보했습니다.

| 출처 | 이미지 | 객체 | 클래스 |
|---|---|---|---|
| 대회 원본 (`base__`) | 232 | 763 | 56 |
| AI-Hub 추가분 (`extra__`) | 11,964 | 45,631 | 116 |
| **병합 결과** | **12,196** | **46,394** | **118** |

병합 시 처리한 것

- 두 출처의 클래스 체계가 달라 단순 병합 시 172개가 되는 문제 → **동일 약품을 가리키는 중복 클래스 54개 그룹을 병합**해 118개로 정리
- 출처 추적을 위해 파일명에 `base__` / `extra__` 접두사 부여
- 라벨 없는 추론 전용 이미지 842장은 별도 분리해 학습 데이터와 혼입 방지

### 최종 데이터셋 계약

| 항목 | 값 |
|---|---|
| 라벨 이미지 | 12,196장 (Train 9,763 / Val 2,433) |
| 객체 | 46,394개 |
| 클래스 | 118개 |
| 촬영 조합 그룹 | 4,104개 (Train/Val 교차 누수 0건) |
| 추론 대상 | 842장 |
| 입력 해상도 | 960 × 960 (letterbox) |

### Train / Validation 분할을 다시 만들지 않은 이유

통합 데이터셋은 추가 데이터셋을 다운로드 시점부터 이미 **Train 9,763 / Val 2,433 으로 나뉜 상태**였습니다.
이를 무시하고 새로 분할하는 대신 **주어진 분할을 그대로 보존**하기로 결정했고, 근거는 다음과 같습니다.

1. **이미 그룹 안전(group-safe) 분할이었음**
   이 데이터는 같은 알약 조합을 70° / 75° / 90° 세 각도로 촬영한 구조입니다. 무작위로 다시 나누면 같은 촬영 세트가 Train과 Val에 흩어져 성능이 과대평가됩니다.
   `split_manifest.csv`의 `group_id`를 기준으로 **4,104개 그룹 전부를 검사한 결과 Train/Val 교차 누수가 0건**임을 확인했습니다.

2. **재분할은 모델 간 비교를 깨뜨림**
   이미 학습을 시작한 실험이 있는 상태에서 분할을 바꾸면 이전 결과와 비교가 불가능해집니다.

3. **대신 전처리는 split을 유지한 채 각각 수행**
   `images/train`, `images/val`, `labels/train`, `labels/val` 구조를 그대로 두고 각 split에 동일한 전처리를 적용했습니다. 최종 산출물도 같은 구조로 내보내 학습 파이프라인이 재분할 없이 바로 사용할 수 있게 했습니다.

이 결정 때문에 전처리 노트북에는 `split_manifest.csv`의 분할을 **변경하지 않고 검증만 하는** 로직이 들어 있으며, 산출물의 `preprocessing_report.json`에 `"split_policy": "preserve_exact_upstream"` 으로 기록됩니다.

### 전처리 절차

모든 이미지에 동일하게 적용됩니다.

```
EXIF orientation 정규화
  → ICC-aware sRGB 색공간 변환
  → bilateral filter 노이즈 제거      (d=5, sigmaColor=20, sigmaSpace=20)
  → clipped gray-world 화이트밸런스    (채널 gain 제한 ±0.05)
  → LAB L채널 전용 CLAHE              (clipLimit=1.5, grid 8×8)
  → 종횡비 보존 letterbox 960×960     (padding = 테두리 중앙 RGB)
  → BBox 좌표 동기 변환 및 재검증
```

---

## 파이프라인 구조

```
1_preprocessing → 2_pipeline → 3_train_evaluation → 4_kaggle_submission → 5_final_output
   전처리·EDA       통합 학습·평가    모델별 평가·정책 선택      제출 CSV 생성          최종 제출물
```

```
.
├── 1_preprocessing/
│   └── YOLO11_전처리_EDA_V6_0_5_SUBMISSION.ipynb      # 전처리 · EDA · 품질 게이트
│
├── 2_pipeline/
│   └── 아기캥거루_통합학습평가_V6_0_5_SUBMISSION.ipynb    # 3모델 공통 학습·평가 파이프라인
│
├── 3_train_evaluation/                                # 체크포인트 기반 평가 · 정책 탐색
│   ├── YOLO11s_평가_V4_SUBMISSION.ipynb
│   ├── YOLO11m_평가_V4_SUBMISSION.ipynb
│   └── YOLO12m_평가_V4_SUBMISSION.ipynb
│
├── 4_kaggle_submission/                                # 케글 제출용 csv 파일로 변환
│   └── 아기캥거루_Kaggle_제출파이프라인_V3_1_SUBMISSION.ipynb
│
├── 5_final_output/
│   └── submission_yolo12m_20260819_101347_....csv     # 최고 점수 제출물
│
├── reports/          # 발표용 보고서
└── docs/journals/    # 팀원별 협업일지
```

> **학습과 평가를 분리한 이유**: 3모델 × 3단계 = 9개 실험의 전체 학습에 수십 시간이 필요해 Colab 세션 제한 안에서 한 번에 처리할 수 없었습니다.
> 학습은 `2_pipeline`에서 나눠 수행해 체크포인트로 저장하고, `3_train_evaluation`에서는 **저장된 체크포인트를 불러와 추론·평가만** 수행합니다.
> 각 평가 노트북은 체크포인트의 SHA-256과 manifest를 대조해 학습이 정상 완료된 것인지 검증한 뒤 사용합니다.

---

## 단계별 입력 · 출력

### STEP 1 — 전처리 `1_preprocessing/YOLO11_전처리_EDA_V6_0_5_SUBMISSION.ipynb`

**입력**
```
Google Drive/baby_kangaroo/baby_kangaroo_cache/IntegratedDataset.tar
```

**실행**
1. 최상단 실행 모드 셀에서 `DEFAULT_BUILD_MODE = 'audit'` 로 전체 실행
   → 검토용 리포트만 생성, 최종 데이터셋은 만들지 않음
2. prebuild 게이트에 `FAIL`이 없는지 확인
3. `DEFAULT_BUILD_MODE = 'release'` 로 변경 → **런타임 재시작 → 전체 실행**

**출력** — `Drive/baby_kangaroo/week2/데이터전처리/yolo_전처리/` 아래 두 폴더가 생성됩니다.

**① `pill_yolo_full_v6_0_preprocessed/`** — 다음 단계로 넘길 최종 번들

| 파일 / 폴더 | 내용 |
|---|---|
| `images/train`, `images/val` | 전처리된 이미지 960×960 (9,763 / 2,433장) |
| `labels/train`, `labels/val` | YOLO 형식 라벨 |
| `data.yaml` | 학습 진입점. `train: images/train`, `val: images/val`, `nc: 118` |
| `class_mapping.csv` / `.json` | YOLO class ID ↔ 원본 category ID 변환표 (**Kaggle 제출 시 필수**) |
| `dataset_manifest.csv` | 파일별 split · group_id · 객체 수 · SHA-256 |
| `preprocessing_report.json` | `release_status`, 계약값, 알려진 한계 |
| `dataset_version.json` | 재현용 해시 (config / label / group contract) |
| `preprocessing_transform_manifest.csv` | 이미지별 scale · padding · 보정 파라미터 |
| `label_contract.csv` | 변환된 BBox 좌표 전체 |
| `training_exclusion_candidates.csv` | 학습 제외 후보 74장 (동일 BBox 다중 클래스) |
| `class_split_feasibility.csv` | 클래스별 그룹 수 |
| `bbox_fixes_applied.csv`, `bbox_visual_review_decisions.csv` | 라벨 수정 이력 |
| `preprocessing_config.json`, `augmentation_policy.json` | 적용된 전처리·증강 설정 |
| `DATASET_CARD.md` | 사람이 읽는 요약 + Known limitations |

**② `pill_yolo_full_v6_0_preprocessed_audit/`** — 검토용 (다음 단계에서 사용하지 않음)

| 폴더 | 내용 |
|---|---|
| `reports/` | 품질 게이트, EDA 근거 CSV·JSON 약 69종 |
| `eda/` | 분포·군집·도메인 시프트 시각화 |
| `review_montages/` | 수동 검토용 몽타주 이미지 |

**정상 종료 확인**
```json
preprocessing_report.json → "release_status": "READY_FOR_TRAINING_PRESPLIT"
```

| 확인 항목 | 기대값 |
|---|---|
| 파일 검색 | `Labeled pool 12,196 / Inference pool 842`, `train=9,763 / val=2,433` |
| 클래스 병합 | `Annotated images 12,196 / annotations 46,394 / Classes 118` |
| 그룹 계약 | `Capture groups 4,104` |
| 수동 검토 | `Blocking unresolved: 0` |

---

### STEP 2 — 학습 `2_pipeline/아기캥거루_통합학습평가_V6_0_5_SUBMISSION.ipynb`

**입력**
```
Drive/baby_kangaroo/week2/데이터전처리/yolo_전처리/pill_yolo_full_v6_0_preprocessed/
```
STEP 1의 산출물 폴더를 그대로 읽습니다. 시작 시 `preprocessing_report.json`의 `release_status`와 계약값(12,196 / 46,394 / 118 / 4,104)을 검증하고, 하나라도 다르면 중단합니다.

**실행** — YOLO11s / YOLO11m / YOLO12m 각각 baseline · tune_a · tune_b 학습 (총 9개)
학습 데이터는 로컬로 staging하며, 파일별 SHA-256을 `dataset_manifest.csv`와 대조해 세 모델이 동일 데이터를 사용함을 강제합니다.

**출력** — `Drive/baby_kangaroo/week2/공통파이프라인/`

| 폴더 | 내용 |
|---|---|
| `checkpoints/` | `{모델}_{단계}_{서명}_best.pt` (9개) + `_results.csv`, `_args.yaml` |
| `manifests/` | 체크포인트별 학습 완료 여부 · split fingerprint · SHA-256 |
| `reports/` | 실험 비교표 |
| `timings/` | 실험별 소요 시간 |

---

### STEP 3 — 평가 `3_train_evaluation/YOLO12m_평가_V4_SUBMISSION.ipynb`

**입력**
```
Drive/.../pill_yolo_full_v6_0_preprocessed/    # 고정 Validation 2,433장
Drive/.../공통파이프라인/checkpoints/            # STEP 2가 만든 체크포인트
Drive/.../공통파이프라인/manifests/              # 체크포인트 검증용
```

**실행**
- 학습 없이 **체크포인트를 불러와 추론만** 수행 (`model.train`을 호출하지 않음)
- baseline / tune_a / tune_b 를 고정 Validation에서 비교
- confidence 8종 × Top-K 4종 그리드 탐색으로 대회 지표 최적 정책 선택

**출력** — `Drive/.../공통파이프라인/reports/`

| 파일 | 내용 |
|---|---|
| `eval_only_{모델}_{fingerprint}_{시각}_summary.json` | **선택된 체크포인트 · confidence · Top-K · mAP** |
| `eval_only_{모델}_..._leaderboard.csv` | 3단계 비교표 |
| `eval_only_{모델}_..._policy_grid.csv` | 정책 그리드 탐색 결과 |
| `eval_only_3model_final_....csv` | 3모델 종합 순위 |

---

### STEP 4 — 제출 CSV 생성 `4_kaggle_submission/아기캥거루_Kaggle_제출파이프라인_V3_1_SUBMISSION.ipynb`

**입력**
```
Drive/.../공통파이프라인/reports/       # STEP 3의 summary.json (체크포인트·정책)
Drive/.../공통파이프라인/checkpoints/    # 선택된 체크포인트
Drive/.../pill_yolo_full_v6_0_preprocessed/class_mapping.csv
Drive/baby_kangaroo/baby_kangaroo_cache/IntegratedDataset.tar   # 추론용 842장
```

**실행**
1. `RUN_MODE = "smoke"` 로 연결·형식 검증 (8장만)
2. `RUN_MODE = "full"` 로 전환 → 842장 전체
   - `IntegratedDataset.tar`에서 `inference_images/` 842장 추출
   - **학습과 동일한 전처리** 적용 (`preprocessing_config.json` 계약 검증 통과)
   - 세 모델 추론 → 960×960 예측 좌표를 저장된 scale·padding으로 원본 좌표계 복원
   - `class_mapping.csv`로 YOLO ID → 원본 category ID 역변환

**출력** — `Drive/baby_kangaroo/week2/kaggle_submission_v3_integrated/`

| 폴더 | 내용 |
|---|---|
| `submissions/` | 모델별 제출 CSV 3종 + `submission_RECOMMENDED_*.csv` |
| `reports/` | 제출 검증 결과, 이미지 ID 매핑표 |
| `figures/` | 예측 시각화 |
| `final_config/` | 사용된 체크포인트·정책 기록 |

**제출 형식** (8컬럼)
```
annotation_id, image_id, category_id, bbox_x, bbox_y, bbox_w, bbox_h, score
```

---

### STEP 5 — 최종 제출물 `5_final_output/`

```
submission_yolo12m_20260819_101347_92a8b01e0f4b201a_a52986240db61f13.csv
```
Kaggle Score **0.46959**

---

> 각 결정의 상세한 근거와 실험 데이터는 보고서를 참고해주세요.

---

> 상세 과정은 보고서를 참고해주세요.

---

## 보고서

📄 **[발표용 보고서 다운로드](reports/report.pdf)**

---

## 협업일지

| 팀원 | 협업일지 |
|---|---|
| 김연준 | (링크 추가) |
| 김연주 | (링크 추가) |
| 권유진 | (링크 추가) |
| 박단비 | (링크 추가) |
| 채윤휘빈센트 | (링크 추가) |

---

## 환경

```bash
pip install -r requirements.txt
```

주요 버전 (평가·제출 노트북에서 고정)
```
ultralytics==8.4.116
torchmetrics==1.9.0
pycocotools==2.0.11
tqdm==4.67.1
```

전 과정은 **Google Colab (GPU)** 환경 기준이며 Google Drive 마운트가 필요합니다.

--- 

## 주의사항

- AI-Hub 원본의 `TL_2_조합.zip`, `TS_2_조합.zip` 은 대회 train/test 원본이므로 학습에 사용하지 않았습니다.
- 데이터·체크포인트 등 대용량 산출물은 `.gitignore` 로 추적 제외되어 있습니다.
