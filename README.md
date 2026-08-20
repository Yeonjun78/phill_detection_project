# 경구약제 이미지 객체 검출 (Pill Object Detection)

헬스케어 시나리오 기반, 알약 사진에서 **알약의 종류(클래스)와 위치(BBox)** 를 검출하는 Object Detection 프로젝트.
초급 프로젝트 — **4팀 아기캥거루**

- **대회**: `[AI13] 경구약제 이미지 객체 검출(Object Detection)` (Kaggle 비공개)
- **평가 지표**: `mAP@[0.75:0.95]` (IoU 0.75 / 0.80 / 0.85 / 0.90 / 0.95 평균)

---

## 팀원 및 역할

| 이름 | 역할 |
|---|---|
| 채윤휘빈센트 | Project Manager |
| 김연준 | 데이터 전처리 / EDA |
| 김연주 | 모델 파이프라인 |
| 박단비 | 모델 학습 |
| 권유진 | Kaggle 제출 파이프라인 / 계획 및 보고서 작성 총괄 |


---

## 최종 결과

| 항목 | 값 |
|---|---|
| **최고 점수 제출물** | `submission_yolo12m_20260819_101347_92a8b01e0f4b201a_a52986240db61f13.csv` |
| **모델** | YOLO12m |
| **Public Score** | (점수 기입) |
| Local Validation `mAP@[0.75:0.95]` | 0.973 |

### 최종 데이터셋 계약

| 항목 | 값 |
|---|---|
| 라벨 이미지 | 12,196장 (Train 9,763 / Val 2,433) |
| 객체 | 46,394개 |
| 클래스 | 118개 |
| 촬영 조합 그룹 | 4,104개 (Train/Val 교차 누수 0건) |
| 추론 대상 | 842장 |
| 입력 해상도 | 960 × 960 (letterbox) |

---

## 파이프라인 구조

```
1_preprocessing  →  2_pipeline  →  3_train  →  4_evaluation  →  5_submission
   전처리 · EDA       공통 학습기      모델별 학습     모델별 평가        Kaggle 제출
```

```
.
├── 1_preprocessing/        # 데이터 전처리 · EDA · 품질 게이트
│   ├── YOLO11_V6_0_5_FINAL_FIXED.ipynb           # 메인 전처리 (audit/release 2-pass)
│   └── build_coco_bridge_from_yolo_V6_0_5_...    # YOLO → COCO 변환 브릿지
│
├── 2_pipeline/             # 세 모델 공통 학습 파이프라인
│   └── 아기캥거루_통합파이프라인_V6_0_5_YOLO11_FINAL_FIXED.ipynb
│
├── 3_train/                # 실험별 학습 (Colab 12h 제한 대응으로 분할)
│   ├── 1_YOLO11m_tune_b_ONLY.ipynb
│   ├── 2_YOLO12m_baseline_ONLY.ipynb
│   ├── 3_YOLO12m_tune_a_ONLY.ipynb
│   └── 4_YOLO12m_tune_b_ONLY.ipynb
│
├── 4_evaluation/           # 모델별 평가 · confidence/Top-K 정책 탐색
│   ├── YOLO11s_EVAL_ONLY.ipynb
│   ├── YOLO11m_EVAL_ONLY.ipynb
│   └── YOLO12m_EVAL_ONLY.ipynb
│
├── 5_submission/           # Kaggle 제출 CSV 생성
│   └── 아기캥거루_Kaggle_제출파이프라인_V2.1_V605_FIXED.ipynb
│
├── submissions/            # 제출 CSV
├── reports/                # 분석 문서
└── docs/
    ├── report.pdf          # 발표용 보고서
    └── journals/           # 팀원별 협업일지
```

---

## 최고 점수 모델 재현 방법

전 과정은 **Google Colab (GPU)** 기준입니다. Google Drive 마운트가 필요합니다.

### 사전 준비

```
Google Drive/
└── baby_kangaroo/
    ├── baby_kangaroo_cache/IntegratedDataset.tar   # 통합 원본 데이터
    └── week2/
        ├── 데이터전처리/yolo_전처리/                  # 전처리 산출물 위치
        └── 공통파이프라인/                            # 체크포인트·리포트 위치
```

### STEP 1 — 전처리

`1_preprocessing/YOLO11_V6_0_5_FINAL_FIXED.ipynb`

1. 최상단 실행 모드 셀에서 `DEFAULT_BUILD_MODE = 'audit'` 로 전체 실행
   → 품질 게이트 리포트만 생성, 최종 데이터셋은 만들지 않음
2. prebuild 게이트에 `FAIL`이 없는지 확인
3. `DEFAULT_BUILD_MODE = 'release'` 로 변경 후 **런타임 재시작 → 전체 실행**
4. `pill_yolo_full_v6_0_preprocessed/` 생성 확인
   (`preprocessing_report.json`의 `release_status == "READY_FOR_TRAINING_PRESPLIT"`)

**확인해야 할 출력값**

| 단계 | 기대값 |
|---|---|
| 파일 검색 | `Labeled pool 12,196 / Inference pool 842`, `train=9,763 / val=2,433` |
| 클래스 병합 | `Annotated images 12,196 / annotations 46,394 / Classes 118` |
| 그룹 계약 | `Capture groups 4,104` |
| 수동 검토 | `Blocking unresolved: 0` |

> **선택**: `build_coco_bridge_...ipynb` 는 YOLO 라벨을 COCO JSON으로 변환합니다.
> 학습에는 필요 없고, COCO 기반 평가 도구를 쓸 때만 실행하세요.

### STEP 2 — 학습

`3_train/3_YOLO12m_tune_a_ONLY.ipynb` (최고 점수 모델)

- 학습 완료 시 체크포인트가 `공통파이프라인/checkpoints/` 에 저장됩니다
- 이미 저장된 체크포인트가 있으면 SHA-256 검증 후 **재사용**하므로 재학습되지 않습니다

> 전체 9개 실험(3모델 × baseline/tune_a/tune_b)을 재현하려면
> `2_pipeline/` 통합 노트북을 사용하거나 `3_train/` 노트북을 순서대로 실행하세요.

### STEP 3 — 평가

`4_evaluation/YOLO12m_EVAL_ONLY.ipynb`

- 고정 Validation 2,433장으로 baseline / tune_a / tune_b 비교
- confidence × Top-K 그리드 탐색 후 최적 정책 선택
- `공통파이프라인/reports/eval_only_yolo12m_*_summary.json` 생성

### STEP 4 — 제출 CSV 생성

`5_submission/아기캥거루_Kaggle_제출파이프라인_V2.1_V605_FIXED.ipynb`

1. `RUN_MODE = "smoke"` 로 먼저 연결·형식 검증 (8장만 처리)
2. 통과하면 `RUN_MODE = "full"` 로 842장 전체 추론
3. `submissions/` 에 제출 CSV 생성

**제출 형식** (8컬럼)

```
annotation_id, image_id, category_id, bbox_x, bbox_y, bbox_w, bbox_h, score
```

---

## 주요 설계 결정

| 결정 | 근거 |
|---|---|
| **YOLO 3종 비교** (11s / 11m / 12m) | 모델 크기 축(11s↔11m)과 세대 축(11m↔12m)을 동시에 비교 |
| **단일 데이터셋 공유** | 모델별 데이터가 다르면 성능 차이의 원인을 분리할 수 없음. staging 시 파일별 SHA-256을 manifest와 대조해 강제 |

---

### 가설 검증 기록 (요약)

로컬 Validation과 실제 점수의 격차 원인을 다섯 개 가설로 세우고 각각 실험으로 검증했습니다.

| 가설 | 검증 방법 | 결과 |
|---|---|---|
| 유령 클래스가 mAP 분모를 희석 | 56클래스만 남긴 CSV 제출 | **기각** (0.46369 → 0.46202) |
| GT 어휘가 56종보다 많아 상한이 존재 | 리더보드 상위 점수 확인 | **기각** (타 팀 0.6+) |
| `conf=0.001` 이 잘못 선택됨 | base-Val 56장으로 재탐색 | **기각** (동일하게 0.001 선택) |
| Top-K 과다 검출 | confidence 순위별 분포 분석 | **기각** (4위까지 고신뢰, 5위부터 0.003) |
| 좌표 역변환 오류 | forward / inverse 수식 대조 | **기각** (정확한 역연산) |

**핵심 발견**: Validation 2,433장 중 대회 원본 분포는 56장(객체 180개)에 불과해,
지표를 정확히 구현했음에도 **측정 대상이 대표성을 갖지 못했습니다.**
지표를 올바르게 구현하는 것과 올바르게 측정하는 것은 별개의 문제였습니다.

---

## 보고서

📄 **[발표용 보고서 다운로드](docs/report.pdf)**

> 보고서 PDF를 `docs/report.pdf` 로 추가한 뒤 위 링크가 동작하는지 확인해주세요.

---

## 협업일지

| 팀원 | 협업일지 |
|---|---|
| 김연준 | [10일차](docs/journals/협업일지_10일차_김연준.pdf) · [11일차](docs/journals/협업일지_11일차_김연준.pdf) · [13일차](docs/journals/협업일지_13일차_김연준.pdf) |
| 김연주 | (링크 추가) |
| 권유진 | (링크 추가) |
| 박단비 | (링크 추가) |
| 채윤휘빈센트 | (링크 추가) |

---

## 환경

```bash
pip install -r requirements.txt
```

주요 버전 (평가 노트북에서 고정)

```
ultralytics==8.4.116
torchmetrics==1.9.0
pycocotools==2.0.11
tqdm==4.67.1
```

