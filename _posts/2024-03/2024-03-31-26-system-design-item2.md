---
layout: post
title: "[대규모 시스템 설계] 개략적인 규모 추정"
date: 2024-03-31
categories: dev
tags: server
---

## 개략적인 규모 추정

보편적으로 통용되는 성능 수치상에서 사고 실험을 행하여 추정치를 계산하는 행위로서, 어떤 설계가 요구사항에 부합할 것인지 보기 위한 것

개략적 규모 추정을 하기 위해 보편적으로 사용되는 수치들을 이해할 필요가 있다.

## 2의 제곱수

데이터 볼륨의 단위를 2의 제곱수로 표현하는 수치에 익숙해져야한다.

| 2의 x제곱 | 근사치 | 이름      | 축약형 |
| --------- | ------ | --------- | ------ |
| 10        | 1천    | 1Kilobyte | 1KB    |
| 20        | 1백만  | 1Megabyte | 1MB    |
| 30        | 10억   | 1Gigabyte | 1GB    |
| 40        | 1조    | 1Terabyte | 1TB    |
| 50        | 1000조 | 1Petabyte | 1PB    |

## 모든 프로그래머가 알아야 하는 응답 지연 값

구글 제프 딘이 2010년에 공개한 통상적인 컴퓨터에서 구현된 연산들의 응답지연 값

| 연산명                                                       | 시간                  |
| ------------------------------------------------------------ | --------------------- |
| L1 캐시 참조                                                 | 0.5ns                 |
| 분기 예측 오류(branch mispredict)                            | 5ns                   |
| L2 캐시 참조                                                 | 7ns                   |
| 뮤텍스(mutex) 락/언락                                        | 100ns                 |
| 주 메모리 참조                                               | 100ns                 |
| Zippy로 1 KB 압축                                            | 10,000ns = 10us       |
| 1 Gbps 네트워크로 2 KB 전송                                  | 20,000ns = 10us       |
| 메모리에서 1 MB 순차적으로 read                              | 250,000ns = 250us     |
| 같은 데이터 센터 내에서 메시지 왕복 지연시간                 | 500,000ns = 500us     |
| 디스크 탐색(seek)                                            | 10,000,000ns = 10ms   |
| 네트워크에서 1 MB 순차적으로 read                            | 10,000,000ns = 10ms   |
| 디스크에서 1 MB 순차적으로 read                              | 30,000,000ns = 30ms   |
| 한 패킷의 CA(캘리포니아)로부터 네덜란드까지의 왕복 지연 시간 | 150,000,000ns = 150ms |

이를 시각화한 도구
![](https://velog.velcdn.com/images/naljajm/post/f8de776d-b6f4-4342-a983-d9519ebfa811/image.png)

결론

- 메모리는 빠르지만 디스크는 아직도 느리다.
- 디스크 탐색(seek)은 가능한 피하라.
- 단순한 압축 알고리즘은 빠르다.
- 데이터를 인터넷으로 전상하기 전에 가능하면 압축하라.
- 데이터 센터는 보통 여러 지역에 분산되어 있고, 센터들 간에 데이터를 주고받는 데는 시간이 걸린다.

## 가용성에 관계된 수치들

- 고가용성(high availability)은 시스템이 오랜 시간 동안 지속적으로 중단 없이 운영될 수 있는 능력을 지칭하는 용어.
- % 로 표시하고, 100%는 시스템이 단 한 번도 중단된 적이 없었음을 의미. 보통 99 ~ 100%이다.
- SLA(Service Level Agreement)는 서비스 사업자와 고객 사이에 맺어진 합의를 의미. 서비스 사업자가 제공하는 서비스의 가용시간이 공식적으로 기술되어 있음.
- 9의 개수와 시스템 장애시간 사이의 관계
  ![](https://velog.velcdn.com/images/naljajm/post/b405216a-6641-47ca-a988-e71443ccacd1/image.png)

## 예제: 트위터 QPS와 저장소 요구량 추정

### 가정

- MAU는 3억 명이다.
- 50% 사용자가 트위터를 매일 사용
- 평균적으로 각 사용자는 매일 2건 트윗을 올림
- 미디어를 포함하는 트윗은 10%
- 데이터는 5년간 보관

### 추정

- **QPS(Query Per Second)** 추정
  - DAU = 3억 x 50% = 1.5억
    - QPS = 1.5억 x 2 / 24시간 / 3600초 ~= 3,500
    - 최대 QPS = 2 x QPS ~= 7,000
- 미디어 저장을 위한 저장소 요구량
  - 평균 트윗 크기: tweet_id 64B, 텍스트 150B, 미디어 1MB
  - 일간 미디어 저장소 요구량 : 1.5억(DAU) x 2(평균 2건 트윗) x 10% X 1MB = 30TB/일
  - 5년간 미디어 보관 요구량 : 30TB x 365 x 5 ~= 55PB

---

_본 포스트는 알렉스 쉬 저자의 **가상 면접 사례로 배우는 대규모 시스템 설계 기초**를 기반으로 스터디하며 정리한 내용들입니다._

- [가상 면접 사례로 배우는 대규모 시스템 설계 기초](https://m.yes24.com/Goods/Detail/102819435)