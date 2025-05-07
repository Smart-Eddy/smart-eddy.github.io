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
  ![옵티마이저 최적화 구조도](assets/img/sql-tuning/optimizer-cbo.png)
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
통계정보는 `DBMS_STATS` 패키지를 통해 수집, 갱신, 삭제할 수 있습니다.<br>

```sql
/** DBMS_STATS 패키지 주요 명령어 */

-- 1. TABLE
BEGIN
DBMS_STATS.GATHER_TABLE_STATS(
    OWNNAME          => 'SCOTT'  -- 소유자(스키마)명
  , TABNAME          => 'EMP'  -- 테이블명
  , ESTIMATE_PERCENT => DBMS_STATS.AUTO_SAMPLE_SIZE -- 샘플링 비율
  , METHOD_OPT       => 'FOR ALL COLUMNS SIZE AUTO' -- 컬럼/히스토그램 수집 방식
  , CASCADE          => TRUE -- 인덱스 통계도 함께 수집
);
END;

-- 2. INDEX
BEGIN
DBMS_STATS.GATHER_INDEX_STATS(
    OWNNAME => 'SCOTT'  -- 소유자(스키마)명
  , INDNAME => 'EMP_IDX1'  -- 인덱스명
);
END;

-- 3. SCHEMA
BEGIN
DBMS_STATS.GATHER_SCHEMA_STATS(
    OWNNAME          => 'SCOTT'  -- 소유자(스키마)명
  , ESTIMATE_PERCENT => DBMS_STATS.AUTO_SAMPLE_SIZE -- 샘플링 비율
  , METHOD_OPT       => 'FOR ALL COLUMNS SIZE AUTO' -- 컬럼/히스토그램 수집 방식
  , CASCADE          => TRUE -- 테이블 및 인덱스 통계도 함께 수집 
);
END;

-- 참고 : METHOD_OPT 파라미터 option에 따라 히스토그램 수집 방식이 달라진다.
```

만약 수집된 통계정보를 갑자기 전부 삭제하거나 오랫동안 갱신하지 않던 통계정보를 재수집하거나 한다면 옵티마이저는 기존과는 다른 판단으로 실행계획을 생성하게 될 수도 있습니다.<br>
이는 SQL의 처리 성능이 갑자기 느려지거나 최악의 경우 장애 상황이 발생할 수도 있음으로 통계정보를 효율적으로 관리하는 전략이 필요합니다.

#### 통계정보의 종류
1. 오브젝트 통계정보
   - 테이블  
   - 
      - NUM_ROWS : 테이블에 저장된 총 레코드 수
      - BLOCKS : 테이블이 사용하는 블록 수(사용된 익스텐트 기준)
      - AVG_ROW_LEN : 각 레코드 별 평균 길이(Bytes 단위)
      - SAMPLE_SIZE : 샘플링한 레코드 수

   - 인덱스
   - 
      - BLEVEL : 브랜치 레벨, 인덱스 루트에서 리프 블록 도달까지 읽게되는 블록의 수(브랜치의 깊이)
      - LEAF_BLOCKS : 인덱스 리프 블록의 수
      - NUM_ROWS : 인덱스에 저장된 레코드 수
      - DISTINCT_KEYS : 인덱스(KEY)값의 조합으로 만들어지는 값의 종류 개수로 `=` 조건으로 조회할 때 `선택도(Selectivity)`를 계산할 때 사용된다.
      - AVG_LEAF_BLOCKS_PER_KEY : 평균 리프 블록의 수
      - AVG_LEAF_DATA_PER_KEY : 평균 테이블 블록의 수
      - CLUSTERING_FACTOR : 인덱스 키값을 기준으로 테이블의 데이터가 모여있는 정도()

   - 컬럼
   - 
      - NUM_DISTINCT : 중복을 제외한 컬럼 값 종류의 개수(NVD, Number of Distinct Values)
      - DENSITY : 컬럼 값의 분포 정도(컬럼 `=` 조건 검색 시 `선택도(Selectivity)`를 미리 구해놓은 값, 히스토그램이 없는 경우 1/NUM_DISTINCT(NVD) 값과 같음)
      - AVG_COL_LEN : 컬럼 평균 길이(Bytes 단위)
      - LOW_VALUE/HIGH_VALUE : 컬럼 값의 최소/최대 값
      - NUM_NULLS : 컬럼 값이 NULL인 레코드의 수

      >
      **컬럼 히스토그램이란?**  
      컬럼의 `=` 조건에 대한 `선택도(Selectivity)`는 컬럼 통계 중 미리 구해놓은 `DENSITY`를 이용하거나 `1/NUM_DISTINCT` 공식으로 구할 수 있습니다.<br>
      컬럼의 데이터 분포가 균일한 경우 위의 공식으로 잘 맞지만, 데이터 분포가 규일하지 않은 경우에는 위의 공식이 잘 맞지 않습니다.<br>
      `선택도(Selectivity)`를 제대로 구하지 못할 경우 비용 계산이 제대로 되지 않을 수 있고 이는 최적의 실행계획을 만들지 못하게 됩니다.<br>
      오라클에서는 데이터 분포가 균일하지 않은 컬럼의 선택도 계산을 위해 `히스토그램`이라는 통계를 추가적으로 활용합니다.<br>
      히스토그램은 실제 컬럼 값별로 데이터를 읽고 데이터의 비중 및 빈도를 미리 계산해놓은 통계정보입니다.<br>
      히스토그램의 유형으로 `FREQUENCY(도수분포)`, `HEIGHT-BALANCED(높이균형)`, `TOP-FREQUENCY(상위도수분포)`, `HYBRID(하이브리드)`가 있습니다.<br>
      히스토그램을 사용하지 않는 경우 `NONE`으로 해당 컬럼의 데이터 분포도가 균일하다고 볼 수 있습니다.<br>
      *히스토그램 통계정보 수집은 `DBMS_STATS`패키지로 통계정보를 수집할 때 METHOD_OPT 파라미터로 지정할 수 있습니다.*
      {: .prompt-tip}


2. 시스템 통계정보
   - 시스템 통계정보는 애플리케이션 및 하드웨어의 성능에 관한 통계정보입니다.
   - 

### 선택도(Selectivity)

### 카디널리티(Cardinality)

