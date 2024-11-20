---
layout: single
title: "소비자가 구매할 가능성이 높은 Top K 상품 추천모델링 (Session-based Recommendation)"
excerpt: "구매이력 데이터를 기반으로 유저가 구매할만한 상품을 제공하는 추천시스템"
last_modified_at: 2024-11-20
comments: true
categories: Recommendation
---

# Data Processing

## Retailrocket recommender system dataset

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

* 유저(visitorid) 152963 은 session 273 에서 상품 302559을 구매하고, 새로운 sessoin 274 번에서 상품 380196 을 둘러보며 시작한다. 

<br/>

---


>![image](https://github.com/user-attachments/assets/776f0b48-81a8-414a-a8a8-03007b2e3fe4)
그림 1. How to define a session in this project


필자는 하나의 세션을 "**구매를 완료할 때마다**" 로 정의했다. 상품 구매를 완료하면 소비자는 구매한 상품과는 먼, 또 다른 필요에 의해 상품을 탐색할 것이라는 전제이다. 그림 1은 각 세션을 표현한 것으로, 각 행동을 나타내는 직사각형의 가로 폭은 상품의 개수와 비례한다. 

즉 첫번째 세션의 경우 유저는 여러 상품을 보다가 장바구니에 상품을 담고 곧바로 구매로 이어졌다. 두번째 세션의 경우 상품을 둘러보는 중간에 장바구니에 상품을 담았고, 마지막에 마음에 드는 상품만 구매했다. 마지막 세션의 경우 따로 장바구니에 담지 않고 연이어 상품을 구매했다. 


>![image](https://github.com/user-attachments/assets/84069994-9c0e-4f60-a741-6ed9928c25b4)
그림 2. How to define inputs and outputs in a session

각 세션 내에서 타겟(Y) 상품은 유저가 원하는 상품, 구매할 가능성이 높은 상품이어야 하므로, 장바구니에 담은 상품(addtocart)과 구매한 상품(transaction)을 대상으로 진행한다. 그 외 유저가 본 상품(view) 을 input으로 하여 Y를 예측했다. 즉 **유저가 본 상품과 그 상품의 카테고리들의 시퀀스를 보고, 구매할 가능성이 높은 상품 n개를 추출**하는 것이다. 


> 🧐 <br/>
> 유저는 상품을 본 후 장바구니에 담거나 구매한다. 즉 X 안에 Y 가 포함될 수 있다. 이것을 문제 삼지 않은 이유는 유저가 보고 지나간 상품이더라도 한번 더 화면에 노출시키거나 알림을 보내 구매를 유도하는 방식으로도 추천시스템이 운영될 수 있기 때문이다. 


## Data Augmentation (데이터 증강)

>![image](https://github.com/user-attachments/assets/837995e6-207b-42eb-b631-cb66f4feb6a1)
그림 3. How to do data augmentation



현재 세션 내에서 유저의 필요 상품을 예측하기 위해서 유저가 보고 있는 상품들의 연관관계를 파악하는 것이 중요하다. 이를 위해 학습 데이터셋에 데이터 증강을 진행했는데, view 상품 이력의 소비 순서를 바꾼 시퀀스를 추가했다. 상품을 본 순서, 선후관계가 중요한 것이 아니라 세션 내의 상품들이 어떻게 비슷한 지를 모델링 하는 것이 중요하다고 판단했기 때문이다. 이는 BERT4Rec과 같은 양방향 학습을 통한 추천 모델의 성능이 일반적으로 높다는 아이디어를 차용했다. 



# Session Based Recommendations with LSTM


>![image](https://github.com/user-attachments/assets/c369f020-1d3d-4c0d-bff9-30778e571019)
그림 4. Session Based Recommender System Flow in this project

세션 내 추천시스템의 전체 흐름도는 위와 같다. 이 중 Candidate Generation 과 Top k에 따른 성능평가는 Test Set에서 수행되며, 학습 시에는 LSTM 기반의 추천모델을 사용한다. 


## Candidate Generation



## LSTM Structure





## Loss Function

처음에는 기본 CrossEntropyLoss를 사용하여 Loss 를 정의해 학습을 진행했으나, 여러 시도 끝에 최종적으로 `Hard Negative Loss`와 `Weighted Positive Loss`를 합쳐 사용했다.

`criterion = nn.CrossEntropyLoss()`

### Hard Negative Loss

```python
weights = torch.where((target == 0) & (outputs > 0.5), 1.0, 0.0)
loss = (weights * criterion(outputs, target)).mean()
```

이름에서 알 수 있듯, 어려운 Negative 샘플에 부여하는 Loss 이다. 어렵다는 것은 예측이 까다로운 샘플을 의미하며, 필자는 **실제 값이 0이지만 예측 값이 0.5 초과**한 데이터들을 대상으로 진행했다. 


### Weighted Positive Loss

```python
weights = torch.where(target == 1, 10.0, 0.0)
weighted_loss = torch.mean(
    (weights * criterion(outputs, target))
)
```

실제 값이 1인, 실제로 유저가 선호한 상품(Positive)을 대상으로 Loss에 가중치 10을 부여하는 방식이다. 처음에는 Negative 에도 가중치 1을 부여해 학습했으나, Negative 는 Hard Negative 샘플에 대한 Loss 만 부여하는 것이 성능이 나았다.



## Evaluation

### Recall@k, Precision@k


