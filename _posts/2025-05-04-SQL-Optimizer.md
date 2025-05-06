---
layout : post
title : SQL 옵티마이저에 대한 이해
date : 2025-05-04 22:55:30 +0900
categories : [SQL Tuning]
tags : [옵티마이저, Optimizer, Oracle]
description : 옵티마이저는 DBMS에서 SQL의 최적의 실행 계획을 생성하는 중요한 기능입니다.<br>옵티마이저가 실행 계획을 결정하고 생성하는 과정에 대해서 정리한 글입니다.
---

## 옵티마이저란?
---
Oracle에서 옵티마이저는 사용자가 SQL 실행을 요청했을 때 처리하는 과정에서 해당 SQL에 대한 최적의 실행계획을 생성하는 오라클의 핵심 기능 입니다.  
모든 SQL은 DBMS에서 최초로 라이브러리 캐시에 올라가기 전 `하드파싱` 단계를 거치게됩니다.  
이 때 옵티마이저는 데이터에 접근하는 가장 효율적인 방법*(=가장 낮은 비용)*을 결정하고 실행계획을 생성하는 역할을 수행합니다.
그렇다면 옵티마이저는 무엇을 근거로 최적의 실행계획을 만들게 될까요?

## 옵티마이저의 종류
---
옵티마이저는 크게 규칙기반(RBO)과 비용기반(CBO)로 구분됩니다.

### 규칙기반 옵티마이저 / RBO(Rule-Based Optimizer)
- 규칙기반 옵티마이저(RBO)는 통계정보를 활용하지 않고 내부적으로 정해진 규칙에 따라 실행계획을 생성하는 옵티마이저입니다.
- 인덱스 구조나 연산자, 조건절의 형태 등이 우선순위를 결정하는 요소(규칙)로 작용하게 됩니다.
- 하지만 대용량 DBMS 환경이나 실제 데이터 분포 등을 고려한 실행계획이 필요해짐에 따라 현재는 대부분 비용기반(CBO) 옵티마이저를 사용하고 있습니다.
  
### 비용기반 옵티마이저 / CBO(Cost-Based Optimizer)
- 비용기반 옵티마이저(CBO)는 가장 최적의 비용을 계산하여 실행계획을 도출하는 옵티마이저입니다.
- 최적의 비용은 `데이터 사전(Data Dictionary)`에서 `통계 정보(Statistics)`를 기반으로 계산합니다.
- CBO의 내부 구성 흐름(SQL 최적화 과정)<br>
 -![옵티마이저 최적화 구조도](assets/img/sql-tuning/optimizer-cbo.png)
_Oracle 공식 문서의 내용을 참고하여 직접 재구성한 도식입니다. (https://docs.oracle.com/en/database/oracle/oracle-database/12.2/tgsql/)_

  1. Query Transformer(쿼리 변환기)
    - Parser에서 전달받은 SQL을 분석 후 쿼리 형식을 변환하는 것이 효율적인지를 검토하고 필요하다고 판단되면 쿼리의 형식을 변환합니다.
  2. Estimator(비용 평가기)
    - 실행 계획의 전체 비용을 결정하는 구성요소 입니다.
    - `데이터 사전(Data Dictionary)`의 `통계 정보(Statistics)`를 참조하여 실행 계획의 비용을 계산합니다.
    - 이 때 통계 정보를 바탕으로 `선택도(Selectivity)`, `카디널리티(Cardinality)`, `비용(Cost)`을 산정하여 전체 실행계획의 비용을 추정합니다.
  3. Plan Generator(실행계획 생성기)
    - 다양한 엑세스 경로 및 조인 방법/순서 등을 바탕으로 다양한 실행 계획을 탐색합니다.
    - 실행 계획 중 가장 낮은 비용의 계획을 선택 후 `Row Source Generator`로 전달합니다.
  
>
**데이터 사전(Data Dictionary)이란?**  
Data Dictionary는 데이터베이스의 운영 및 관리를 위해 사용되는 메타데이터를 저장하는 시스템 테이블 및 뷰의 집합을 말합니다.<br>
Data Dictionary에는 Object나 사용자 및 권한, 통계정보 등이 저장되어 있습니다.<br>
*USER_, ALL_, DBA_ 등의 VIEW로 제공됨(Oracle 기준)*
{: .prompt-tip}

## 옵티마이저의 비용 계산 원리
---
옵티마이저가 비용을 계산하는 과정은 DBMS 내부적으로 이루어지고 복잡한 과정을 거치기 때문에 전부 학습하기는 쉽지 않습니다.<br>
하지만 SQL튜닝 및 안정적인 DBMS운영을 위해서는 옵티마이저가 통계정보를 활용해서 어떻게 비용을 산정하는지 정도는 간략하게라도 알고 있어야합니다.

### 통계정보(Statistics)
Oracle 공식문서를 보면 비용기반옵티마이저(CBO)의 비용평가기(Estimator)가 통계정보를 참조해서 실행계획의 비용을 계산한다고 되어 있습니다. 즉, 옵티마이저는 통계정보를 기반으로 비용을 계산하기 때문에 실행계획에 있어서 통계정보는 매우 중요한 요소입니다.<br>
통계정보는 `DBMS_STATS` 패키지를 통해 수집 및 갱신할 수 있습니다.<br>
만약 통계정보를 갑자기 전부 삭제하거나 오랫동안 갱신하지 않던 통계정보를 재수집하거나 한다면 옵티마이저는 잘못된 판단으로 실행계획을 생성하게 될 수도 있습니다.<br>
이는 SQL의 처리 성능이 갑자기 느려지거나 심한 경우 장애 상황이 발생할 수도 있음으로 통계정보를 효율적으로 관리하는 전략이 필요합니다.

#### 통계정보의 종류
1. 오브젝트 통계정보
   - 테이블
      ```SQL
      SELECT * FROM DUAL;
      ```
      |통계항목|설명|
      |--|--|
      |NUM_ROWS|테이블에 저장된 총 레코드 수|
      |BLOCKS|사용된 익스텐트에 속한 총 블록 수(실제 테이블에 할당된 총 블록수는 `DBA_SEGMENTS`, `USER_SEGMENTS` 에서 확인 가능)|
      |AVG_ROW_LEN|레코드당 평균 길이(Bytes)|
      |SAMPLE_SIZE|샘플링한 레코드 수|
      |LAST_ANALYZED|통계정보 수집일|

   - 인덱스
      ```SQL
      SELECT * FROM DUAL;
      ```
      |통계항목|설명|
      |--|--|
      |BLEVEL|브랜치 레벨, 인덱스 루트에서 리프 블록 도달까지 읽게되는 블록의 수(깊이)로 값이 클 수록 I/O가 증가함|
      |LEAF_BLOCKS|인덱스 리프 블록의 총 수|
      |NUM_ROWS|인덱스에 저장된 총 레코드 수|
      |DISTINCT_KEYS|인덱스(KEY)값의 조합으로 만들어지는 값의 종류 개수, 인덱스 키값을 모두 `=` 조건으로 조회할 때 `선택도`를 계산하는데 사용됨|
      |AVG_LEAF_BLOCKS_PER_KEY|인덱스의 키값을 모두 `=` 조건으로 조회할 때 읽게 되는 리프 블록의 수|
      |AVG_LEAF_DATA_PER_KEY|인덱스의 키값을 모두 `=` 조건으로 조회 할 때 읽게 되는 테이블 블록의 수|
      |CLUSTERING_FACTOR|인덱스 키값을 기준으로 테이블의 데이터가 모여있는 정도|

   - 컬럼
      ```SQL
      SELECT * FROM DUAL;
      ```
      |통계항목|설명|
      |--|--|
      |NUM_DISTINCT|중복을 제외한 컬럼 값 종류의 개수(NVD, Number of Distinct Values)|
      |DENSITY|컬럼 값의 분포 정도로 `선택도`를 계산할 때 사용됨<br> 히스토그램이 없거나 균등하게 분포된 경우 1/NUM_DISTINCT(NVD) 값과 같음|
      |AVG_COL_LEN|컬럼 평균 길이(Bytes)|
      |LOW_VALUE|컬럼 값 중 가장 작은 값|
      |HIGH_VALUE|컬럼 값 중 가장 큰 값|
      |NUM_NULLS|컬럼 값 중 NULL값인 레코드의 수|
      - 히스토그램<br>
        ```SQL
        SELECT * FROM DUAL;
        ```
        |유형|설명|
        |--|--|
        |FREQUENCY(도수분포)||
        |HEIGHT-BALANCED(높이균형)||
        |TOP-FREQUENCY(상위도수분포)||
        |HYBRID(하이브리드)||
2. 시스템 통계정보

### 선택도(Selectivity)와 카디널리티(Cardinality)

### 비용(Cost)

##

##

##

