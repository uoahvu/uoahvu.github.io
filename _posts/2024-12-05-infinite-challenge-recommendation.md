---
layout: single
title: "PAIR-WISE 기반 취향 랭킹 학습으로 무한도전 콘텐츠 추천하기"
excerpt: "[Crawling Dataset] 콘텐츠 기반 임베딩 Retrieval 과 Pair-wise Ranking Model을 결합한 동영상 추천 시스템"
last_modified_at: 2024-12-05
toc: true
comments: true
categories: Recommendation
use_math: true
---

🎯 유저의 이력 데이터가 존재하지 않는 경우, 간단한 선호 조사를 기반으로 작동하는 추천시스템을 구현한다. 현재 시청중인 것과 유사도가 높은 순서대로 콘텐츠를 추천하되 각 콘텐츠를 표현하는 벡터는 유저 선호도에 따라 재학습된 것이다.

# 🖱️ Data Crawling

무한도전 콘텐츠를 웹크롤링으로 직접 수집하여 사용했으며, 동영상 시청 이력은 따로 존재하지 않는다.

[MBC 무한도전 다시보기 동영상](https://playvod.imbc.com/templete/VodList?bid=1000786100000100000)


> ![image](https://github.com/user-attachments/assets/17e4eacb-61ea-4ef9-be36-acabacf88cdc)
그림 1. 무한도전 다시보기
> ![image](https://github.com/user-attachments/assets/8ff2d1cb-1055-4fad-b7f2-00d5326bce0d)
그림 2. 무한도전 다시보기 - 회차별 설명

총 589개 회차의 제목과 방영일자, 영상길이 등 기본 메타정보를 수집했고, 각 회차 썸네일 클릭 시 확인가능한 설명 텍스트(그림2)를 추가로 수집했다.



# Content-based Embedding Model

## 전체 concat 구조

개별 임베딩 (텍스트, torch 임베딩..)


# Pair-Wise Ranking Model

## Triplet Margin Loss


# Recommendation Flow



# 🧑‍💻 Code

[🗂️ Github - Recommender system for Korean variety show "Infinite Challenge"](https://github.com/uoahvu/infinite-challenge-contents-recommendation)