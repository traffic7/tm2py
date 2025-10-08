# TM2PY 교통모델러를 위한 코드 리뷰 (Code Review for Transport Modelers)

## 개요 (Overview)

TM2PY는 Bay Area Metro의 Travel Model 2 Python 패키지로, 교통 수요 예측 및 네트워크 분석을 위한 종합적인 프레임워크입니다. 이 문서는 교통계획 및 모델링 분야의 전문가들이 TM2PY를 효과적으로 학습하고 활용할 수 있도록 코드 구조, 주요 기능, 그리고 실무 적용 방안을 설명합니다.

## 주요 활용 방안 (Main Applications)

### 1. 도로 교통 수요 예측 (Highway Demand Forecasting)
- 통행배정(Traffic Assignment) 및 통행시간 산정
- 다양한 차종별 분석 (승용차, 트럭, HOV 등)
- 도로 용량 및 혼잡도 분석
- 통행료(Toll) 영향 분석

### 2. 대중교통 수요 예측 (Transit Demand Forecasting)
- 대중교통 통행배정
- 환승 분석 및 최적 경로 탐색
- 대중교통 접근성 분석
- 요금 체계 영향 분석

### 3. 네트워크 분석 (Network Analysis)
- 존(Zone) 간 통행시간 산출 (Skim Matrix)
- MAZ(Micro-Analysis Zone) 단위 세밀 분석
- 보행 및 자전거 등 비동력 수단 분석

## 아키텍처 및 핵심 구성요소 (Architecture and Key Components)

### 1. 제어 구조 (Control Structure)

#### RunController (tm2py/controller.py)
**역할**: 전체 모델 실행의 중앙 제어 장치

```python
# 사용 예시
from tm2py.controller import RunController
controller = RunController(['scenario.toml', 'model.toml'])
controller.run()
```

**핵심 기능**:
- 설정 파일(TOML) 로드 및 검증
- 컴포넌트 실행 순서 관리
- 반복 계산(Iteration) 제어
- Emme 프로젝트 관리

**학습 포인트**:
- `config`: 모든 모델 설정을 담고 있는 Configuration 객체
- `emme_manager`: Emme API와의 인터페이스
- `iteration`: 현재 반복 계산 단계 (균형 배정을 위한 반복)
- `completed_components`: 완료된 컴포넌트 이력

### 2. 구성 요소 기반 설계 (Component-Based Design)

#### Component (tm2py/components/component.py)
**역할**: 모든 모델 컴포넌트의 추상 기본 클래스

**핵심 개념**:
```python
class Component(ABC):
    def __init__(self, controller):
        self._controller = controller
    
    @abstractmethod
    def run(self):
        """각 컴포넌트가 반드시 구현해야 하는 실행 메서드"""
        pass
```

**컴포넌트 종류**:
- **네트워크 준비**: `PrepareNetwork`
- **도로 배정**: `HighwayAssignment`
- **대중교통 배정**: `TransitAssignment`
- **수요 준비**: `PrepareHighwayDemand`, `PrepareTransitDemand`
- **특수 배정**: `AssignMAZSPDemand` (MAZ 단위 최단경로 배정)

## 도로 교통 수요 예측 워크플로우 (Highway Demand Forecasting Workflow)

### 단계별 처리 과정

#### 1단계: 네트워크 준비 (Network Preparation)
**파일**: `tm2py/components/network/highway/highway_network.py`

**주요 작업**:
```python
class PrepareNetwork(Component):
    def run(self):
        # 링크 속성 생성
        self._create_class_attributes()
        # 통행료 설정
        self._set_tolls()
        # VDF(Volume Delay Function) 설정
        self._set_vdf_attributes()
        # 차종별 모드 설정
        self._set_link_modes()
```

**생성되는 네트워크 속성**:
- `@capacity`: 링크 용량
- `@free_flow_time`: 자유 통행 시간
- `@tollXX_YY`: 차종별 통행료
- `@ft`: 도로 기능 분류 (Functional Type)
- `@useclass`: 차종 제한 분류 (일반도로, HOV 전용 등)

**학습 포인트**:
- Emme 네트워크 속성의 명명 규칙 (@로 시작하는 사용자 정의 속성)
- 통행료 파일 구조 및 적용 방법
- VDF 함수와 용량 제약 조건

#### 2단계: 수요 로딩 (Demand Loading)
**파일**: `tm2py/components/demand/demand.py`

**주요 작업**:
```python
class PrepareHighwayDemand(PrepareDemand):
    def run(self):
        # OMX 파일에서 수요 행렬 읽기
        demand = self._read_demand()
        # MSA(Method of Successive Averages) 적용
        if iteration > 1:
            averaged_demand = self._apply_msa(demand)
        # Emmebank에 저장
        self._save_demand()
```

**OMX 파일 형식**:
- Open Matrix 형식의 수요 행렬
- 기종점(OD) 간 통행량 데이터
- 차종, 시간대, 목적별로 분리된 행렬

**학습 포인트**:
- OMX 파일 구조 및 접근 방법
- MSA 알고리즘의 의미 (균형 해로 수렴하기 위한 방법)
- Emme 행렬 관리 체계

#### 3단계: 통행 배정 (Traffic Assignment)
**파일**: `tm2py/components/network/highway/highway_assign.py`

**핵심 알고리즘**: SOLA (Second Order Linear Approximation)

**주요 작업**:
```python
class HighwayAssignment(Component):
    def run(self):
        # 수요 준비
        demand = PrepareHighwayDemand(self.controller)
        demand.run()
        
        for time_period in self.time_period_names():
            # Emme 시나리오 로드
            scenario = self.get_emme_scenario(time_period)
            
            # 배정 클래스 설정 (차종별)
            assign_classes = self._prepare_classes()
            
            # 통행배정 실행
            self._run_assignment(assign_classes)
            
            # 통행시간 행렬(Skim) 계산
            self._calculate_skims()
            
            # 결과 내보내기
            self._export_results()
```

**배정 클래스 (Assignment Classes)**:
- `DA`: 일반 승용차 (Drive Alone)
- `S2`: 2인 공유 차량
- `S3`: 3인 이상 공유 차량
- `TRUCK_*`: 다양한 트럭 유형

**학습 포인트**:
- 균형 배정(Equilibrium Assignment)의 개념
- SOLA 알고리즘의 작동 원리
- 경로 분석(Path Analysis)을 통한 통행시간 산출
- PCE(Passenger Car Equivalent) 변환
- 차종별 VOT(Value of Time) 적용

#### 4단계: Skim 행렬 생성
**생성되는 Skim 종류**:
- **시간(Time)**: 통행시간 (분 단위)
- **거리(Distance)**: 통행거리 (마일 단위)
- **통행료(Toll)**: 교량 통행료 및 가변 통행료 (센트 단위, 2010년 기준)
- **일반화 비용(Generalized Cost)**: VOT를 적용한 총 비용

**Intrazonal 처리**:
```python
def _set_intrazonal_values(self, matrix, method="nearest_neighbor"):
    """존 내부 통행(Intrazonal) 값 설정"""
    # 가장 가까운 이웃 존까지 거리의 절반 사용
```

### MAZ 단위 분석 (MAZ-to-MAZ Analysis)
**파일**: `tm2py/components/network/highway/highway_maz.py`

**목적**: TAZ보다 세밀한 MAZ 단위 통행 분석

**주요 기능**:
```python
class AssignMAZSPDemand(Component):
    """MAZ 간 최단경로 배정"""
    def run(self):
        # 네트워크 준비 (MAZ 접근 링크 포함)
        self._prepare_network()
        
        # 수요 그룹화 (효율적 처리)
        grouped_demand = self._group_demand()
        
        # 최단경로 계산 (Dijkstra 알고리즘)
        paths = self._run_shortest_path()
        
        # 네트워크에 통행량 할당
        self._assign_flow()
```

**학습 포인트**:
- MAZ vs TAZ의 차이점과 활용 사례
- 최단경로 알고리즘 (Dijkstra)
- 배경 교통량(Background Traffic) 처리

## 대중교통 수요 예측 워크플로우 (Transit Demand Forecasting Workflow)

### 구성 요소

#### 1. 대중교통 네트워크 구조
**설정 파일**: `tm2py/config.py - TransitConfig`

**주요 요소**:
```python
@dataclass
class TransitConfig:
    # 대중교통 모드 정의
    modes: Tuple[TransitModeConfig, ...]
    
    # 대중교통 차량 정의
    vehicles: Tuple[TransitVehicleConfig, ...]
    
    # 운임 관련
    fares_path: str
    use_fares: bool
    
    # 통행 인식 계수
    initial_wait_perception_factor: float  # 첫 대기 시간
    transfer_wait_perception_factor: float  # 환승 대기 시간
    walk_perception_factor: float  # 보행 시간
    
    # 환승 관련
    max_transfers: int
    initial_boarding_penalty: float
    transfer_boarding_penalty: float
```

#### 2. 대중교통 모드 (Transit Modes)
**유형**:
- `WALK`: 보행 접근/이동
- `ACCESS`: 접근 링크
- `EGRESS`: 진출 링크
- `LOCAL`: 버스, 지역 철도 등
- `PREMIUM`: 고속철도, BRT, 광역철도 등

**모드 설정 예시**:
```python
TransitModeConfig(
    type="LOCAL",
    assign_type="TRANSIT",
    mode_id="b",  # 단일 문자 ID
    name="Bus",
    in_vehicle_perception_factor=1.0  # 차내 시간 인식
)
```

#### 3. 대중교통 배정 (Transit Assignment)
**파일**: `tm2py/components/network/transit/transit_assign.py`

**배정 방법**: 최적 전략(Optimal Strategy) 기반
- 통행자는 여러 경로를 고려하여 기대 통행시간을 최소화
- 배차 간격(Headway)를 고려한 대기 시간 계산
- 환승 패널티 적용

**Skim 계산**:
```python
class TransitAssignment(Component):
    """대중교통 배정 및 Skim 계산"""
    # 생성되는 Skim 종류:
    # - 차내 시간 (In-Vehicle Time)
    # - 보행 시간 (Walk Time)
    # - 대기 시간 (Wait Time)
    # - 환승 횟수 (Transfers)
    # - 운임 (Fare)
    # - 총 일반화 비용 (Total Generalized Cost)
```

#### 4. 접근/진출 시간 (Connector Times)
**처리 방법**:
- TAZ에서 대중교통 정류장까지의 접근 시간
- 사전 계산된 보행 거리 기반
- 설정 파일에서 override 가능

**학습 포인트**:
- 대중교통 네트워크의 계층 구조 (노선, 정류장, 접근 링크)
- Headway 기반 대기 시간 계산
- 환승 모델링의 복잡성
- 운임 체계 통합 방법

## Emme 통합 (Emme Integration)

### EmmeManager (tm2py/emme/manager.py)
**역할**: Emme Desktop API의 중앙 집중식 관리

**주요 기능**:
```python
class EmmeManager:
    def project(self, project_path):
        """Emme 프로젝트 열기"""
        
    def emmebank(self, path):
        """Emmebank 접근"""
        
    def modeller(self, emme_project):
        """Modeller API 초기화"""
        
    def tool(self, namespace):
        """Emme 도구 접근"""
        # 예: "inro.emme.traffic_assignment.sola_traffic_assignment"
```

**Emme 주요 객체**:
- **Emmebank**: 시나리오, 네트워크, 행렬 데이터 저장소
- **Scenario**: 특정 시점의 네트워크 상태
- **Network**: 노드, 링크, 존, 경로 등의 네트워크 요소
- **Matrix**: 기종점 행렬 (수요, Skim 등)

**학습 포인트**:
- Emme 객체 모델 이해
- Emmebank vs 프로젝트 파일
- 시나리오 관리 전략
- 병렬 처리 설정 (`num_processors`)

## 설정 시스템 (Configuration System)

### TOML 설정 파일 구조
**파일**: `tm2py/config.py`

#### 1. scenario.toml
```toml
[scenario]
year = 2050
maz_landuse_file = "input/maz_data.csv"
verify = false
```

#### 2. model.toml
```toml
[run]
start_iteration = 0
end_iteration = 4
initial_components = ["prepare_network_highway"]
iterations = ["highway_maz_assign", "highway"]

[emme]
project_path = "emme_project"
highway_database_path = "database/highway.emmebank"
transit_database_path = "database/transit.emmebank"
num_processors = "MAX-1"

[time_periods.EA]
name = "Early AM"
emme_scenario_id = 1
highway_capacity_factor = 1.0
```

**학습 포인트**:
- TOML 문법 이해
- 시간대(Time Period) 정의 방법
- 반복 계산 구조 설정
- 데이터베이스 경로 관리

## 중점 학습 영역 (Key Learning Focus Areas)

### 1. 초급 (Beginner Level)
**목표**: TM2PY 실행 및 기본 개념 이해

**학습 순서**:
1. **설치 및 환경 설정**
   - Conda 환경 생성
   - Emme 패키지 설치
   - 예제 데이터 다운로드

2. **기본 실행**
   ```bash
   tm2py -s scenario.toml -m model.toml
   ```

3. **설정 파일 이해**
   - TOML 파일 구조
   - 시간대 정의
   - 컴포넌트 순서

4. **로그 분석**
   - 실행 로그 읽기
   - 에러 메시지 이해
   - 경고 메시지 해석

**주요 파일**:
- `README.md`: 설치 및 기본 사용법
- `docs/starting.md`: 시작 가이드
- `examples/`: 예제 설정 파일

### 2. 중급 (Intermediate Level)
**목표**: 네트워크 준비 및 배정 프로세스 이해

**학습 순서**:
1. **네트워크 데이터 구조**
   - Emme 네트워크 속성
   - 링크 타입 및 기능 분류
   - 존 체계 (TAZ, MAZ)

2. **수요 데이터**
   - OMX 파일 형식
   - 수요 행렬 구조
   - 차종 및 목적별 분류

3. **배정 알고리즘**
   - 균형 배정 이론
   - SOLA 알고리즘
   - MSA 수렴 방법

4. **Skim 행렬**
   - Skim 종류 및 용도
   - 경로 분석(Path Analysis)
   - 일반화 비용 계산

**주요 파일**:
- `tm2py/components/network/highway/highway_assign.py`
- `tm2py/components/demand/demand.py`
- `docs/architecture.md`

### 3. 고급 (Advanced Level)
**목표**: 코드 수정 및 확장

**학습 순서**:
1. **컴포넌트 아키텍처**
   - Component 추상 클래스
   - 새로운 컴포넌트 개발
   - 컴포넌트 간 데이터 전달

2. **Emme API 활용**
   - Network Calculator
   - Matrix 조작
   - 사용자 정의 도구 개발

3. **성능 최적화**
   - 병렬 처리
   - 메모리 관리
   - 캐싱 전략

4. **디버깅 및 검증**
   - 로깅 시스템
   - 중간 결과 검증
   - 단위 테스트 작성

**주요 파일**:
- `tm2py/controller.py`
- `tm2py/components/component.py`
- `tm2py/emme/manager.py`
- `tests/`: 테스트 코드 예제

## 실무 활용 사례 (Practical Applications)

### 1. 시나리오 분석
**사용 사례**: 새로운 도로/대중교통 노선의 영향 분석

**작업 흐름**:
1. 기본 시나리오 실행 (Base Case)
2. 네트워크 수정 (새 노선 추가)
3. 대안 시나리오 실행
4. Skim 행렬 비교 분석
5. 통행량 변화 분석

### 2. 통행료 정책 분석
**사용 사례**: 혼잡 통행료 도입 영향 평가

**필요한 수정**:
- `highway.tolls.file_path`: 통행료 데이터
- `@tollseg` 네트워크 속성
- VOT 설정 조정

### 3. MAZ 단위 접근성 분석
**사용 사례**: 대중교통역 주변 접근성 평가

**활용 컴포넌트**:
- `AssignMAZSPDemand`: MAZ 간 통행 배정
- `SkimMAZCosts`: MAZ 간 통행비용 산출

**분석 지표**:
- 특정 거리 내 도달 가능한 MAZ 수
- 평균 통행시간
- 통행비용 분포

### 4. 대중교통 서비스 개선 분석
**사용 사례**: 배차 간격 단축 효과 분석

**수정 방법**:
1. Transit 노선 파일에서 headway 조정
2. 대중교통 배정 재실행
3. 대기시간 및 통행시간 비교
4. 수요 변화 분석

### 5. 다중 시간대 분석
**시간대 구분 예시**:
- `EA`: Early AM (오전 3시-6시)
- `AM`: AM Peak (오전 6시-10시)
- `MD`: Midday (오전 10시-오후 3시)
- `PM`: PM Peak (오후 3시-7시)
- `EV`: Evening (오후 7시-자정)

**시간대별 분석 항목**:
- 혼잡도 변화
- 통행시간 변동
- 경로 선택 변화

## 데이터 입출력 (Data Input/Output)

### 입력 데이터
**필수 입력**:
1. **네트워크 데이터**
   - Emme 네트워크 파일 (.emp)
   - 링크 속성 (용량, 통행료 등)
   - 노드 및 존 정보

2. **수요 데이터**
   - OMX 파일
   - 기종점 행렬
   - 차종/목적/시간대별 분류

3. **설정 파일**
   - scenario.toml
   - model.toml
   - 통행료 참조 파일

4. **토지이용 데이터**
   - MAZ 단위 인구/고용 데이터
   - 접근 링크 정보

### 출력 데이터
**생성되는 결과**:
1. **Skim 행렬**
   - OMX 형식
   - 시간대별 파일
   - 차종별 통행시간/거리/비용

2. **배정 결과**
   - 링크별 교통량 (`volau`, `@flow_XX`)
   - 링크별 통행시간 (`timau`)
   - 경로 통계

3. **로그 파일**
   - 실행 로그
   - 수렴도 통계
   - 에러/경고 메시지

4. **검증 데이터**
   - VMT (Vehicle Miles Traveled)
   - VHT (Vehicle Hours Traveled)
   - 평균 통행시간

## 모범 사례 (Best Practices)

### 1. 프로젝트 구조
```
project_root/
├── input/
│   ├── maz_data.csv
│   ├── tolls.csv
│   └── demand/
│       ├── EA_demand.omx
│       └── ...
├── emme_project/
│   └── database/
│       ├── highway.emmebank
│       └── transit.emmebank
├── output/
│   └── skims/
├── scenario.toml
├── model.toml
└── logs/
```

### 2. 버전 관리
- 설정 파일을 Git으로 관리
- 네트워크 변경 이력 문서화
- 주요 결과 백업

### 3. 검증 절차
1. **입력 데이터 검증**
   - 수요 행렬 합계 확인
   - 네트워크 연결성 검사
   - 속성 값 범위 확인

2. **중간 결과 검증**
   - 반복 계산 수렴도 모니터링
   - Gap 값 확인
   - 경고 메시지 검토

3. **최종 결과 검증**
   - 관측 교통량과 비교
   - 합리성 검사 (평균 속도, VMT 등)
   - 전문가 리뷰

### 4. 성능 최적화
- `num_processors` 설정 최적화
- 불필요한 Skim 계산 제외
- MAZ 배정 범위 제한 (`max_dist`)
- 중간 결과 캐싱 활용

## 문제 해결 (Troubleshooting)

### 일반적인 문제

#### 1. Emme 패키지 import 오류
```python
ModuleNotFoundError: No module named 'inro'
```
**해결**: Emme Desktop에서 Modeller 패키지 재설치

#### 2. 배정 수렴 실패
**증상**: Gap이 목표치까지 감소하지 않음
**원인**:
- 네트워크 용량 부족
- VDF 파라미터 문제
- 수요 데이터 오류

**해결**:
- 최대 반복 횟수 증가
- 용량 검토 및 조정
- 수요 데이터 검증

#### 3. 메모리 부족
**증상**: Out of Memory 오류
**해결**:
- 시간대별 순차 실행
- 불필요한 행렬 삭제
- MAZ 배정 그룹 크기 축소

#### 4. 수행 시간 과다
**해결**:
- 병렬 처리 활성화
- 정밀도 요구사항 조정
- 네트워크 단순화 (불필요한 링크 제거)

## 추가 학습 자료 (Additional Resources)

### 문서
- [Emme API Reference](https://www.inrosoftware.com/): Emme 공식 문서
- `docs/architecture.md`: 아키텍처 다이어그램
- `docs/api.md`: API 문서

### 예제 코드
- `examples/`: 예제 설정 파일
- `notebooks/`: Jupyter Notebook 튜토리얼 (추가 예정)
- `tests/`: 단위 테스트 예제

### 관련 도구
- **Pandas**: 데이터 분석 및 조작
- **NumPy**: 행렬 연산
- **OpenMatrix (OMX)**: 수요 행렬 파일 형식
- **TOML**: 설정 파일 형식

## 결론 (Conclusion)

TM2PY는 현대적이고 확장 가능한 교통 수요 예측 프레임워크입니다. 이 문서에서 설명한 내용을 바탕으로 다음과 같은 학습 경로를 추천합니다:

1. **단계 1 (1-2주)**: 설치 및 예제 실행
2. **단계 2 (2-4주)**: 설정 파일 수정 및 간단한 시나리오 분석
3. **단계 3 (1-2개월)**: 네트워크 및 수요 데이터 구조 이해
4. **단계 4 (2-3개월)**: 배정 알고리즘 및 Emme API 학습
5. **단계 5 (지속적)**: 코드 수정 및 새로운 기능 개발

교통 계획 및 모델링 분야에서 TM2PY를 효과적으로 활용하기 위해서는:
- **도로 교통**: 균형 배정 이론, VDF, 통행료 모델링에 집중
- **대중교통**: 최적 전략, 환승 모델링, 운임 체계에 집중
- **공통**: Emme 네트워크 구조, OMX 데이터 형식, Python 프로그래밍에 대한 이해 필요

지속적인 학습과 실습을 통해 TM2PY를 마스터하고, 복잡한 교통 문제 해결에 활용할 수 있기를 바랍니다.
