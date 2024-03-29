## 14장 유튜브 설계

## 1. 문제 이해 및 설계 범위 확정

- DAU 5백만
- 한 사용자 하루 평균 5개 비디오 시청
- 10%의 사용자가 하루에 1개의 비디오 업로드 → 평균 크기 300MB
- 비디오 저장을 위한 매일 새로 요구되는 저장용량 
- CDN 비용 고려필요

## 2. 계략적 설계안

![1](https://github.com/organization-for-study/study-system-design-interview/assets/126097518/3d62c2b9-e12e-4cdd-b2fb-af2eff4d1f0f)


- 사용자 단말: 컴퓨터, 폰, 스마트 TV 등을 통해 유튜브를 시청할 수 있음.
- CDN: 비디오 재생 시 CDN으로부터 스트리밍이 이루어짐
    - 넷플릭스: AWS (CloudFlare)
    - 페이스북: 아카마이 CDN사용
- API 서버: 비디오 스트리밍을 제외한 모든 요청은 API 서버가 처리. 피드 추천, 비디오 업로드 URL 생성, DB 및 캐시 갱신, 사용자 가입 등을 수행.

### 2-1. 비디오 업로드

![비디오 업로드](https://github.com/organization-for-study/study-system-design-interview/assets/126097518/2bbd6540-9ce4-4d9e-9d15-72bb85e069b6)


1. 비디오를 원본 저장소에 저장함.
2. 트랜스 코딩 서버는 원본 저장소에서 비디오를 가져와 트랜스코딩을 수행.
3. 트랜스 코딩 완료 후
    1. 완료된 비디오를 트랜스 코딩 저장소로 업로드. → CDN에 업로드.
    2. 완료 이벤트를 트랜스코딩 완료 큐에 넣고 완료 핸들러는 완료 큐에서 각 이벤트를 꺼내 메타데이터 DB와 캐시를 업데이트.
4. API 서버가 단말에게 스트리밍 준비가 되었다고 알림

[ 알고가기 ]

*트랜스코딩: 비디오 트랜스코딩은 해상도, 인코딩, 비트 전송률과 같은 매개 변수를 조정하여 비디오 파일을 한 형식에서 다른 형식으로 변환하는 프로세스.

⇒ 자세한 내용은 하단 링크 참고.

https://aws.amazon.com/ko/what-is/video-transcoding/

*트랜스코딩 서버: 비디오의 포맷(MPEG, HLS 등)을 변한해서 단말이나 대역폭에 맞는 최적의 비디오 스트림을 제공하기 위해 필요.

*트랜스코딩과 인코딩

비디오 인코딩은 비디오 데이터를 압축하여 품질에 영향을 주지 않으면서 파일 크기를 줄임. 이는 비디오 트랜스코딩 프로세스의 한 단계이지만 대규모 트랜스코딩 파이프라인과 독립적으로 수행할 수도 있음. ⇒ 데이터 압축과 관련.

트랜스코딩은 비디오의 형식, 코덱, 비트레이트, 해상도 또는 기타 주요 특성을 변경하는것.

### 2-2. 비디오 스트리밍

![비디오 스트리밍](https://github.com/organization-for-study/study-system-design-interview/assets/126097518/502e819c-52b4-4f67-be87-63d3e72bf7d1)


비디오 스트리밍은 CDN을 통해 실시간으로 이루어짐.

각 단말기마다 가장 가까운 CDN을 사용하기 때문에 전송지연에 대한 문제는 크게 일어나지 않음.

주의해야 할 점: 요청 프로토콜마다 지원하는 비디오 인코딩과 플레이어가 다르다는 것

## 3. 세부설계

### 3-1. 비디오 트랜스코딩

특정 단말에서 생성한 비디오가 다른 단말에서도 원활하게 재생되려면, 다른 단말과 호환되는 비트 레이트 저장되어야 함.

트랜스 코딩의 목적성은 아래와 같다고 할 수 있음.

- 효율적인 저장공간 사용 → 비용절감
- 단말 간의 호환성
- 비디오 품질 최적화

**DAG (Directed Acyclic Graph): 유향 비순환 그래프 모델.**

병렬성을 높이기 위해 적절한 수준의 추상화가 필요.

![DAG](https://github.com/organization-for-study/study-system-design-interview/assets/126097518/ee575973-2583-4c76-981a-f210993b751c)


**트랜스코딩 아키텍처**

![트랜스코딩 아키텍처](https://github.com/organization-for-study/study-system-design-interview/assets/126097518/09bdae1c-2070-40a4-8672-3839321ece22)


- **전처리기**
    - 비디오 분할
    - DAG 생성
    - 데이터 캐시: 분할된 비디오의 메타데이터를 임시 저장소에 저장
- **DAG 스케줄러**
- **자원 관리자**
    - 세 개의 큐(작업 큐, 작업 서버 큐, 실행 큐)와 작업 스케줄러로 구성됨
    - 작업 관리자는 작업 큐에서 가장 높은 우선순위의 작업을 꺼내고 작업 실행을 지시.
- **작업 서버**
- **임시 저장소**
    - 어떤 시스템을 선택할 것이냐는 저장할 데이터의 유형, 크기, 이용 빈도, 데이터 유효기간 등에 따라 달라짐. → 참조 횟수가 많기 때문에 메모리에 캐시해두는 것이 좋음.
- **인코딩된 비디오**

### 3-2. 최적화

- **속도 최적화**: 비디오 병렬 업로드, 병렬 프로세스.
- **비용 최적화**
    - 인기 비디오만 CDN을 통해 재생하고 그렇지 않은 비디오들은 원본 그대로 재생.
    - 짧은 비디오라면 필요할 때 인코딩해서 재생.
    - 특정 비디오는 특정 지역의 CDN에만 저장.

### 3-3. 에러처리

## 4. 마무리

- API 계층의 규모 확장성 확보 방안 : API 서버는 무상태 서버이므로 수평적 규모 확장이 가능
- 데이터베이스 계층의 규모 확장성 확보 방안 : 데이터베이스의 다중화와 샤딩 방법 논의
- 라이브 스트리밍 : 라이브 스트리밍은 비디오를 실시간으로 녹화하고 방송하는 절차
- 비디오 삭제: 저작권을 위반한 비디오, 선정적 비디오, 불법적 행위에 관계된 비디오는 삭제.
