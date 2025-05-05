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
그렇다면 옵티마이저는 무엇을 근거로 최적의 실행계획을 만들게 될까요?

## 옵티마이저의 종류
---
옵티마이저의 대표적인 종류는 크게 규칙기반(RBO)와 비용기반(CBO)로 나뉘게 됩니다.
1. 규칙기반 옵티마이저 / RBO(Rule-Based Optimizer)
- 규칙기반 옵티마이저(RBO)는 통계정보를 활용하지 않고 내부적으로 정해진 규칙에 따라 실행계획을 생성하는 옵티마이저입니다.
- 인덱스 구조나 연산자, 조건절의 형태 등이 우선순위를 결정하는 요소(규칙)로 작용하게 됩니다.
- 하지만 대용량 DBMS 현실적인 데이터 분포도나 업무적인 부분으로 인해 현재는 비용기반(CBO) 옵티마이저를 사용하고 있습니다.
  
2. 비용기반 옵티마이저 / CBO(Cost-Based Optimizer)
- 비용기반 옵티마이저(CBO)는 가장 최적의 비용을 계산하여 실행계획을 도출하는 옵티마이저입니다.
- 최적의 비용은 `데이터 사전(Data Dictionary)`에서 `통계 정보(Statistics)`를 기반으로 계산합니다.
- CBO의 내부 구성 흐름(SQL 최적화 과정)
![옵티마이저 최적화 구조도](assets/img/sql-tuning/optimizer-components.gif){: .normal }
_출처: Oracle 공식 문서 (https://docs.oracle.com/en/database/oracle/oracle-database/12.2/tgsql/)_
  1. Query Transformer(쿼리 변환기)
    - Paser에서 전달받은 SQL을 분석 후 더 나은 실행계획을 생성할 수 있도록 쿼리 구조를 변환합니다.
  2. Estimator(비용 평가)
    - `데이터 사전(Data Dictionary)`의 `통계 정보(Statistics)`를 참조하여 실행 계획의 비용을 계산합니다.
  3. Plan Generator(실행계획 생성기)
    - 실행 계획 중 가장 낮은 비용의 계획을 선택 후 `Row Source Generator`로 전달합니다.
  
>
**데이터 사전(Data Dictionary)이란?**  
데이터베이스를 운영 및 관리하기 위한 메타 데이터를 저장하는 시스템 테이블과 뷰의 집합을 말합니다.<br>
Data Dictionary에는 Object나 사용자 및 권한, 통계정보 등이 저장되어 있습니다.<br>
*USER_, ALL_, DBA_ 등의 VIEW로 제공됨(Oracle 기준)*
{: .prompt-tip}

- 

##

##

##

##

##

