# TM2PY 한국어 문서

## 코드 리뷰 및 학습 가이드

Transport Modeler를 위한 종합 학습 가이드가 준비되어 있습니다:

📚 **[코드 리뷰: Transport Modeler를 위한 학습 가이드](code_review_korean.md)**

### 문서 내용

이 가이드는 TM2PY를 처음 접하는 교통 모델러부터 고급 사용자까지 모두를 위한 내용을 담고 있습니다:

#### 1. 기초 개념
- TM2PY 소개 및 Activity-Based Model(ABM) 이해
- 핵심 아키텍처: Controller, Component, Configuration
- Python 및 Emme 기초

#### 2. 실무 활용
- **도로 수요 예측**: Highway Assignment, MAZ-to-MAZ 배정, 균형 배정 알고리즘
- **대중교통 수요 예측**: Transit Assignment, 요금 체계, 승객 행동 모델링
- **시나리오 분석**: 혼잡 통행료, 신규 노선, 자율주행차 등

#### 3. 학습 자료
- 18주 단계별 학습 로드맵
- 실전 활용 예제 (코드 포함)
- 추가 학습 자료 및 참고 문헌

### 주요 활용 방안

#### 도로 교통 (Highway)
```python
# 도로 배정 실행
from tm2py.controller import RunController

controller = RunController(
    ["scenario.toml", "model.toml"]
)
controller.run()
```

- 신규 도로 건설 효과 분석
- 통행료 정책 시뮬레이션
- 교통량 예측 및 혼잡도 분석
- 존간 통행 시간 스킴 생성

#### 대중교통 (Transit)
```python
# 대중교통 노선 및 요금 설정
[transit]
use_fares = true
value_of_time = 10.0
max_transfers = 3
```

- 신규 노선 효과 분석 (BRT, 지하철 등)
- 대중교통 수요 예측
- 환승 최적화 분석
- 서비스 품질 평가

### 시작하기

1. **설치**: README.md의 설치 가이드 참조
2. **예제 실행**: 
   ```bash
   get_test_data <location>
   tm2py -s scenario.toml -m model.toml
   ```
3. **학습**: [한국어 코드 리뷰 문서](code_review_korean.md) 읽기
4. **실습**: 예제 수정 및 시나리오 분석

### 추천 학습 순서

초보자라면 다음 순서로 학습하는 것을 권장합니다:

1. 📖 기본 개념 및 아키텍처 이해 (1-2주)
2. 🛠️ Python 및 Emme 기초 익히기 (2-3주)
3. 🚗 도로 배정 실습 (2-3주)
4. 🚌 대중교통 배정 실습 (2-3주)
5. 📊 실전 프로젝트 수행 (4-6주)

### 도움이 필요하신가요?

- 📝 **문서**: [전체 한국어 가이드](code_review_korean.md)
- 💬 **커뮤니티**: [GitHub Issues](https://github.com/BayAreaMetro/tm2py/issues)
- 📧 **공식 문서**: [영문 문서](../README.md)

---

*이 문서는 교통 계획 및 모델링 전문가를 위해 작성되었습니다.*
