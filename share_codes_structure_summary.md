# Share_codes 코드 구조 요약 (현재까지 “열어 확인한 것” 기준)

## A. 공용 설정/유틸 (루트)

- `settings.py` (확인함)  
  - 경로(`data/*`), 버전(`label_ver`, `feature_ver`), 42개 feature 목록(`all_feature_names`), 모델 HP 범위(RF/XGB/SVM).  
  - 라벨 dict와 CV split pkl을 **파일이 존재하면 import 시점에 로딩**하는 구조.

- `data_utils.py` (확인함)  
  - 입력 데이터 → `torch.Tensor` 변환(`to_tensor`), 시간 포맷/기준일 변환(`convert_seconds`, `convert_date`).
  - `step` 컬럼 기준으로 CC/CV/Rest_chr/Dch/Rest_dch 구간 분리(`slicing_data_by_step`); cell_id에 'Vm4.3'가 포함되면 2차 충전(Second_CC/Second_CV/Second_chr/Second_Rest_chr)도 추가 분리.
  - 분리된 구간 중 필요한 데이터 반환(`get_charge_data`: 기본은 `Chr_data`, 옵션으로 CC/CV/Dch/2차 충전 반환).
  - 하이퍼파라미터 후보값 생성(`get_param_values_int/log/float/category`).

- `feature_names.py` (확인함)  
  - feature 코드 → 표시명 매핑 dict 제공: `feature_name_html`(HTML/웹 표기), `feature_name_mpl`(Matplotlib LaTeX 표기). `get_feature_name(feature_code, format='html'|'mpl')`로 포맷 선택 조회, feature_name_dict는 html dict alias(하위호환).
  - 매핑 문서/검증 도구: 콘솔 출력(`print_all_conversions`), PNG 생성(`test_matplotlib_rendering`, `create_feature_grid`, `create_category_view`), HTML 리포트 생성(`save_features_html`: matplotlib 렌더링 이미지를 base64로 HTML에 포함하고 브라우저 오픈).
  - CLI 지원: python feature_names.py [html|test|grid|category|all|print].

- `figure_default_settings.py` (확인함)  
  - Plotly/Matplotlib 기본 스타일(템플릿/rcParams) 설정 유틸: Plotly `custom_mirror_grid` 템플릿 생성(`plotly_default`) + legend 위치 전환(`outside_legend`/`inside_legend`), Matplotlib rcParams 프리셋(`matplot_default`).
  - Figure 크기 유틸: Plotly는 mm→px(`mm_to_px`, `get_figure_size_mm`), Matplotlib은 mm→inch(`get_matplot_figure_size_mm`). PREVIEW_DPI(96)/EXPORT_DPI(600) 구분.
  - 생성/저장/표시: Plotly `create_fig`, `add_subplot`, `save_with_scale`(write_image scale로 물리폭 맞춤, 선택적 HTML 저장), `show_fig`; Matplotlib `create_matplot_fig`, `save_matplot_with_scale`, `show_matplot_fig`.

---

## B. 데이터 포맷/레지스트리 (라이브러리)

- `tool_battery_data/battery_data.py` (확인함)  
  - `CycleData`, `BatteryData`(및 `CyclingProtocol`) 정의.
  - `BatteryData.load(path)`: pickle에 저장된 dict를 로드 후 객체로 재구성.
  - `BatteryData.dump(path)`: 객체를 `to_dict()`로 dict 직렬화하여 pickle 저장.

- `tool_battery_data/data_registry.py` (확인함)  
  - `data_registry.pkl` 캐시가 있으면 로드, 없으면 `processed_data_path`의 pkl들을 순회하며 `BatteryData.load`로 실제 로드해 `cell_id → 파일경로`, `cell_type → 파일경로 리스트` dict 생성 후 캐시 저장.  
  - `cell_type`은 `cell_id.split('_')[0]` 기반이며 'Ref' 포함 시 'Ref'로 통합.
  - 캐시가 없으면 import 시점에 전체 pkl 로드가 발생(초기 실행 비용 큼).

---

## C. Feature 계산/조립 + 라벨 생성 (라이브러리)

- `tool_feature_extraction/battery_features.py` (확인함)  
  - 개별 feature 계산 함수(dQ, charge time, rest voltage, SOH/CE, 기타) 모음.
  - 캐시(pkl) 구조: `cell_capacity_data.pkl`, `Rest_voltage_data.pkl`, `charge_time_data.pkl` 경로를 잡고 존재 시 `LazyMapping`으로 지연 로딩(최초 접근 시 `pickle.load`).
  - 주의: `Capacity_dict`, `Rest_voltage_data_dict`는 캐시가 없으면 일부 feature 계산에서 NameError/KeyError 위험이 있음(사전 생성 함수 save_* 실행 필요).
  - 또한 `cell_id_path_dict`를 import하므로, `data_registry`의 초기 로딩 비용/부작용이 영향을 줄 수 있음.

- `tool_feature_extraction/feature_extractor.py` (확인함)  
  - `FeatureExtractor.get_features(cur_cyc_i)`는 `settings.all_feature_names`에 포함된 feature만 계산/조립(기본 설정에선 42개). need()/need_suffix()로 불필요 계산을 건너뛰어 비용 절감.
  - 라벨: `labeling_regression`는 RSL(=hard_short_cycle−현재 cycle), `labeling_classification`은 `RSL<=N_cycle_interval` 기준 이진 라벨(+ RSL==0이면 예외).
  - 누락 feature 존재 시 `ValueError`로 강하게 검증.

---

## D. 파이프라인 실행 (스크립트/노트북)

### 0) 전처리

- `scripts/0_data_preprocessing/txt_data_to_csv_data.ipynb` (확인함)  
  - txt → csv 변환 노트북. 헤더는 `Index`로 시작하는 줄을 탭 분리해 추출하고, 데이터는 `1` 또는 `1\t`로 시작하는 줄부터 끝까지를 `split()`으로 파싱해 저장. `miss_col_list`(기본 IR(ohm)) 컬럼 제거 옵션 포함. cell_type_list를 순회하며 `txt_data/{cell_type}` → `raw_data/{cell_type}`로 저장(경로 하드코딩).

- `scripts/0_data_preprocessing/fast_data_preprocessing_with_gpu.ipynb` (확인함)  
  - `raw_data/` 하위 csv들을 스캔해 셀별 `processed_data/*.pkl` 생성. `dask_cudf/cudf`로 CSV를 읽고, cycle별로 Step_No.×Current(A) 평균을 이용해 음수 전류 step을 방전 구간으로 판정한 뒤 `|Q|(Ah)`를 0 처리해 charge/discharge capacity를 분리. Test_Time(s)를 초 단위로 변환해 저장. 이후 `nominal_capacity.csv`로 `nominal_capacity_in_Ah`를 pkl에 후처리 업데이트. 경로 하드코딩 존재.

### 1) Feature 추출 (엔트리)

- `scripts/1_feature_extraction/feature_extraction.py` (확인함)  
  - `cell_id_path_dict`의 각 셀 pkl을 `BatteryData.load`로 로드 후, cycle별로 `FeatureExtractor.get_features` + RSL/Class 라벨을 생성하고 cell_id, cycle_number 메타를 추가해 통합 DataFrame으로 저장(feature_path = .../features_{feature_ver}.pkl).  
  - 누수 방지: `hard_short_dict[cell_id]`(ISC 사이클) 기준으로 `cycle_number >= ISC`에서 즉시 break하여 ISC 사이클 포함 이후 데이터를 제외. 또한 `feature_start_cycle` 이전 cycle은 skip.
  - 실행 방식: 멀티프로세싱(`ProcessPoolExecutor`) + 배치 처리/중간 저장 + 중단 시 이어달리기(기존 pkl 로드 후 resume) + 완료 후 컬럼을 (cell_info/label/feature/other) 2-level MultiIndex로 변환해 최종 저장.

### 2) Split

- `scripts/2_data_split/split.py` (확인함)  
  - feature_path(features pkl)을 로드 후, `set_test_ids`(cell_id 리스트) 기준으로 셀 ID 홀드아웃 분할하여 train/test DataFrame을 구성.
  - `x_train/x_test`(feature), `y_train/y_test`(Regression=RSL 또는 Classification=Class)을 `to_tensor`로 텐서 변환 후 pkl 저장. 저장 산출물: x_train.pkl, x_test.pkl, y_train_{RSL|Class}.pkl, y_test_{RSL|Class}.pkl, train_cell_ids.pkl, test_cell_ids.pkl, feature_columns.pkl (저장 경로는 train_data_path, test_data_path 설정값).
  - 파일 하단에서 `split_data('Regression')`을 직접 호출하므로 실행/임포트 시 자동 수행됨.

### 3) Sequence/Flatten 생성

- `scripts/3_data_preparation/sequential_data.py` (확인함)  
  - features pkl(`feature_path`) 로드 후 `set_test_ids`로 **cell_id 기준 train/test 분리**. window_size=10으로 슬라이딩 윈도우 시퀀스 생성(입력: 과거 10 cycles feature, 타깃: 현재 cycle의 RSL/Class). NaN 포함 시퀀스는 skip, feature 차원 불일치 시 ValueError.
  - 저장(모두 pkl, torch tensor 기반):
    - train: X_seq_train.pkl, y_seq_train_RSL.pkl, y_seq_train_Class.pkl, seq_train_cell_ids.pkl, X_flat_train.pkl, y_flat_train_RSL.pkl, y_flat_train_Class.pkl
    - test: X_seq_test.pkl, y_seq_test_RSL.pkl, y_seq_test_Class.pkl, seq_test_cell_ids.pkl, X_flat_test.pkl, y_flat_test_RSL.pkl, y_flat_test_Class.pkl
  - `GroupKFold` 기반 CV split 예시 코드는 **주석 처리**되어 있으며, 그대로 uncomment 시 y_seq_train 변수 미정의로 수정이 필요할 수 있음.

### 4) 모델 학습 (현재 공유된 범위: RF 회귀)

- `scripts/4_model_training/.../2_RF/RF_tuning_reg.ipynb` (확인함)  
  - `X_flat_train`, `y_flat_train_RSL` 로드 후, `cv_splits_seq_train`을 사용해 RF 회귀 하이퍼파라미터 탐색을 수행(GridSearchCV, RandomizedSearchCV).
  - 탐색 공간은 `settings.RF_reg_param_ranges`를 `data_utils.get_param_values_*`로 샘플링해 `grid_params`를 구성.
  - 추가로 `StandardScaler + RF 파이프라인` 기반 GridSearch도 포함.

- `scripts/4_model_training/.../2_RF/RF_eval_reg.ipynb` (확인함)  
  - X_flat_train/test, y_flat_train/test_RSL 로드 후, 노트북 내에 **수동 정의한 `selected_params`**로 RF 학습/예측 및 RMSE(train/test) 계산.
  - Plotly 기반 시각화(Train/Test 산점도 + test error 히스토그램 인셋).
  - `save_with_scale`를 이용한 저장 코드는 예시로 존재하나 현재는 주석 처리.

---

## E. 라벨링/테스트 노트북

- `notebooks/labeling/Hard short labeling.ipynb` (확인함)  
  - `labeling_hard_short_cycle()`에서 6개 조건(cond_1~cond_6)로 flag를 생성하고, 가장 이른 flag 발생 cycle을 hard short cycle로 반환(규칙 기반 탐지/QC).
  - 전체 cell을 순회하며 기존 `hard_short_dict`와 재탐지 결과를 비교 출력하며, 일부 셀은 hard short cycle을 **수동으로 고정(하드코딩 오버라이드)**.
  - hard short cycle dict 저장(`hard_short_cycle_{...}.pkl`/`hard_short_dict`)은 **주석 처리**되어 있고, 대신 `hard_short_flag_{label_ver}.pkl`(flag 목록 dict)은 **실제로 저장**.
  - flag 요약 xlsx 생성 코드는 존재하나 **주석 처리**.

- `notebooks/labeling/Soft short labeling.ipynb` (확인함)  
  - soft short 탐지 로직은 3개 조건(1: CC 전압 drop peak, 2: CV 전류 increase peak, 3: 저전압(2.5V 이하 & I=0)) 기반이며, 조건 1~2는 peak pkl을 사용.
  - peak pkl(CC_charge_max_voltage_drop_peak.pkl, CV_charge_max_current_increase_peak.pkl)이 없으면 노트북에서 직접 계산 후 **pkl로 저장**. 
  - soft_short_dict(cycle dict) 저장과 soft_short_flag_{label_ver}.pkl 저장은 모두 **주석 처리**.
  - 추가 QC 도구로, xlsx(soft_short_flag_detection_{label_ver}.xlsx)를 입력으로 받아 “최초 flag” 기준 플롯을 생성해 HTML 리포트를 만드는 함수가 있으나, xlsx 생성 코드는 주석 처리되어 있어 별도 준비가 필요.

- `notebooks/analysis/model_HP_default_values.ipynb` (확인함)  
  - RF/XGB/SVM의 **회귀·분류 모델(RFReg/RFClf, XGBReg/XGBClf, SVR/SVC)**을 대상으로, 작은 더미 데이터로 fit() 1회 수행 후 get_params()를 출력.
  - 각 모델별로 **주요 하이퍼파라미터 목록 + 전체 파라미터 목록**을 함께 출력하며, XGB 분류/회귀 차이 요약 텍스트 포함.
  - 주의: 출력되는 default 값은 라이브러리 버전에 따라 달라질 수 있으므로(확실하지 않음) 필요하면 버전도 함께 기록하는 것이 좋음.

- `notebooks/test/test.ipynb` (확인함)  
  - feature 캐시 생성 함수(`save_charge_time_data`, `save_capacity_data`, `save_rest_voltage_data`)를 import하고 **호출은 주석 처리**.
  - `label_ver` 출력 + `data_registry.cell_id_path_dict`의 첫 key 출력(레지스트리 스모크 테스트).
  - 주의: `data_registry`는 캐시가 없으면 import 시점에 전체 processed_data pkl 로드가 발생할 수 있어 테스트가 무거워질 수 있음(코드 구조상).

- `notebooks/test/figure_test.ipynb` (확인함)  
  - `figure_default_settings` + `feature_names` 사용 예시. x_train/y_train_RSL을 DataFrame으로 변환 후 특정 cell(Cr0.5_1)의 feature 2종(예: Chr_rest_V_drop_10min, D_Rest_V_min)을 **RSL 대비** Plotly/Matplotlib로 시각화(단일 플롯 + subplot).
  - `save_with_scale` 호출은 예시로 존재하나 현재는 주석 처리.

---

## “모르는 것 / 확인하지 못함” (폴더·파일명 기준으로 명시)

- 루트 `__init__.py` : 확인하지 못함
- `tool_battery_data/__init__.py` : 확인하지 못함
- `tool_feature_extraction/__init__.py` : 확인하지 못함
- `info_feature/feature_name_list.html` : 확인하지 못함
- `info_feature/feature_matplotlib_latex_names.png` : 확인하지 못함
- `info_labeling/soft_flag_detection_summary.xlsx` : 확인하지 못함
- `info_labeling/soft_short_first_flag_only.html` : 확인하지 못함
- `notebooks/visualization/` 내부 파일들 : 확인하지 못함
- `data/cross_validation/` 내부 파일들 : 확인하지 못함
- `cv_splits_seq_train.pkl`이 **어떤 코드에서 생성되는지** : 현재 공유된 코드들에서는 생성부를 확인하지 못함(사용/로드만 존재)
