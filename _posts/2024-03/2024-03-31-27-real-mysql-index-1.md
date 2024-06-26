---
layout: post
title: "[MySQL] 인덱스: B-Tree 인덱스 - 1"
date: 2024-03-31
categories: dev
tags: database
---

# 인덱스란?

책의 마지막에 있는 "찾아보기"가 인덱스에 비유된다면, 책의 **내용**은 **데이터 파일**이라고 볼 수 있다. 책의 찾아보기 페이지를 통해 알 수 있는 **페이지 번호**는 데이터 파일에 저장된 **레코드의 주소**에 비유될수 있다.

데이터베이스 테이블의 모든 데이터를 검색해서 결과를 가지고 오는 것은 시간이 오래걸린다. 그래서 **칼럼의 값과 해당 레코드가 저장된 주소를 Key-Value로 인덱스**를 만들어 둔다.
책의 찾아보기도 내용이 많아지면 검색어를 찾는데 시간이 걸리기 때문에 이름 순서대로 정렬해두는 것처럼 DBMS의 인덱스도 **칼럼의 값을 주어진 순서로 미리 정렬**해서 보관한다.

- Sorted List: DBMS의 인덱스와 같은 자료구조, 저장되는 값을 항상 정렬된 상태로 유지하는 자료구조
- Array List: 데이터 파일과 같은 자료구조, 값을 저장되는 순서대로 그대로 유지하는 자료구조.

SortedList 자료구조는 데이터가 저장될 때마다 정렬해야 하므로 저장하는 과정이 느리지만, 빨리 원하는 값을 찾아올 수 있다.
**DBMS의 인덱스도 인덱스가 많은 테이블은 WRITE 문장 (INSERT, UPDATE, DELETE)은 느려지지만 SELECT를 빠르게 처리할 수 있다.**
테이블의 인덱스를 하나 더 추가할지 말지는 데이터의 저장 속도와 읽기속도 간의 타협을 찾아야 한다. 전부 인덱스로 생성하면 데이터 저장 성능이 떨어지고 인덱스 크기가 비대해져 역효과가 발생한다.

인덱스를 역할별로 구분한다면 Primary Key와 Secondary Key로 구분할 수 있다.

- Primary Key : 레코드를 대표하는 칼럼의 값으로 만들어진 인덱스. 레코드를 식별할 수 있는 기준값이기 때문에 식별자라고도 부른다. NULL이 허용이 안되며 중복이 허용되지 않는다.
- Secondary Key : PK를 제외한 모든 인덱스, 유니크 인덱스는 PK와 성격이 비슷하고 PK와 대체가 가능하기 때문에 대체 키라고도 부른다.

인덱스 저장 알고리즘별로 구분하는 것은 많지만 대표적으로 **B-Tree 인덱스**와 **Hash 인덱스**가 있다.

데이터 중복 허용 여부로 분류하면 Unique Index와 Non-Unique Index로 구분할 수 있다.
실제 DBMS의 쿼리를 실행하는 옵티마이저는 유니크한 값인지에 따라 처리 방식의 변화가 많다. 이는 추후에 다룬다.

인덱스의 기능별로 분류한다면 전문 검색용 인덱스나 공간 검색용 인덱스 등을 예로 들 수 있다. 추후에 살펴본다.

# B-Tree 인덱스

- 가장 일반적으로 사용되고 먼저 도입된 알고리즘이다.
- 여러 가지 변형 형태 알고리즘이 있는데 B+-Tree, B\*-Tree 가 사용된다.
- 바이너리 트리가 아니라 **Balanced Tree** 이다.
- 칼럼의 값을 변형시키지 않고 인덱스 구조체 내에서 항상 정렬된 상태로 유지한다.

## 구조 및 특성

- 트리 구조의 최상위에 "루트 노드"가 존재하고 그 하위에 "자식 노드"가 붙어 있는 형태
- 가장 하위에 있는 노드를 "리프 노드"라 하고 중간을 "브랜치 노드"라고 한다.
- 리프 노드는 항상 실제 데이터 레코드를 찾아가기 위한 주소 값을 가지고 있다.
  ![](https://velog.velcdn.com/images/naljajm/post/8787731d-d0be-4eaa-b5ba-793fed6bce68/image.png)

- 인덱스의 키값은 모두 정렬돼 있지만 데이터 파일의 레코드는 임의의 순서대로 저장돼 있다. (Insert 순서가 아니다)

## B-Tree 인덱스 키 추가 및 삭제

### 인덱스 키 추가

- 테이블의 스토리지 엔진에 따라 새로운 키값이 즉시 저장될 수도 있고 아닐 수도 있다.
- 저장될 키값을 이용해 B-Tree 상의 적절한 위치를 검색
- 위치가 결정되면 레코드의 키와 주소를 리프노드에 저장
- 리프 노드가 꽉 찼다면 리프 노드 분리. 이는 상위 브랜치 노드까지 범위가 넓어지면서 비용이 많이 들게 됨
- 인덱스 추가로 인한 INSERT, UPDATE 문장이 받는 영향

  - 테이블 레코드 추가 비용이 1이라고 가정
  - 인덱스 키를 추가하는 작업이 1 ~ 1.5 로 예측하는 것이 일반적
  - 인덱스가 3개 있다면, 인덱스가 하나도 없는 경우 1이고 3개인 경우 5.5 정도의 비용으로 예측
  - 비용의 대부분이 메모리나 CPU가 아니라 디스크로부터 인덱스 페이지를 읽고 쓰는 비용이기 때문에 시간이 오래걸림

- MyISAM, Memory 스토리지 엔진 사용시 INSERT 문장 실행시 즉시 새로운 키값을 B-Tree 인덱스에 반영하기 때문에 키 추가 작업이 완료될때까지 쿼리 수행이 완료되지 않는다.
- InnoDB 스토리지 엔진은 아래와 같이 상황을 판단하여 인덱스 키 추가 작업을 나중에 처리할지, 아니면 바로 처리할지 결정한다.

  - 사용자 쿼리 실행
  - 버퍼 풀에 새로운 키값을 추가해야 할 페이지가 존재한다면 즉시 추가 작업 처리
  - 버퍼 풀에 리프 노드가 없다면 인서트 버퍼에 추가할 키값과 레코드 주소를 임시로 기록해 두고 작업 완료
  - 백그라운드 작업으로 인덱스 페이지를 읽을 때마다 인서트 버퍼에 머지할 인덱스 키값이 있는지 확인한 후 인다면 병합
  - 데이터베이스 서버 자원의 여유가 생기면 조금씩 머지함

### 인덱스 키 삭제

- 해당 키값이 저장된 리프 노드를 찾아 삭제 마크하면 작업 완료
- 삭제된 인덱스 키 공간은 방치 혹은 재활용 할 수 있음.
- MySQL 5.5 이상 버전에서는 버퍼링되어 지연 처리 가능

### 인덱스 키 변경

- 키값이 변경되는 경우 단순히 인덱스상의 키값만 변경하는 것은 불가능
- 키값을 삭제한 후, 다시 새로운 키값을 추가하는 형태로 처리.

### 인덱스 키 검색

- 루트 노드부터 시작해서 브랜치 노드를 거쳐 리프 노드까지 이동하면서 비교 작업을 수행하는 **트리 탐색**을 한다.
- 100 % 일치 또는 값의 앞부분만 일치하는 경우에 사용할 수 있다.
- 인덱스의 키값에 변형이 가해진 후 비교되는 경우에는 B-Tree의 빠른 검색 기능을 사용할 수 없다.

---

_본 포스트는 이성욱 저자의 **Real MySQL**를 기반으로 스터디하며 정리한 내용들입니다._

- [Real MySQL](https://m.yes24.com/Goods/Detail/6960931)
