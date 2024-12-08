---
layout: single
title: "PAIR-WISE 기반 취향 랭킹 학습으로 무한도전 콘텐츠 추천하기"
excerpt: "[Crawling Dataset] 콘텐츠 기반 임베딩 Retrieval 과 Pair-wise Ranking Model을 결합한 동영상 추천 시스템"
last_modified_at: 2024-12-08
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



# 🔎 Content-based Embedding Model (Retrieval)

>![image](https://github.com/user-attachments/assets/528b2a32-1eb9-42fb-b528-5371e2108788)
그림 3. 무한도전 콘텐츠 회차별 정보 데이터셋



## 🧑‍💻 Data Preprocessing

유사도 기반 검색을 위해 각 회차별 콘텐츠를 최적의 임베딩 벡터로 표현하기 위한 데이터 전처리 과정을 거쳤다.  

### 🧐 Feature 1 ) 무한도전 시즌

무한도전은 약 13년간 방영한 프로그램으로, 기간 내 크고작은 멤버 구성 변화가 잦았다. 멤버의 구성에 따라 회차별 선호도가 달라진다는 점을 반영하기 위해 방영일자별 시즌을 나누어 콘텐츠 벡터에 반영했다.

```python
# 방영일자 => 시즌
def date2season(x):
    periods = [
        ("2006-05-06", "2008-03-08", 1),  # 2006.05.06 ~ 2008.03.08
        ("2008-03-15", "2009-05-30", 2),  # 2008.03.15 ~ 2009.05.30 +전진 -하하
        ("2009-06-06", "2010-03-20", 3),  # 2009.06.06 ~ 2010.03.20 +길 -전진
        ("2010-03-27", "2012-01-31", 4),  # 2010.03.27 ~ 2012.01 +하하 (7인)
        # ("2012-02-01", "2012-07-14", 5),  # 2012.02 ~ 2012.07.14 파업
        ("2012-07-14", "2014-05-03", 5),  # 2012.07.14 ~ 2014.05.03 (7인)
        ("2014-05-03", "2014-11-15", 6),  # 2014.05.03 ~ 2014.11.08 -길
        ("2014-11-17", "2015-05-09", 7),  # 2014.11.17 ~ 2015.05.09 -홍철
        ("2015-05-09", "2015-11-12", 8),  # 2015.05.09 ~ 2015.11.12 +광희
        ("2015-11-12", "2016-04-09", 9),  # 2015.11.12 ~ 2016.04.09 -형돈
        ("2016-04-09", "2017-03-25", 10),  # 2016.04.09 ~ 2017.03.25 +세형
        ("2017-03-25", "2017-09-08", 11),  # 2017.03.25 ~ 2017.09.08 -광희
        ("2017-09-08", "2017-11-18", 12),  # 2017.09.08 ~ 2017.11.18 파업
        ("2017-11-25", "2018-03-31", 13),  # 2017.11.25 ~ 2018.03.31 +세호
    ]
    for start_date, end_date, season in periods:
        if x <= end_date and x >= start_date:
            return season
    return 0
```

멤버 구성을 기준으로 시즌을 정의했으며, 필자는 개인적으로 7인체제를 좋아한다.


### 👀 Feature 2 ) 특집회 여부

>![image](https://github.com/user-attachments/assets/e3b1e0a0-80aa-4a72-9ae2-7155d2ba1a27)
그림 4. 특집회

간혹 정규방영분이 아닌 이미 방영했던 것을 재구성하여 내보내는 등의 특집회가 존재했다. 이를 구분하기 위해 특집회라면 `1` 아니라면 `0`으로 구분하여 전처리했다.

```python
df["special"] = df["vod_num"].apply(lambda x: 1 if x == "특집회" else 0)
```


### 🎞️ Feature 3 ) 런타임

```python
def timestr2min(x):
    splited_time = x.split(":")
    return (
        float(splited_time[2]) / 60.0
        + float(splited_time[1])
        + float(splited_time[0]) * 60.0
    )
```

각 회차의 영상 길이를 확인할 수 있는데, `01:19:57` 와 같은 형태로 수집된다. 이를 분 단위의 실수(float) 타입으로 전처리 해주었다.

`01:19:57` 에서 '초' 는 60으로 나누고, '분'은 그대로, '시'에는 60을 곱해 더해준다. 즉, 0.95 + 19 + 60 = `79.95` 로 표현된다. 



### 🪄 Feature 4 ) 제목 및 설명



```python
# Description
df["description"] = df["description"].str.replace(
    r"[^a-zA-Z가-힣0-9 ]", "", regex=True
)

df["title_"] = df["title"].str.replace(r"[무한도전]", "")
```


각 회차의 특징을 표현하는 데 가장 중요한 단서를 제공하는 것은 텍스트라고 생각한다. 
제목 텍스트에서 공통적으로 등장하는 `"무한도전"` 은 삭제하고, 설명텍스트는 특수문자 등을 제외한 한글&영어&숫자 로만 구성되도록 전처리한다.


## 📖 Embedding Model

>![image](https://github.com/user-attachments/assets/7b72fe8a-7802-4ac9-90f4-62aaa23e34ee)
그림 5. 회차별 임베딩 벡터 구성도

전처리된 데이터는 임베딩 벡터로 표현된다. 시즌, 특집회 여부, 제목 및 설명 텍스트가 Feature로 사용되며 그림 5와 같은 임베딩 과정을 거친다.
시즌은 길이가 64인 임베딩 벡터로 표현하여 각 시즌의 특징이 학습될 수 있도록 한다. 특집 여부는 1과 0으로 표현되었으며, One-Hot 인코딩하여 표현한다.


### Sentence Transformer 기반 사전 학습 텍스트 임베딩 모델

```python
from sentence_transformers import SentenceTransformer

class ContentEmbedding(nn.Module):
    def __init__(self, hidden_size, num_season):
        super(ContentEmbedding, self).__init__()
        ...
        self.description_embedding = SentenceTransformer(
            "sentence-transformers/all-MiniLM-L6-v2"
        )
    def forward(self, x):
        ...
        # 제목텍스트
        embedded_title = torch.tensor(
            self.description_embedding.encode(x.loc[:, "title_"])
        )
        # 설명텍스트
        embedded_description = torch.tensor(
            self.description_embedding.encode(x.loc[:, "description"])
        )
        ...
```

위 코드는 콘텐츠 임베딩 모델의 일부이다. 제목 및 설명 텍스트를 임베딩하기 위해 sentence_transformers 패키지의 SentenceTransformer 모델을 사용한다.
그 중 [all-MiniLM-L6-v2](https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2) 은 다국어 데이터셋으로 사전학습된 모델로, 문장에 대한 384 차원의 임베딩 벡터를 도출할 수 있다.


## FAISS(Vector Store)


# Pair-Wise Ranking Model

## Triplet Margin Loss


# Recommendation Flow



# 🧑‍💻 Code

[🗂️ Github - Recommender system for Korean variety show "Infinite Challenge"](https://github.com/uoahvu/infinite-challenge-contents-recommendation)