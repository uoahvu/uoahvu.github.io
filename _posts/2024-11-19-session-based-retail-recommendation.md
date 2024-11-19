---
layout: single
title: "소비자가 구매할 가능성이 높은 Top K 상품 추천모델링 (Session-based Recommendation)"
comments: true
categories: Recommendation
---

## 문제 정의하기

### Retailrocket recommender system dataset

구매이력 데이터를 기반으로 유저가 구매할만한 상품을 제공하는 추천시스템을 구현한다.

[🗂️ Kaggle : Retailrocket recommender system dataset](https://www.kaggle.com/datasets/retailrocket/ecommerce-dataset)

Kaggle의 retailrocket 상품소비이력 데이터셋을 사용하며, 해당 데이터셋은 전자상거래 웹사이트에 방문한 유저의 행동 데이터(events.csv), 상품의 속성 데이터(item_properties.csv), 카테고리 트리 데이터(category_tree.csv)로 이루어져 있다.

필자는 오직 행동 데이터의 유저 행동 3가지(view, addtocart, transaction(구매))를 바탕으로 모델링했으며, 아이템의 속성으로는 아이템의 카테고리와 부모 카테고리를 포함했다. 



| idx     | timestamp           | visitorid | event   | itemid | transactionid   | session | category | parentid | 
|--------|---------------------|---------|--------------|---------|---------|------------|------------|------------|
| 194510 | 2015-09-18 00:42:47 | 152963  | view         | 302559  | NaN     | 273        | -1         | -1.0       | 
| 194511 | 2015-09-18 00:42:51 | 152963  | addtocart    | 302559  | NaN     | 273        | -1         | -1.0       | 
| 194512 | 2015-09-18 00:45:03 | 152963  | transaction  | 302559  | 2553.0  | 273        | -1         | -1.0       | 
| 194531 | 2015-09-18 02:04:16 | 152963  | view         | 380196  | NaN     | 274        | 973        | 20.0       | 
| 194532 | 2015-09-18 02:04:43 | 152963  | view         | 122984  | NaN     | 274        | 1493       | 594.0      | 
| 194533 | 2015-09-18 02:04:53 | 152963  | view         | 380196  | NaN     | 274        | 973        | 20.0       | 
| 194534 | 2015-09-18 02:05:13 | 152963  | view         | 122984  | NaN     | 274        | 1493       | 594.0      | 
| 194535 | 2015-09-18 02:05:21 | 152963  | view         | 380196  | NaN     | 274        | 973        | 20.0       | 
| 194536 | 2015-09-18 02:05:34 | 152963  | addtocart    | 380196  | NaN     | 274        | 973        | 20.0       | 
| 194537 | 2015-09-18 02:08:16 | 152963  | transaction  | 72462   | 5772.0  | 274        | 1493       | 594.0      |
| 194538 | 2015-09-18 02:08:16 | 152963  | transaction  | 12504   | 5772.0  | 274        | 1493       | 594.0      | 
| 194539 | 2015-09-18 02:08:16 | 152963  | transaction  | 380196  | 5772.0  | 274        | 973        | 20.0       | 
| 194540 | 2015-09-18 02:08:16 | 152963  | transaction  | 122984  | 5772.0  | 274        | 1493       | 594.0      | 
| 194541 | 2015-09-18 02:08:16 | 152963  | transaction  | 334401  | 5772.0  | 274        | 1043       | 226.0      |
| 194542 | 2015-09-18 02:12:38 | 152963  | view         | 342659  | NaN     | 275        | 1177       | 381.0      | 
| 194543 | 2015-09-18 02:34:21 | 152963  | view         | 366177  | NaN     | 275        | 808        | 1272.0     | 
| 194544 | 2015-09-18 02:35:07 | 152963  | view         | 362697  | NaN     | 275        | 1663       | 1398.0     | 
| 194545 | 2015-09-18 02:36:34 | 152963  | addtocart    | 362697  | NaN     | 275        | 1663       | 1398.0     | 
| 194546 | 2015-09-18 02:38:18 | 152963  | transaction  | 362697  | 5670.0  | 275        | 1663       | 1398.0     | 


위 데이터프레임은 유저가 특정 상품을 보거나(view), 장바구니에 담았거나(addtocart), 구매한(transaction) 행동데이터를 시간(timestamp) 순으로 정렬한 데이터이다. 상품(itemid)이 어떤 카테고리와 부모카테고리에 속해있는 지를 포함하도록 가공했다.


>![image](https://github.com/user-attachments/assets/776f0b48-81a8-414a-a8a8-03007b2e3fe4)
그림 1. How to define a session in this project


필자는 하나의 세션을 "**구매를 완료할 때마다**" 로 정의했다. 상품 구매를 완료하면 소비자는 구매한 상품과는 먼, 또 다른 필요에 의해 상품을 탐색할 것이라는 전제이다. 그림 1은 각 세션을 표현한 것으로, 각 행동을 나타내는 직사각형의 가로 폭은 상품의 개수와 비례한다. 즉 첫번째 세션의 경우 유저는 여러 상품을 보다가 장바구니에 상품을 담고 곧바로 구매로 이어졌다. 두번째 세션의 경우 상품을 둘러보는 중간에 장바구니에 상품을 담았고, 마지막에 마음에 드는 상품만 구매했다. 마지막 세션의 경우 따로 장바구니에 담지 않고 연이어 상품을 구매했다. 


>![image](https://github.com/user-attachments/assets/84069994-9c0e-4f60-a741-6ed9928c25b4)
그림 2. How to define inputs and outputs in a session

각 세션 내에서 타겟(Y) 상품은 유저가 원하는 상품, 구매할 가능성이 높은 상품이어야 하므로, 장바구니에 담은 상품(addtocart)과 구매한 상품(transaction)을 대상으로 진행한다. 그 외 유저가 본 상품(view) 을 input으로 하여 Y를 예측했다. 즉 **유저가 본 상품과 그 상품의 카테고리들의 시퀀스를 보고, 구매할 가능성이 높은 상품 n개를 추출**하는 것이다. 


## 데이터 증강 (Data Augmentation)

>![image](https://github.com/user-attachments/assets/837995e6-207b-42eb-b631-cb66f4feb6a1)



현재 세션 내에서 유저의 필요 상품을 예측하기 위해서 유저가 보고 있는 상품들의 연관관계를 파악하는 것이 중요하다. 이를 위해 학습 데이터셋에 데이터 증강을 진행했는데, view 상품 이력의 소비 순서를 바꾼 시퀀스를 추가했다. 상품을 본 순서, 선후관계가 중요한 것이 아니라 세션 내의 상품들이 어떻게 비슷한 지를 모델링 하는 것이 중요하다고 판단했기 때문이다. 이는 BERT4Rec과 같은 양방향 학습을 통한 추천 모델의 성능이 일반적으로 높다는 아이디어를 차용했다. 



## Session Based Recommendations with LSTM



![image](https://github.com/user-attachments/assets/c369f020-1d3d-4c0d-bff9-30778e571019)



## Hard Negative Loss + Weighted Loss



## Recall@k, Precision@k

