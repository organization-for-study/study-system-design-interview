## 메세지 큐

> 메세지의 버퍼 역할을 담당하고, 무손실, 비동기 통신 기능을 지원하는 컴포넌트이다.

<img width="659" alt="스크린샷 2023-11-26 오전 11 58 42" src="https://github.com/organization-for-study/study-system-design-interview/assets/72437309/80e35642-a9f0-40fd-8a5e-c7cd782f56cb"><br/>

생산자(이하 pub)는 저장소(이하 큐)에 요청 메세지를 보내 저장시킨다.

소비자(이하 sub)는 여유가 될때마다 큐에 저장된 요청 메세지를 하나 하나씩 꺼내 처리한다.

### 버퍼 역할

- 요청 메시지를 저장하는 보관소 = 큐

### 무손실

- sub이 요청 메세지를 가져갈 때까지 계속 저장되어 있음 ⇒ 메세지 유실 x

### 비동기 통신

- pub, sub 중 하나가 죽어도 서로 각자의 역할은 수행할 수 있다.
- pub, sub은 느슨한 결합으로 서로 어떤 상태인지 몰라도 되기 때문에 독립적으로 확장 가능하다.
- 큐에 저장되는 메세지가 계속 늘어나면 sub을 확장시키고, 없으면 sub을 축소하는 전략을 세울 수 있다.

---

## 로그, 메트릭 그리고 자동화

> 사업 규모가 커지고 나서는 시스템 에러 핸들링과 통계 측정 및 시스템 자동화는 꼭 필요하다.

### 로그

- 에러 로그를 모니터링 하면 시스템 오류를 보다 쉽게 파악할 수 있음
- 프로메테우스나 그라파나 같이 로그를 단일 서비스로 모아주는 도구를 활용하면 편리하게 검색 조회 할수 있다.

### 메트릭

- 시스템의 현재 상태를 한눈에 파악하기 쉬움
- 지표 종류
    - 호스트(장비) - CPU, 메모리, 디스크 I/O
    - 종합 - DB 성능, 캐시 성능
    - 비즈니스 - 일/월별 능동 사용자(DAU, MAU) , 수익(revenue), 재방문(retention)

### 자동화

- 코드에 대한 자동 검증 → 에러 쉽게 판별
- 빌드, 테스트, 배포 등의 절차 자동화 → 개발 생산성 향상

---

## 데이터베이스의 규모 확장

> 데이터가 많아지면 DB에 부하가 증가하기때문에 이를 해소하기 위해서는 DB 확장이 필요하다. <br/>
> 크게 2가지가 방식이 있다.

### 수직적 확장

- 방법
    - 장비를 더 증설하기 (cpu, ram, 디스크 등)
- 예시
    - AWS - 24 TB ram을 가진 RDS
    - 스택오버플로 - 한대의 마스터 DB로만 처리함 (3만 DAU)
- 단점
    - 하드웨어는 한계가 있기 때문에 무한 증설할 수 없음 → 사용자가 계속 늘어나면 감당하기 어려운 수준이 반드시 옴
    - 단일 장애 지점(SPOF) → 하나의 장애 = 시스템 전체 장애
    - super expensive

### 수평적 확장 (샤딩)

- 수평적 확장은 샤딩이라고도 부른다
- 샤드의 사전적 의미는 **조각**이라는 뜻이고, 여기서는 분할된 DB를 의미한다.
- 방법
    - 아래 그림처럼 id를 나눌 샤드 개수에 맞게 할당
      <img width="468" alt="shard" src="https://github.com/organization-for-study/study-system-design-interview/assets/72437309/1bdf8a56-a1e5-4048-b5eb-2470fc08f12b">
- 고려사항
    - 샤딩키(a.k.a 파티션키)를 정할때는 데이터가 고르게 분산될 수 있도록 신경써야 한다 - 위 그림에서는 `user_id`
- 이슈사항
    - 재샤딩 (resharding)
        - 다음과 같은 상황에서 데이터 재정렬이 필요하다
            - 데이터가 너무 많아져 샤드가 더 필요할 때
            - 샤드 간 데이터 분포가 균등하지 못해 한 쪽 샤드가 빨리 데이터가 찰 때
        - 샤드 키를 계산하는 함수를 변경 및 데이터 재정렬
            - 안정 해시 기법을 활용하는 방법이 있다
    - 유명인사, 핫스팟 키 문제 (celebrity, hotspot key)
        - 유명인사와 관련된 키워드가 하나의 샤드에 집중 → 수많은 read 요청으로 해당 샤드에만 과부하
        - 골고루 부하가 분포되도록 서로 다른 샤드에 배치
    - 조인과 비정규화 (join and de-normalization)
        - 샤딩이 되어있는 테이블을 조인하기 쉽지 않기 때문에, 중복을 허용하고 한 테이블에서 쿼리가 수행되도록 비정규화하는 방법을 고려해 볼 수 있다.

---

## 백만 사용자, 그리고 그 이상

> 시스템 규모 확장은 지금까지 언급한 것(아래 과정들)의 지속 & 반복이다.

- 웹 계층 → 무상태 계층
- 모든 계층 → 다중화
- 많은 데이터를 캐시
- 여러 데이터 센터
- 정적 컨텐츠 → CDN
- 데이터 계층 → 샤딩
- 각 계층은 독립적서비스로
- 시스템 지속적 모니터링 및 자동화

어느 정도 규모는 위의 과정으로도 커버가 가능하지만,

더 나아가기 위해서는 시스템을 최적화하고 더 작은 단위의 서비스로 분할이 필요하며,

새로운 기술이 필요할 수 있다.

→ 위의 과정들은 새로운 기술을 위한 기본 배경지식과 자양분이 될 것
