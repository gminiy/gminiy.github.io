---
layout: post
title: "[대규모 시스템 설계] 안정 해시"
date: 2024-04-09
categories: dev
tags: server
---

수평적 규모 확장시 요청과 데이터를 서버에 균등하게 나누기 위해 **안정해시**를 보편적으로 사용한다.

## 해시 키 재배치(rehash) 문제

N개의 캐시 서버가 있다고 하면 부하를 나누는 쉬운 방법은 모듈러를 사용하는 것이다.

- serverIndex = hash(key) % N
  ![](https://velog.velcdn.com/images/naljajm/post/619bd7a8-607b-4b19-a057-a13fb0b925b2/image.png)

이 방법은 서버 풀의 크기가 고정되어 있고, 데이터 분포가 균등할 때는 괜찮지만 모종의 이유로 서버가 풀의 개수가 변동되면 대부분 데이터의 서버 인덱스 값이 틀어지게 된다.
![](https://velog.velcdn.com/images/naljajm/post/2985d728-4c11-4e0a-9996-e87e74f2f60c/image.png)
![](https://velog.velcdn.com/images/naljajm/post/064059bb-ceff-4388-bfc5-11bc16533d2c/image.png)
결과적으로 대규모 캐시 미스가 발생하게 될 것이다.
이 문제를 효과적으로 해결하기 위해 **안정해시**를 사용한다.

## 안정 해시

해시 테이블 크기가 조정될 때 평균적으로 오직 k/n개의 키만 재배치하는 해시 기술이다. k는 키의 개수이고 n은 슬롯의 개수이다.

### 해시 공간과 해시 링

해시 함수는 SHA-1을 사용한다고 하면, 그 함수의 출력 값 범위는 0 ~ (2^160-1)이 된다. x0는 0, xn은 2^160-1 이면 x1부터 (xn -1) 까지는 그 사이값을 가지게 된다.
이 해시 공간을 다음과 같이 표현한다.
![](https://velog.velcdn.com/images/naljajm/post/7c01695f-1f2f-482e-95b0-6a210223f5e3/image.png)

이제 이 해시 공간의 양쪽을 구부려 접으면 다음과 같은 해시 링이 만들어진다.
![](https://velog.velcdn.com/images/naljajm/post/44534bb0-bdd5-4af7-b33f-e5f2b9734c42/image.png)

### 해시 서버

서버를 해시 링 위에 배치한다. 예시로 4개를 배치한다.
![](https://velog.velcdn.com/images/naljajm/post/dd078beb-de28-4f03-b323-002c2c1f48b3/image.png)

### 해시 키

캐시할 키 key0, key1, key2, ke3 를 해시 링 위에 배치한다.
![](https://velog.velcdn.com/images/naljajm/post/b5fcb914-7f42-463a-840c-9c8a17ca9af4/image.png)

### 서버 조회

키가 저장되는 서버는 해당 키의 위치로부터 시계방향으로 링을 탐색해 나가면서 만나는 첫 번째 서버다.
![](https://velog.velcdn.com/images/naljajm/post/b5dfbdeb-a104-4058-be3f-48e32f741303/image.png)

### 서버 추가

만약 server 4를 추가하게 되면 key0 만 재배치 되고 다른 키들은 재배치되지 않는다.
![](https://velog.velcdn.com/images/naljajm/post/bd2bbb52-2043-4dcd-b672-c0532269116e/image.png)

### 서버 제거

서버가 제거되면 키 가운데 일부만 재배치된다.
![](https://velog.velcdn.com/images/naljajm/post/e05e9aaa-25c5-4067-8b11-28d59ccf22e3/image.png)

### 기본 구현법의 두 가지 문제

기본 절차는 다음과 같다.

- 서버와 키를 균등 분포 해시 함수를 사용해 해시 링에 배치한다.
- 키의 위치에서 링을 시계 방향으로 탐색하다 만나는 최초의 서버가 키가 저장될 서버다.

이 방식에는 두가지 문제가 있다.

1. 서버 추가 및 삭제를 생각하면 파티션의 크기를 균등하게 유지하는 것이 불가능하다.
   ![](https://velog.velcdn.com/images/naljajm/post/362e8a1a-3e1e-40c3-96a0-879d02825177/image.png)

2. 키의 균등 분포를 달성하기가 어렵다 서버 1과 서버 3은 아무 데이터도 갖지 않지만 대부분의 키는 서버 2에 보관될 것이다.
   ![](https://velog.velcdn.com/images/naljajm/post/70b74654-e101-4b1c-957b-1e43147696b8/image.png)

이 문제를 해결하기 위해 가상노드가 나왔다.

### 가상 노드

가상 노드는 실제 서버를 가리키는 노드로서, 하나의 서버는 링 위에 여러 개의 가상 노드를 갖는다. 각 서버는 하나가 아닌 여러 개 파티션을 관리한다. 키의 위치로부터 시계방향으로 링을 탐색하다 만나는 최초의 가상 노드가 해당 키가 저장될 서버이다.
![](https://velog.velcdn.com/images/naljajm/post/d7760152-6ba9-4b21-91a4-e46a797670f3/image.png)

가상 노드 개수를 늘리면 표준 편차가 작아져서 데이터가 고르게 분포된다. 가상 노드 개수를 늘리면 표준 편차 값은 떨어지지만 가상 노드 데이터를 저장할 공간이 더 많이 필요하게 되므로 트레이드 오프가 필요하다.

### 재배치할 키 결정

서버가 추가되거나 제거되면 데이터 일부는 재배치해야 한다.
서버 4가 추가되었다고 해 보자. 이에 영향 받은 범위는 s4부터 그 반시계 방향에 있는 첫 번째 서버 s3까지이다.
![](https://velog.velcdn.com/images/naljajm/post/eec14a19-ae47-450e-8d12-c280dbf2cc97/image.png)

## 정리

안정 해시의 이점은 아래와 같다.

- 서버가 추가되거나 삭제될 때 재배치되는 키의 수가 최소화된다.
- 데이터가 보다 균등하게 배치되므로 수평적 규모 확장을 하기 좋다.
- 핫스팟 키 문제를 줄인다. 특정한 샤드에 대한 접근이 지나치게 빈번하면 서버 과부하 문제가 발생한다. 안정 해시는 데이터를 좀 더 균등하게 분배하므로 이런 문제가 생길 가능성을 줄인다.

---

_본 포스트는 알렉스 쉬 저자의 **가상 면접 사례로 배우는 대규모 시스템 설계 기초**를 기반으로 스터디하며 정리한 내용들입니다._

- [가상 면접 사례로 배우는 대규모 시스템 설계 기초](https://m.yes24.com/Goods/Detail/102819435)
