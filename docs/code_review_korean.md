# TM2PY 코드 리뷰: Transport Modeler를 위한 학습 가이드

## 목차
1. [개요](#개요)
2. [TM2PY란?](#tm2py란)
3. [핵심 아키텍처](#핵심-아키텍처)
4. [주요 활용 방안](#주요-활용-방안)
5. [도로 수요 예측 활용](#도로-수요-예측-활용)
6. [대중교통 수요 예측 활용](#대중교통-수요-예측-활용)
7. [중점 학습 포인트](#중점-학습-포인트)
8. [실전 활용 예제](#실전-활용-예제)
9. [학습 로드맵](#학습-로드맵)

---

## 개요

TM2PY는 San Francisco Bay Area의 Travel Model 2(TM2)를 Python으로 구현한 교통 수요 예측 패키지입니다. 
이 문서는 교통 모델러(Transport Modeler)가 TM2PY를 효과적으로 학습하고 활용할 수 있도록 
코드 구조, 핵심 개념, 그리고 실무 적용 방법을 설명합니다.

---

## TM2PY란?

### 기본 개념
TM2PY는 **Activity-Based Model (ABM)** 기반의 교통 수요 예측 시스템으로, 다음과 같은 특징을 가집니다:

- **4단계 교통 수요 모델의 진화형**: 전통적인 4단계 모델을 넘어서 개인의 활동 패턴을 기반으로 통행 수요를 예측
- **Emme 연동**: INRO의 Emme 교통 계획 소프트웨어와 긴밀하게 통합
- **Python 기반**: 확장성과 유지보수가 용이한 현대적인 프로그래밍 언어 사용
- **모듈화된 구조**: 각 기능이 독립적인 컴포넌트로 구성되어 있어 재사용성이 높음

### 주요 구성 요소
```
tm2py/
├── components/          # 모델 컴포넌트들
│   ├── demand/         # 수요 생성 모듈
│   │   ├── household.py        # 가구 기반 수요
│   │   ├── air_passenger.py    # 항공 여객 수요
│   │   ├── internal_external.py # 내부-외부 통행
│   │   └── commercial.py       # 화물/상업 통행
│   └── network/        # 네트워크 배정 모듈
│       ├── highway/            # 도로 배정
│       └── transit/            # 대중교통 배정
├── emme/               # Emme 연동 관리
├── config.py           # 설정 스키마
└── controller.py       # 메인 실행 컨트롤러
```

---

## 핵심 아키텍처

### 1. Controller 패턴
TM2PY는 **RunController**를 중심으로 모든 컴포넌트를 관리합니다.

```python
# controller.py의 핵심 구조
class RunController:
    """모델 실행의 중앙 제어"""
    
    def __init__(self, config_file, run_dir):
        self.config = Configuration.load_toml(config_file)
        self.logger = Logger(self)
        self.emme_manager = EmmeManager()
        self._component_map = {}  # 컴포넌트 매핑
        
    def run(self):
        """반복 계산을 통한 모델 실행"""
        for iteration, name, component in self._queued_components:
            self._iteration = iteration
            component.run()
```

**학습 포인트:**
- Controller는 전체 모델 흐름을 관리하는 중앙 허브
- 설정(config) → 초기화 → 검증 → 반복 실행 → 로깅의 순서로 진행
- 각 컴포넌트는 독립적으로 실행되지만 Controller를 통해 데이터 공유

### 2. Component 기반 설계
모든 모델 요소는 **Component** 추상 클래스를 상속합니다.

```python
# components/component.py
class Component(ABC):
    """모든 모델 컴포넌트의 기본 클래스"""
    
    def __init__(self, controller: RunController):
        self._controller = controller
        
    @abstractmethod
    def run(self):
        """각 컴포넌트의 핵심 실행 로직"""
        pass
        
    def validate_inputs(self):
        """입력 데이터 검증"""
        pass
```

**활용 방안:**
- 새로운 수요 예측 모듈이나 배정 알고리즘을 추가할 때 Component를 상속
- `run()` 메서드만 구현하면 Controller가 자동으로 관리
- 기존 컴포넌트를 복사하여 수정하는 것이 가장 효율적

### 3. Configuration 관리
TOML 파일을 통해 모델 파라미터를 관리합니다.

```python
# config.py의 설정 구조 예시
@dataclass(frozen=True)
class HighwayConfig(ConfigItem):
    """도로 배정 파라미터"""
    classes: Tuple[HighwayClassConfig, ...]  # 차량 클래스
    maz_to_maz: HighwayMAZConfig            # MAZ 간 통행
    tolls: HighwayTollsConfig               # 통행료
    
@dataclass(frozen=True)
class TransitConfig(ConfigItem):
    """대중교통 배정 파라미터"""
    modes: Tuple[TransitModeConfig, ...]     # 교통수단
    value_of_time: float                     # 시간가치
    max_transfers: int                       # 최대 환승 횟수
```

**실무 적용:**
- 시나리오별로 다른 TOML 파일을 준비하여 빠른 비교 분석
- `scenario.toml`: 연도, 토지이용 등 시나리오 정보
- `model.toml`: 모델 파라미터, 실행 옵션

---

## 주요 활용 방안

### 1. 교통 계획 시나리오 분석
TM2PY를 활용하여 다양한 교통 정책의 영향을 평가할 수 있습니다.

**활용 사례:**
- **신규 도로 건설 효과**: 네트워크에 새 링크 추가 후 통행 패턴 변화 분석
- **통행료 정책**: 혼잡 통행료(Congestion Pricing) 도입 시 수요 변화 예측
- **대중교통 노선 개편**: 버스/지하철 노선 변경에 따른 이용자 변화 분석
- **공유 교통수단**: 차량 공유, 자율주행차 시나리오 모델링

### 2. 수요 예측 및 배정 통합
4단계 모델의 진화된 형태로 수요 생성부터 배정까지 통합 분석:

```python
# 전형적인 실행 흐름
# config.run 섹션에서 정의
initial_components = [
    "prepare_network_highway",    # 1. 네트워크 준비
]
global_iteration_components = [
    "household",                  # 2. 가구 수요 생성 (ABM)
    "air_passenger",              # 3. 항공 여객 수요
    "commercial",                 # 4. 화물 수요
    "highway",                    # 5. 도로 배정
    "transit",                    # 6. 대중교통 배정
]
```

### 3. 정책 민감도 분석
파라미터를 변경하여 정책 효과를 정량화:

- **통행 시간 가치 변화**: `value_of_time` 파라미터 조정
- **용량 제약**: `highway_capacity_factor`로 혼잡도 시뮬레이션
- **요금 탄력성**: 대중교통 요금 변화에 따른 수요 변화

---

## 도로 수요 예측 활용

### 도로 네트워크 구조

TM2PY의 도로 모델은 Emme의 네트워크 구조를 활용합니다.

#### 주요 속성
```python
# 링크 속성 (network attributes)
- length: 링크 길이 (feet)
- vdf: Volume Delay Function (용량-지체 함수)
- @free_flow_time: 자유 흐름 통행 시간 (분)
- @useclass: 차량 등급별 이용 제한
- @tollXX_YY: 시간대별, 차량별 통행료

# 노드 속성
- @maz_id: Micro Analysis Zone ID
- x, y: 좌표
- #node_county: 카운티 정보
```

### 도로 배정 프로세스

#### 1. 수요 준비 (PrepareHighwayDemand)
```python
# components/demand/demand.py
class PrepareHighwayDemand(PrepareDemand):
    """OMX 파일에서 수요 행렬 로드"""
    
    def run(self):
        # 1. OMX 파일에서 기종점(O-D) 행렬 읽기
        # 2. 반복 계산 시 MSA (Method of Successive Averages) 적용
        # 3. Emme 데이터베이스에 저장
```

**실무 활용:**
- **수요 파일 형식**: OMX (Open Matrix) 형식 사용
- **차량 구분**: 승용차(da), 상용차(sr2, sr3), 트럭(vsm, sml, med, lrg)
- **시간대 구분**: EA(이른 아침), AM(오전 첨두), MD(낮), PM(오후 첨두), EV(저녁)

#### 2. 교통 배정 (HighwayAssignment)
```python
# components/network/highway/highway_assign.py
class HighwayAssignment(Component):
    """SOLA 알고리즘을 사용한 균형 배정"""
    
    def run(self):
        # 1. 수요 로드
        demand = PrepareHighwayDemand(self.controller)
        demand.run()
        
        # 2. 시간대별 배정
        for time in self.time_period_names():
            scenario = self.get_emme_scenario(path, time)
            
            # 3. 배정 클래스 준비
            assign_classes = [
                AssignmentClass(c, time, iteration)
                for c in self.config.highway.classes
            ]
            
            # 4. 경로 선택 및 통행 배정
            self._get_assignment_spec(assign_classes)
            
            # 5. 스킴 행렬 계산 (통행 시간, 거리, 비용)
            self._export_skims()
```

**핵심 개념:**
- **균형 배정 (Equilibrium Assignment)**: 모든 경로의 통행 비용이 균형을 이루도록 반복 계산
- **SOLA (Second Order Linear Approximation)**: Emme의 고속 배정 알고리즘
- **스킴 (Skim)**: 존간 최소 통행 시간/비용 행렬 → 수요 모델의 입력으로 재사용

#### 3. MAZ-to-MAZ 배정
단거리 통행(주로 도보/자전거 거리)에 대한 특수 처리:

```python
# components/network/highway/highway_maz.py
class AssignMAZSPDemand(Component):
    """MAZ(Micro Analysis Zone) 간 최단 경로 배정"""
    
    def run(self):
        # 1. 거리별로 수요를 그룹화 (0.9, 1.2, 1.8, 2.5, 5.0, 10.0 마일)
        # 2. 각 그룹에 대해 최단 경로 계산
        # 3. 경로에 통행량 배정
```

**활용 포인트:**
- 근거리 통행 분석에 유용
- 보행/자전거 접근성 평가
- First/Last mile 문제 분석

### 도로 수요 예측 실전 예제

#### 시나리오: 신규 고속도로 효과 분석

```python
# 1. 기준 시나리오 실행
base_config = [
    "scenario_base.toml",
    "model.toml"
]
base_controller = RunController(base_config)
base_controller.run()

# 2. 신규 도로 추가 (Emme 네트워크 수정)
# - Emme Desktop에서 새 링크 추가
# - 링크 속성 설정 (용량, 속도 제한, VDF 등)

# 3. 대안 시나리오 실행
alt_config = [
    "scenario_new_highway.toml",
    "model.toml"
]
alt_controller = RunController(alt_config)
alt_controller.run()

# 4. 결과 비교
# - 링크별 통행량 변화
# - 존간 통행 시간 변화
# - 전체 VMT (Vehicle Miles Traveled) 비교
```

### 주요 파라미터 설정

```toml
# model.toml 예시
[[highway.classes]]
name = "da"  # 승용차 단독 (Drive Alone)
description = "Single occupant vehicle"
mode_code = "d"
value_of_time = 18.93  # $/hr
pce = 1.0  # Passenger Car Equivalent

[[highway.classes]]
name = "sr2"  # 2인 이상 승용차 (Shared Ride 2+)
description = "Shared ride 2 persons"
mode_code = "d"
value_of_time = 18.93
pce = 1.0

[[highway.classes]]
name = "lrg"  # 대형 트럭
description = "Large trucks"
mode_code = "d"
value_of_time = 45.0
pce = 2.5
```

---

## 대중교통 수요 예측 활용

### 대중교통 네트워크 구조

대중교통 모델은 복합 교통수단(Multi-modal) 네트워크로 구성됩니다.

#### 네트워크 요소
```python
# transit 모드
- 버스 (local bus, express bus, BRT)
- 철도 (light rail, heavy rail, commuter rail)
- 페리 (ferry)

# 주요 속성
- 노선 (route): 정류장 순서와 운행 간격
- 세그먼트 (segment): 정류장 간 링크
- 운행 시간표 (headway): 배차 간격
- 요금 (fare): 거리별, 환승별 요금
```

### 대중교통 배정 프로세스

#### 1. 네트워크 준비
```python
# transit 네트워크 속성
class TransitConfig(ConfigItem):
    modes: Tuple[TransitModeConfig, ...]
    vehicles: Tuple[TransitVehicleConfig, ...]
    
    # 승객 행동 파라미터
    value_of_time: float = 10.0  # $/hr
    initial_wait_perception_factor: float = 2.0
    transfer_wait_perception_factor: float = 2.0
    walk_perception_factor: float = 2.0
    initial_boarding_penalty: float = 5.0  # 분
    transfer_boarding_penalty: float = 5.0
    max_transfers: int = 3
```

**핵심 개념:**
- **대기 시간 인지 계수 (Wait Perception Factor)**: 승객이 느끼는 대기 시간의 가중치
  - 초기 대기 시간은 차내 시간의 2배로 인식
- **환승 페널티 (Transfer Penalty)**: 환승에 따른 불편함을 시간으로 환산
- **보행 인지 계수**: 정류장까지 걸어가는 시간의 가중치

#### 2. 대중교통 배정 알고리즘
```python
# components/network/transit/transit_assign.py
class TransitAssignment(Component):
    """Extended Transit Assignment (확장 대중교통 배정)"""
    
    def run(self):
        # 1. 수요 로드
        # 2. 최적 경로 탐색
        #    - 일반화 비용 (Generalized Cost) 최소화
        #    - GC = 차내시간 + wait_factor*대기시간 + 
        #           walk_factor*보행시간 + transfer_penalty
        # 3. 혼잡도 고려 (crowding)
        # 4. 결과 스킴 출력
```

#### 3. 요금 체계
```python
# config에서 요금 설정
[transit]
use_fares = true
fares_path = "input/fares.csv"
fare_max_transfer_distance_miles = 0.5

# 요금 계산
# - 기본 요금
# - 거리 요금
# - 환승 할인
```

### 대중교통 수요 예측 실전 예제

#### 시나리오: 신규 BRT 노선 도입

```python
# 1. 기존 네트워크에 BRT 노선 추가
# - Emme에서 새 transit line 생성
# - 정류장 위치 설정
# - 배차 간격 설정 (예: 5분 간격)

# 2. 운행 속도 및 용량 설정
[transit.modes]
[[transit.modes]]
type = "BRT"
speed = 25  # mph
in_vehicle_perception_factor = 1.0

[[transit.vehicles]]
type = "BRT_vehicle"
seated_capacity = 60
total_capacity = 100

# 3. 배정 실행 및 분석
# - 노선별 승객 수
# - 구간별 혼잡도
# - 기존 노선에서의 전환 수요
```

### 대중교통 성능 지표

TM2PY는 다음과 같은 주요 지표를 산출합니다:

```python
# 출력 지표
- 통행 시간 (travel time)
  - 차내 시간 (in-vehicle time)
  - 대기 시간 (wait time)
  - 보행 시간 (walk time)
  - 환승 시간 (transfer time)
  
- 통행 비용 (travel cost)
  - 요금 (fare)
  - 일반화 비용 (generalized cost)
  
- 승객 지표
  - 승차 인원 (boardings)
  - 노선별 혼잡도 (crowding)
  - 환승 횟수 (number of transfers)
```

**활용 방안:**
- **서비스 품질 평가**: 혼잡도, 환승 횟수로 서비스 수준 측정
- **형평성 분석**: 지역별, 소득계층별 대중교통 접근성 비교
- **재무 분석**: 승객 수 예측으로 운영 수익 추정

---

## 중점 학습 포인트

### 1. Python 프로그래밍 기초
TM2PY를 효과적으로 사용하려면 다음 Python 개념을 이해해야 합니다:

#### 객체 지향 프로그래밍 (OOP)
```python
# 클래스와 상속
class MyCustomComponent(Component):
    def __init__(self, controller):
        super().__init__(controller)  # 부모 클래스 초기화
        self.my_parameter = None
    
    def run(self):
        self._load_data()
        self._process()
        self._export()
```

#### 데이터 처리
```python
import numpy as np
import pandas as pd

# NumPy 배열 (O-D 행렬)
demand_matrix = np.zeros((num_zones, num_zones))

# Pandas DataFrame (통계 분석)
results_df = pd.DataFrame({
    'zone': zones,
    'total_trips': trip_counts
})
```

### 2. Emme API 이해
Emme는 TM2PY의 핵심 엔진입니다.

#### Emme 주요 개념
```python
# EmmeManager를 통한 접근
emme_manager = EmmeManager()

# 프로젝트 열기
project = emme_manager.project("path/to/project.emp")

# 시나리오 접근
emmebank = emme_manager.emmebank("emmebank_path")
scenario = emmebank.scenario(scenario_id)

# 네트워크 수정
network = scenario.get_network()
for link in network.links():
    if link.volume > link.data3:  # data3 = capacity
        print(f"Link {link.id} is congested")

# 행렬 작업
matrix = emmebank.matrix("mfXX")
matrix.set_numpy_data(demand_array)
```

**학습 자료:**
- Emme Desktop 매뉴얼: Modeller API 섹션
- INRO 커뮤니티 포럼
- TM2PY `emme/manager.py` 코드 분석

### 3. 교통 계획 이론

#### 균형 배정 (User Equilibrium)
- **Wardrop의 제1원칙**: 모든 사용되는 경로의 통행 비용이 동일
- **수렴 조건**: 반복 계산을 통해 균형 상태 달성
- **Gap 지표**: 현재 해가 균형 상태에 얼마나 가까운지 측정

```python
# TM2PY의 반복 계산
[run]
start_iteration = 0
end_iteration = 3  # 보통 3-5회 반복으로 수렴

# 각 반복마다
# 1. 현재 네트워크 상태로 최단 경로 계산
# 2. 수요를 최단 경로에 배정
# 3. 링크 통행량 업데이트
# 4. VDF로 새로운 통행 시간 계산
# 5. 수렴 판정 → 반복 또는 종료
```

#### 활동 기반 모델 (Activity-Based Model)
TM2PY의 가구 수요 모델은 ABM 접근법을 사용합니다:

```python
# 전통적 4단계 모델
1. 통행 생성 (Trip Generation)
2. 통행 분포 (Trip Distribution)
3. 수단 선택 (Mode Choice)
4. 통행 배정 (Trip Assignment)

# ABM 접근법
1. 인구 합성 (Population Synthesis)
2. 활동 패턴 생성 (Activity Pattern)
3. 목적지 선택 (Destination Choice)
4. 수단 선택 (Mode Choice)
5. 시간대 선택 (Time-of-Day Choice)
→ 개인의 하루 전체 활동을 시뮬레이션
```

### 4. 데이터 형식 및 입출력

#### OMX (Open Matrix Format)
```python
# OMX 파일 읽기/쓰기
from tm2py.emme.matrix import OMXManager

with OMXManager("demand.omx", "r") as omx:
    matrix = omx.read("SOV_GP_AM")  # 행렬 이름으로 읽기
    
with OMXManager("skims.omx", "w") as omx:
    omx.write("time", time_matrix)
    omx.write("distance", dist_matrix)
```

#### TOML 설정 파일
```toml
# 계층적 구조
[scenario]
year = 2050
maz_landuse_file = "input/mazdata.csv"

[highway]
    [[highway.classes]]
    name = "da"
    value_of_time = 18.93
    
    [[highway.classes]]
    name = "sr2"
    value_of_time = 18.93
```

### 5. 디버깅 및 로깅

```python
# Logger 활용
self.logger.log("Processing demand...", level="INFO")
self.logger.log_dict(assignment_spec, level="DEBUG")
self.logger.log_time("Assignment completed")

# 중간 결과 확인
demand = self._read_demand(config, time_period, num_zones)
print(f"Total demand: {demand.sum()}")
print(f"Max O-D pair: {demand.max()}")

# 네트워크 검증
for link in network.links():
    if link.volume > link.data3 * 1.5:
        self.logger.log(
            f"Warning: Link {link.id} severely congested",
            level="WARNING"
        )
```

---

## 실전 활용 예제

### 예제 1: 혼잡 통행료 시나리오 분석

**목표**: 특정 구간에 혼잡 통행료를 부과했을 때 교통량과 수익 분석

#### Step 1: 기준 시나리오 설정
```toml
# scenario_base.toml
[scenario]
name = "Base 2050"
year = 2050
maz_landuse_file = "input/mazdata_2050.csv"
```

#### Step 2: 통행료 정책 설정
```toml
# scenario_congestion_pricing.toml
[scenario]
name = "Congestion Pricing 2050"
year = 2050

[highway.tolls]
# 첨두 시간대 통행료 ($/mile)
[[highway.tolls.periods]]
time = "AM"
corridor = "I-80_bridge"
da_toll = 5.0
sr2_toll = 2.5
sr3_toll = 1.0
```

#### Step 3: 모델 실행
```python
# run_scenarios.py
from tm2py.controller import RunController

# 기준 시나리오
base = RunController(
    ["scenario_base.toml", "model.toml"],
    run_dir="runs/base"
)
base.run()

# 통행료 시나리오
toll = RunController(
    ["scenario_congestion_pricing.toml", "model.toml"],
    run_dir="runs/toll"
)
toll.run()
```

#### Step 4: 결과 분석
```python
import pandas as pd
import matplotlib.pyplot as plt

# 링크 통행량 비교
base_volumes = pd.read_csv("runs/base/output/link_volumes_AM.csv")
toll_volumes = pd.read_csv("runs/toll/output/link_volumes_AM.csv")

# 통행료 구간 통행량 변화
bridge_links = [101, 102, 103]  # I-80 bridge link IDs
base_traffic = base_volumes[base_volumes['link_id'].isin(bridge_links)]['volume'].sum()
toll_traffic = toll_volumes[toll_volumes['link_id'].isin(bridge_links)]['volume'].sum()

print(f"통행량 변화: {(toll_traffic - base_traffic) / base_traffic * 100:.1f}%")

# 수익 계산
toll_revenue = toll_volumes[toll_volumes['link_id'].isin(bridge_links)].apply(
    lambda row: row['volume'] * row['toll'], axis=1
).sum()
print(f"시간당 통행료 수익: ${toll_revenue:,.0f}")
```

### 예제 2: 신규 대중교통 노선 효과 분석

**목표**: 신규 BRT 노선의 승객 수요 및 기존 노선 영향 예측

#### Step 1: 기존 네트워크 (Emme Desktop)
1. Emme Desktop 열기
2. 기존 transit network 확인
3. 노선별 배차 간격, 운행 속도 검토

#### Step 2: 신규 BRT 노선 추가
```python
# scripts/add_brt_line.py (Emme Modeller script)
import inro.modeller as _m

def add_brt_line(scenario):
    network = scenario.get_network()
    
    # 노선 생성
    route = network.create_transit_line(
        id="BRT_1",
        vehicle="BRT_60",
        headway=5  # 5분 간격
    )
    
    # 정류장 추가
    stops = [1001, 1005, 1010, 1015, 1020]
    for stop_node in stops:
        route.add_stop(stop_node)
    
    scenario.publish_network(network)
```

#### Step 3: 설정 파일 업데이트
```toml
# model.toml
[[transit.modes]]
id = "BRT"
type = "BUS_RAPID_TRANSIT"
speed_factor = 1.2  # 일반 버스보다 20% 빠름
in_vehicle_perception = 0.9  # 승차감 우수

[[transit.vehicles]]
id = "BRT_60"
mode = "BRT"
seated_capacity = 60
total_capacity = 100
```

#### Step 4: 결과 비교
```python
# 승객 수 비교
def analyze_transit_ridership():
    # 기준 시나리오 결과
    base_boardings = emme_scenario_base.get_attribute_values(
        "TRANSIT_LINE", 
        "boardings"
    )
    
    # BRT 시나리오 결과
    brt_boardings = emme_scenario_brt.get_attribute_values(
        "TRANSIT_LINE",
        "boardings"
    )
    
    # BRT 노선 승객 수
    brt_line_ridership = brt_boardings["BRT_1"]
    print(f"BRT 노선 승객: {brt_line_ridership:,}명/일")
    
    # 기존 노선 영향
    for line_id in ["BUS_10", "BUS_20"]:
        base = base_boardings[line_id]
        brt = brt_boardings[line_id]
        change = (brt - base) / base * 100
        print(f"{line_id} 승객 변화: {change:+.1f}%")
```

### 예제 3: 자율주행차 시나리오

**목표**: 자율주행차(AV) 보급률에 따른 도로 용량 및 수요 변화

#### Step 1: AV 파라미터 설정
```toml
# scenario_av_30pct.toml
[scenario]
name = "30% AV Penetration"
av_share = 0.30

[highway]
# 자율주행으로 인한 용량 증가
av_capacity_factor = 1.15  # 15% 용량 증가

# AV 가치 시간 (낮음, 차내 생산 활동 가능)
[[highway.classes]]
name = "av"
value_of_time = 10.0  # 일반 승용차의 53%
mode_code = "d"
```

#### Step 2: 네트워크 용량 조정
```python
# scripts/adjust_capacity_for_av.py
def adjust_link_capacity(scenario, av_share, capacity_factor):
    network = scenario.get_network()
    
    for link in network.links():
        base_capacity = link.data3
        # 혼합 교통류: AV와 일반 차량
        adjusted_capacity = base_capacity * (
            1 + av_share * (capacity_factor - 1)
        )
        link.data3 = adjusted_capacity
    
    scenario.publish_network(network)
```

#### Step 3: 수요 분할 (AV vs. 일반 차량)
```python
# components/demand/av_demand.py
class AVDemandSplit(Component):
    """AV와 일반 차량으로 수요 분할"""
    
    def run(self):
        av_share = self.config.scenario.av_share
        
        # 기존 승용차 수요 로드
        da_demand = self._read_demand("SOV_GP_AM")
        
        # AV와 일반 차량으로 분할
        av_demand = da_demand * av_share
        conventional_demand = da_demand * (1 - av_share)
        
        # 별도 행렬로 저장
        self._save_demand("av", av_demand)
        self._save_demand("da", conventional_demand)
```

#### Step 4: 결과 분석
```python
# 교통 성능 지표 비교
scenarios = {
    "Base": "runs/base_2050",
    "10% AV": "runs/av_10pct",
    "30% AV": "runs/av_30pct",
    "50% AV": "runs/av_50pct"
}

metrics = {}
for name, path in scenarios.items():
    results = pd.read_csv(f"{path}/output/summary.csv")
    metrics[name] = {
        "VMT": results["total_vmt"].sum(),
        "VHT": results["total_vht"].sum(),
        "Delay": results["total_delay"].sum(),
        "AvgSpeed": results["vmt"].sum() / results["vht"].sum()
    }

comparison = pd.DataFrame(metrics).T
print(comparison)

# 시각화
comparison.plot(kind="bar", subplots=True, layout=(2,2), figsize=(12,8))
plt.savefig("av_scenario_comparison.png")
```

---

## 학습 로드맵

TM2PY를 마스터하기 위한 단계별 학습 경로를 제안합니다.

### Phase 1: 기초 다지기 (2-4주)

#### 1주차: Python 기초
- [ ] Python 문법: 클래스, 함수, 모듈
- [ ] NumPy: 배열 연산, 인덱싱
- [ ] Pandas: DataFrame 조작, 읽기/쓰기
- [ ] 실습: 간단한 O-D 행렬 처리 스크립트 작성

#### 2주차: Emme 기초
- [ ] Emme Desktop 사용법
- [ ] 네트워크 편집: 노드, 링크, 존
- [ ] 시나리오 관리
- [ ] 기본 배정 실행
- [ ] 실습: 샘플 네트워크로 배정 수행

#### 3-4주차: TM2PY 구조 이해
- [ ] `controller.py` 분석: 모델 실행 흐름
- [ ] `config.py` 분석: 설정 스키마
- [ ] `Component` 구조 이해
- [ ] 실습: 예제 모델 실행 (README의 example_union_test_highway)

```bash
# 예제 실행
conda activate tm2py
cd example_union_test_highway
tm2py -s scenario.toml -m model.toml
```

### Phase 2: 핵심 기능 마스터 (4-6주)

#### 5주차: 도로 배정 심화
- [ ] `highway_assign.py` 코드 분석
- [ ] 배정 클래스 설정 (highway.classes)
- [ ] VDF (Volume Delay Function) 이해
- [ ] 스킴 행렬 출력 및 활용
- [ ] 실습: 차량 클래스 추가, 파라미터 변경

#### 6주차: 대중교통 배정
- [ ] Transit 네트워크 구조
- [ ] 노선, 정류장, 운행 간격 설정
- [ ] Extended Transit Assignment 알고리즘
- [ ] 요금 체계 설정
- [ ] 실습: 새 버스 노선 추가 및 배정

#### 7-8주차: 수요 모델 이해
- [ ] `demand.py`: 수요 로딩 메커니즘
- [ ] OMX 파일 형식
- [ ] 수요 모델 출력물 연동
- [ ] MSA (Method of Successive Averages)
- [ ] 실습: 외부 수요 파일 생성 및 로딩

#### 9-10주차: 고급 기능
- [ ] MAZ-to-MAZ 배정 (`highway_maz.py`)
- [ ] Path analysis: 경로 추적
- [ ] Matrix cache: 성능 최적화
- [ ] 병렬 처리 (num_processors)
- [ ] 실습: 특정 O-D 쌍의 경로 분석

### Phase 3: 실전 프로젝트 (6-8주)

#### 11-12주차: 시나리오 설계
- [ ] 프로젝트 정의: 분석 목표, 대안 설정
- [ ] 입력 데이터 준비: 네트워크, 토지이용, 인구
- [ ] 시나리오별 TOML 파일 작성
- [ ] 프로젝트: 실제 지역의 교통 계획 시나리오

#### 13-14주차: 모델 실행 및 검증
- [ ] 반복 계산 모니터링
- [ ] 수렴 판정 (gap, RMSE)
- [ ] 결과 검증: 관측 교통량과 비교
- [ ] 민감도 분석
- [ ] 프로젝트: 시나리오별 실행 및 문제 해결

#### 15-16주차: 결과 분석 및 시각화
- [ ] Python 분석 스크립트 작성
- [ ] 교통 성능 지표 계산
- [ ] 공간 분석: GIS 연동
- [ ] 시각화: 차트, 지도
- [ ] 프로젝트: 최종 보고서 작성

#### 17-18주차: 커스터마이징
- [ ] 새 Component 개발
- [ ] 기존 Component 수정
- [ ] 후처리 모듈 추가
- [ ] 테스트 작성
- [ ] 프로젝트: 특화 기능 구현

### Phase 4: 전문가 수준 (지속적)

#### 고급 주제
- [ ] **성능 최적화**: 
  - 병렬 처리 최적화
  - 메모리 관리
  - 네트워크 간소화 기법
  
- [ ] **고급 분석**:
  - 형평성 분석 (Equity Analysis)
  - 환경 영향 평가 (Emissions)
  - 경제성 분석 (Benefit-Cost Analysis)
  
- [ ] **통합 모델링**:
  - 토지이용-교통 통합 모델 (LUTI)
  - 동적 교통 배정 (DTA)
  - 실시간 교통 관리

- [ ] **오픈소스 기여**:
  - GitHub 이슈 해결
  - 새 기능 제안 및 구현
  - 문서화 개선

---

## 추가 학습 자료

### 공식 문서
- **TM2PY GitHub**: https://github.com/BayAreaMetro/tm2py
- **Emme API Reference**: Emme Desktop → Help → Modeller Reference
- **Bay Area Travel Model 2**: MTC 공식 문서

### 관련 논문 및 서적
- **교통 계획 이론**:
  - Ortúzar & Willumsen, "Modelling Transport" (4th ed.)
  - Sheffi, "Urban Transportation Networks"
  
- **Activity-Based Models**:
  - Bowman & Ben-Akiva, "Activity-Based Travel Forecasting"
  - PAS Report 588: "Activity-Based Travel Demand Models"

### 온라인 강좌
- **Python for Data Science**: Coursera, DataCamp
- **Traffic Flow Theory**: Transportation Research Board
- **GIS for Transportation**: Esri Training

### 커뮤니티
- **TRB (Transportation Research Board)**: ABM Committee
- **GitHub Issues**: 질문 및 토론
- **Emme User Forum**: INRO Community

---

## 결론 및 권장사항

### TM2PY의 강점
1. **확장성**: Python 기반으로 쉽게 커스터마이징 가능
2. **통합성**: Emme의 강력한 배정 엔진 활용
3. **현대적**: ABM 기반의 최신 교통 수요 예측 방법론
4. **오픈소스**: 투명하고 협업 가능한 개발 환경

### 학습 시 주의사항
1. **단계적 접근**: 한 번에 모든 것을 이해하려 하지 말고 순차적으로 학습
2. **실습 중심**: 코드를 직접 실행하고 수정하면서 체득
3. **문서화**: 자신이 이해한 내용을 노트에 정리
4. **커뮤니티 활용**: 막히는 부분은 GitHub Issues에 질문

### Transport Modeler로서의 활용
TM2PY는 단순한 도구를 넘어서, 현대 교통 계획의 **분석 플랫폼**입니다:
- **정책 평가**: 다양한 교통 정책의 효과를 정량화
- **의사결정 지원**: 데이터 기반의 합리적인 판단 근거 제공
- **시나리오 플래닝**: 미래 불확실성을 고려한 전략 수립
- **연구 개발**: 새로운 방법론을 테스트하고 검증하는 플랫폼

교통 모델러로서 TM2PY를 마스터하면:
- 복잡한 도시 교통 문제를 체계적으로 분석할 수 있습니다
- 정책 입안자에게 과학적 근거를 제시할 수 있습니다
- 지속가능한 교통 시스템 설계에 기여할 수 있습니다

**시작은 지금입니다!** 예제를 실행해보고, 코드를 읽고, 직접 수정해보세요.

---

*이 문서는 TM2PY v0.1.0 기준으로 작성되었습니다. 최신 버전은 공식 GitHub 저장소를 참조하세요.*

*문서 작성: 2024*
*라이선스: Apache 2.0*
