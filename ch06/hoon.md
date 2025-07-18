## 키-값 저장소 개요

> ✅키-값 저장소(key-value store)  
> 키-값 데이터베이스라고도 불리는 비 관계형 (non-relational) 데이터베이스

이때 저장되는 값은 고유 식별자를 키로 가져야 한다. 키와 값 사이의 이런 연결 관계를 키-값 쌍이라고 지칭한다.

![image](https://github.com/user-attachments/assets/bf7ca496-a524-4993-9d4a-76879f6b5080)
↑요런식

다음 연산을 지원하는 키-값 저장소를 설계해보며 공부해보자.

-   put(key, value)： 키-값 쌍을 저장소에 저장한다.
-   get(key)： 인자로 주어진 키 에 매달린 값을 꺼 낸다.

## 요구사항 지정(특성 추출)

-   키-값 쌍의 크기는 10KB 이하이다.
-   큰 데이터를 저장할 수 있어야 한다.
-   높은 가용성을 제공해야 한다. 따라서 시스템은 설사 장애가 있더라도 빨리 응답해야 한다.
-   높은 규모 확장성을 제공해야 한다. 따라서 트래픽 양에 따라 자동적으로 서버 증설/삭제가 이루어져야 한다.
-   데이터 일관성 수준은 조정이 가능해야 한다.
-   응답 지연시간(latency)이 짧아야 한다.

## 단일 서버 키-값 저장소

단일 서버에서 키-값 저장소를 설계하는 가장 직관적인 방법은 키-값 쌍 전부를 메모리에 해시 테이블로 저장하는 것이다.

히자만 빠른 속도를 보장하긴 하지만 모든 데이터를 메모리에 저장이 불가능할 수도 있다는 약점이 있다.

## 분산 키-값 저장소

분산 키-값 저장소(분산 해시 테이블, 키-값 쌍을 여러 서버에 분산시킴)을 포함하는 분산 시스템을 설계할 때는 CAP(Consistency, Availability, Partition Tolerance theorem) 정리를 이해하고 있어야 한다.

### CAP 정리

CAP 정리는 데이터 일관성, 가용성, 파티션 감내(partition tolerance)의 3가지 요구사항을 동시에 만족하는 분산 시스템을 설계하는 것은 불가능하다는 정리이다.

-   데이터 일관성 : 분산 시스템에 접속하는 모든 클라이언트는 어떤 노드에 접속했느냐에 관계없이 언제나 같은 데이터를 보게 되어야 한다.
-   가용성 : 분산 시스템에 접속하는 클라이언트는 일부 노드에 장애가 발생하더라도 항상 응답을 받을 수 있어야 한다.
-   파티션 감내 : 파티션은 두 노드 사이에 통신 장애가 발생하였음을 의미한다. 파티션 감내는 네트워크에 파티션이 생기더라도 시스템은 계속 동작하여야 한다는 것을 뜻한다.

CAP 정리는 아래의 그림과 같이 어떤 두 가지를 충족하려면 나머지 하나는 반드시 희생되어야 한다는 것을 의미한다.

![image](https://github.com/user-attachments/assets/3a11a1e4-6a0a-44c0-b2f6-3f1a99d88f55)


어느 두 가지를 만족하는 부분인 각 교집합은 다음과 같다.

-   CP 시스템 : 일관성과 파티션 감내를 지원하는 키-값  저장소. 가용성을 희생한다.
-   AP 시스템 : 가용성과 파티션 감내를 지원하는 키-값 저장소. 데이터 일관성을 희생한다.
-   CA 시스템 : 일관성과 가용성을 지원하는 키-값 저장소. 파티션 감내는 지원하지 않는다. 하지만 통상 네트워크 장애는 피할 수 없는 일로 여겨지므로, 분산 시스템은 반드시 파티션 문제를 감내할 수 있도록 설계되어야 한다. 그러므로 실세계에 CA 시스템은 존재하지 않는다.

#### 이상적 상태

이상적 환경이라면 네트워크가 파티션되는 상황은 절대로 일어나지 않고, 파티션 문제가 발생하면 일관성과 가용성 사이에서 하나를 선택해야 한다.

-   파티션 문제: 네트워크 문제로 인해 서버 간 통신이 불가능해지는 상황을 의미합니다. (그림 6-3에서 n3가 n1, n2와 통신 불가) 이 경우 데이터 불일치(오래된 사본)가 발생할 수 있다.
-   **일관성을 선택하는 시스템 (CP 시스템)**: 데이터 불일치를 피하기 위해 쓰기 연산을 중단하여 가용성을 희생한다. 은행권 시스템과 같이 데이터 일관성이 중요한 경우 (온라인 뱅킹 시스템이 최신 계좌 정보를 출력하지 못하는 문제) 파티션 문제가 해결될 때까지 오류를 반환한다.
-   **가용성을 선택하는 시스템 (AP 시스템)**: 오래된 데이터를 반환할 위험이 있더라도 계속 읽기/쓰기 연산을 허용합니다. 파티션 문제 해결 후 새로운 데이터를 동기화한다.

![image](https://github.com/user-attachments/assets/b73c479d-e41e-4c08-98ab-e0d54df779df)


### 시스템 컴포넌트

구현에 사용될 핵심 컴포넌트들 및 기술들은 다음과 같다.

-   데이터 파티션
-   데이터 다중화
-   일관성
-   일관성 불일치 해소
-   장애 처리
-   시스템 아키텍처 다이어그램
-   쓰기 경로
-   읽기 경로

#### 데이터 파티션

대규모 어플리케이션의 경우 전체 데이터를 한 대 서버에 욱여 넣는 것은 불가능하다.

가장 단순한 해결책은 데이터를 작은 파티션들로 분할한 다음 여러 대 서버에 저장하는 것이다. 이때, 다름 두가지 문제를 중요하게 따져봐야 한다.

-   데이터를 여러 서버에 고르게 분산할 수 있는가
-   노드가 추가되거나 삭제될 때 데이터의 이동을 최소화할 수 있는가

이때 안정 해시를 사용하여 해결할 수 있다.

![image](https://github.com/user-attachments/assets/876b8a0a-70ad-418e-9789-7be935e6f0cf)


장점은 다음과 같다.

-   규모 확장 자동화 : 시스템 부하에 따라 서버가 자동으로 추가되거나 삭제되도록 만들 수 있다.
-   다양성 : 각 서버의 용량에 맞게 가상 노드의 수를 조정할 수 있다.

#### 데이터 다중화

높은 가용성과 안정성을 확보하기 위해서는 데이터를 N(N은 튜닝 가능한 값)개 서버에 비동기적으로 다중화할 필요가 있다.

어떤 키를 해시 링 위에 배치한 후, 그 지점으로부터 시계 방향으로 링을 수노히하면서 만나는 첫 N개 서버에 데이터 사본을 보관하는 것이다.

가상 노드를 사용한다면 선택된 N개의 노드가 대응될 실제 물리 서버의 개수가 N보다 작아질 수 있다. 따라서 노드를 선택할 때 같은 물리 서버를 중복 선택하지 않도록 해야 한다.

#### 데이터 일관성

여러 노드에 다중화된 데이터는 적절히 동기화 되어야 한다. 정족수 합의(Quorum Consensus) 프로토콜을 사용하면 읽기/쓰기 연산 모두에 일관성을 보장할 수 있다.

-   N = 사본 개수
-   W = 쓰기 연산에 대한 정족수. 쓰기 연산이 성공한 것으로 간주되려면 적어도 W개의 서버로부터 쓰기 연산이 성공했다는 응답을 받아야 한다.
-   R = 읽기 연산에 대한 정족수. 읽기 연산이 성공한 것으로 간주되려면 적어도 R개의 서버로부터 응답을 받아야 한다.

**\*N = 3인 경우**

![image](https://github.com/user-attachments/assets/45824c01-cb1c-458e-8d27-c1bb4ec9556e)


몇 가지 예시

-   R=i, W = N： 빠른 읽기 연산에 최적화된 시스템
-   W=1, R=N： 빠른 쓰기 연산에 최적화된 시스템
-   W+R>N： 강한 일관성이 보장됨(보통 N = 3, W = R=2)
-   W + R <= N : 강한 일관성이 보장되지 않음

#### 일관성 모델

-   강한 일관성(strong consistency) : 모든 읽기 연산은 가장 최근에 갱신된 결과를 반환한다. 다시 말해서 클라이언트는 절대로 낡은(out-of-date) 데이터 를 보지 못함.
-   약한 일관성(weak consistency) : 읽기 연산은 가장 최근에 갱신된 결과를 반환하지 못할 수 있음.
-   결과적 일관성(eventual consistency) : 약한 일관성의 한 형태로, 갱신 결과가 결국에는 모든 사본에 반영(즉, 동기화)되는 모델.

#### 장애 처리

**장애 감지**

분산 시스템에서 서버 하나가 죽었다고해도 바로 장애처리 하지는 않는다. 두 대 이상의 서버가 똑같이 서버 A의 장애를 보고해야 해당 서버에 실제로 장애가 발생했다고 간주하게 된다.

모든 노드 사이에 멀티캐스팅 채널을 구축하는 것이 서버 장애를 감지하는 가장 손쉬운 방법이다. 하지만 서버가 많을 때는 분명 비효율적이다.

![image](https://github.com/user-attachments/assets/8cf90b76-4e24-4a23-b779-1d82a067af28)


**일시적 장애 처리**

느슨한 정족수(sloppy quorum) 접근법은 이 조건을 완화하여 가용성을 높인다. 접근법을 이용해서 가용성을 높일 수 있다. 정족수 요구사항을 강제하는 대신, 쓰기 연산을 수행할 W개의 건강한 서버와 읽기 연산을 수행할 R개의 건강한 서버를 해시 링에서 고른다.(장애서버는 무시)

장애 상태인 서버로 가는 요청은 다른 서버가 잠시 맡아 처리한다. 이런 장애 처리 방안을 단서 후 임시 위탁 기법이라 부른다.

**영구 장애 처리**

그럼 영구적인 노드의 장애 상태는 어떻게 처리해야 할까? 반-엔트로피 프로토콜을 구현하여 사본들을 동기화할 수 있다.

사본 간의 일관성 이 망가진 상태를 탐지하고 전송 데이터 의 양을 줄이기 위해서는 머클(Merkle) 트리를 사용한다.

**데이터 센터 장애 처리**

데이터 센터 장애에 대응할 수 있는 시스템을 만들려면 데이터를 여러 데이터 센터에 다중화하는 것이 중요하다.

**시스템 아키텍처 다이어그램**

다음을 바탕으로 다이어그램을 그려보자.

-   클라이언트는 키-값 저장소가 제공하는 두 가지 단순한 API, 즉 get(key) 및 put(key, value)와 통신한다.
-   중재자(coordinator)는 클라이언트에게 키-값 저장소에 대한 프락시(proxy) 역할을 하는 노드다.
-   노드는 안정 해시 (consistent hash) 의 해시 링 (hash ring) 위 에 분포한다.
-   노드를 자동으로 추가 또는 삭제할 수 있도록, 시스템은 완전히 분산된다(decentralized).
-   데이터는 여러 노드에 다중화된다.
-   모든 노드가 같은 책 임을 지므로, SPOF(Single Point of Failure)는 존재하지 않는다.

![image](https://github.com/user-attachments/assets/e14e8b42-76a6-402e-b4c2-cf078206aac5)


**쓰기 경로**

카산드라의 사례를 참고하여 쓰기 요청이 특정 노드에 전달되면 무슨 일이 벌어지는지를 그려보면 다음과 같다.

![image](https://github.com/user-attachments/assets/67000a45-32af-4721-8ce6-a2f5c6d3c93b)


① 쓰기 요청 이 커밋 로그(commit log) 파일에 기록된다.  
② 데이터가 메모리 캐시에 기록된다.  
③ 메모리 캐시가 가득차거나 사전에 정의된 어떤 임계치에 도달하면 데이터는 디스크에 있는 SSTable에 기록된다. SSTable은 Sorted-String Table의약어로,〈키, 값〉의 순서쌍을 정렬된 리스트 형태로 관리하는 테이블이다.

**읽기 경로**

읽기 요청을 받았을 때 데이터를 클라이언트에 다음과 같이 반환한다.

![image](https://github.com/user-attachments/assets/edb61eb0-e602-4a80-951c-893c5fba3eca)


## 요약

![image](https://github.com/user-attachments/assets/d9fa461d-2731-4bfb-981c-40b38e6fefe4)
