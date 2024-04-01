---
layout: post
title: "[대규모 시스템 설계] 시스템 설계 면접 공략법"
date: 2024-04-01
categories: dev
tags: server
---

저자는 이번 장에서 시스템 설계 면접에 대한 접근법을 설명하고 있다.
이번 장은 꼭 면접이 아니더라도 업무 중 시스템 설계를 팀원들과 진행할때 유용하게 사용할수 있을거라 생각한다.

# 효과적 면접을 위한 4단계 접근법

## 1단계 문제 이해 및 설계 범위 확정

엔지니어가 가져야 할 가장 중요한 기술 중 하나는 **올바른 질문을 하는 것, 적절한 가정을 하는 것, 시스템 구축에 필요한 정보를 모으는 것**이다.
그렇다면 어떤 질문을 해야할까?

- 구체적으로 어떤 기능을 만들어야 하나?
- 제품 사용자 수는 얼마나 되나?
- 회사의 규모는 얼마나 빨리 커지리라 예상하나?
- 회사가 주로 사용하는 기술 스택은 무엇인가? 설계를 단순화하기 위해 활용할 수 있는 기존 서비스로는 어떤 것들이 있는가?

### 예제

문제: 뉴스 피드 시스템을 설계하라

- 기능 정의
  - 질문 1: 모바일 앱, 웹 앱 가운데 어느 쪽을 지원해야 하나, 둘 다인가?
  - 질문 2: 뉴스 피드의 정렬 기준이 있는가? 각 피드에 특정 가중치를 부여해야 하는가?
  - 질문 3: 피드에 이미지나 비디오도 올라올 수 있는가?
- 예상 사용량 정의
  - 질문 1: 한 사용자는 최대 몇 명의 사용자와 친구를 맺을 수 있는가?
    - 질문 2: 트래픽 규모는 어느 정도인가?

## 2단계 개략적인 설계안 제시 및 동의 구하기

개략적인 초기 설계안을 제시한다.

- 설계안에 대한 최초 청사진을 제시하고 의견을 구하라.
- 화이트보드나 종이에 핵심 컴포넌트를 포함하는 다이어그램을 그려라. (클라이언트, API, 웹 서버, 데이터 저장소, 캐시, CDN, 메시지 큐...)
- 최소 설계안이 시스템 규모에 관계된 제약사항들을 만족하는지 계략적으로 계산해보라.
- 가능하다면 시스템의 구체적 사용 사례도 몇 가지 살펴보자.

### 예제

개략적으로 보자면 뉴스피드 설계는 피드 발행과 피드 생성 두 가지 처리 플로우로 나눠 생각해 볼 수 있다.

- 피드 발행 : 사용자가 포스트를 올리면 관련 데이터가 캐시/데이터베이스에 기록되고, 해당 사용자의 친구 뉴스 피드에 뜨게 된다.
- 피드 생성 : 사용자의 뉴스 피드는 사용자 친구들의 포스트를 시간 역순으로 정렬하여 표시한다.

![](https://velog.velcdn.com/images/naljajm/post/e8f28153-994e-4609-8e8d-afa934946c5b/image.png)

## 3단계 상세 설계

### 예제

이제 두가지 중요한 욜례를 보다 깊이 탐구한다.

1. 피드 발행
2. 뉴스 피드 가지고 오기

상세 설계는 11장에서 더 자세히 다룬다.

## 4단계 마무리

후속 질문 및 추가 논의를 진행한다.

- 시스템 병목구간, 개선 가능 지점을 찾아보자.
- 설꼐를 다시 한번 요약해보자.
- 오류 발생 케이스를 생각해보자
- 운영 이슈, 로그, 데이터 모니터링, 배포 전략들을 수립해보자
- 미래에 닥칠 규모 확장에 대한 대처를 다뤄보자.
- 세부적인 개선 사항들을 제안해보자.

---

_본 포스트는 알렉스 쉬 저자의 **가상 면접 사례로 배우는 대규모 시스템 설계 기초**를 기반으로 스터디하며 정리한 내용들입니다._

- [가상 면접 사례로 배우는 대규모 시스템 설계 기초](https://m.yes24.com/Goods/Detail/102819435)