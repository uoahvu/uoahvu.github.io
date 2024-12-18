---
layout: single
title: "PAIR-WISE 기반 취향 랭킹 학습으로 무한도전 콘텐츠 추천하기"
excerpt: "[Crawling Dataset] 콘텐츠 기반 임베딩 Retrieval 과 Pair-wise Ranking Model을 결합한 동영상 추천 시스템"
last_modified_at: 2024-12-11
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

### - Feature 1 ) 무한도전 시즌

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


### - Feature 2 ) 특집회 여부

>![image](https://github.com/user-attachments/assets/e3b1e0a0-80aa-4a72-9ae2-7155d2ba1a27)
그림 4. 특집회

간혹 정규방영분이 아닌 이미 방영했던 것을 재구성하여 내보내는 등의 특집회가 존재했다. 이를 구분하기 위해 특집회라면 `1` 아니라면 `0`으로 구분하여 전처리했다.

```python
df["special"] = df["vod_num"].apply(lambda x: 1 if x == "특집회" else 0)
```


### - Feature 3 ) 런타임

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



### - Feature 4 ) 제목 및 설명



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


### - Sentence Transformer : 사전 학습 텍스트 임베딩 모델

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


## 📂 FAISS(Vector Store)

```python
class ContentEmbedding(nn.Module):
    ...
    def inference(self, x):
        output = self(x)
        index = faiss.IndexFlatL2(output.size(-1))
        index.add(output.detach().numpy())
        return index, output


def candidate_generator(index, query, k):
    D, I = index.search(query.reshape(1, -1), k=k)

    return D, I
```

각 콘텐츠를 임베딩 벡터로 표현했다면, FAISS Vector Store에 저장하여 사용한다. 이 후 추천 시 유저가 시청하고 있는 콘텐츠와 유사한 콘텐츠를 검색하기 위해 `candidate_generator` 함수를 사용하여 search 할 수 있다.

<br/>

---

# 🤼 Pair-Wise Ranking Model

## 🗒️ Preference Survey

본 추천시스템의 구현 전제는 유저의 이력 데이터가 존재하지 않는 경우이므로 간단한 선호 조사를 기반으로 작동하도록 했다. 유저에게 한쌍의 콘텐츠를 제시하면, 유저는 둘 중 더 선호하는 콘텐츠를 선택한다.


>![image](https://github.com/user-attachments/assets/53e10836-b264-43b3-95b2-1a590b4734a9)
그림 6. Example of Preference Survey

둘 다 별로인 경우는 학습에서 제외하고, 하나 이상 좋다고 대답한 경우를 학습에 포함했다. (그림 6의 1,2,3) 선호도 조사 결과는 아래 설명할 `Triplet Margin Loss` 에 적용된다.


## 👩‍👧‍👦 Triplet Margin Loss


본 추천시스템의 목적은 유저의 취향을 반영하여 콘텐츠 벡터를 재학습하고, 콘텐츠 벡터가 단순히 내용만이 아닌 유저의 취향까지 내포하도록 하는 것이다.


>![image](https://github.com/user-attachments/assets/130766af-85ef-402f-9afd-0e24ceed5f3f)
그림 7. How to do Learning To Rank

```python
anchor_embeddings = self.embeddings(torch.tensor(anchors, dtype=torch.long))
positive_embeddings = self.embeddings(torch.tensor(positives, dtype=torch.long))
negative_embeddings = self.embeddings(torch.tensor(negatives, dtype=torch.long))

loss = triplet_loss(anchor_embeddings, positive_embeddings, negative_embeddings)
```

해당 로직을 반영하기 위해 **Triplet Margin Loss** 를 사용했다. Triplet Margin Loss는 기준이 되는 Anchor 벡터와 같은 그룹의 벡터와 가까워지도록 학습되고, 다른 그룹의 벡터와는 멀어지도록 학습된다.
이 때 기준은 선호하는 콘텐츠이며, 이와 같은 그룹은 내용 유사도가 높은 콘텐츠, 다른 그룹은 선호하지 않는 콘텐츠로 정의했다. 


>![image](https://github.com/user-attachments/assets/ab863ccc-6438-4825-b5e1-4cb200cf4021)
그림 8. How to apply Triplet Margin Loss

예를 들어, 유저가 <돈가방을 찾아라!> 를 <서해안 고속도로 가요제> 보다 더 재밌게 시청했다고 답한 경우, 
기준이 되는 <돈가방을 찾아라!> 콘텐츠는 그와 유사한 콘텐츠(100빡빡이의 습격 ...)들과 가까워지도록 학습되며 그와 반대로 선택하지 않은 <서해안 고속도로 가요제>를 포함하여 유사한 콘텐츠와는 멀어지도록 학습된다.


그 결과, 유저가 실제로 좋아할 만한 콘텐츠끼리는 서로 더 가까워지도록 하는 것이다. 그래서 만약 비슷한 내용을 갖는 콘텐츠라도, 유저의 선택을 받지 못했다면 추천 순위가 밀려나게 설계된다.

> 🧐 그런데, 만약 현재 콘텐츠가 사용자가 싫어하는 콘텐츠라면? 
> 
> 오히려 Positive 콘텐츠가 추천순위에서 밀려나게 되는 것이다(벡터 유사도가 낮으므로). 유저는 좋아하는 콘텐츠를 시청하고 있을 것이라는 가정하에 단순히 설계된 모델의 한계점이다. 


본 Loss 를 적용한 학습은 Epoch 100 회 시행했으며, 이는 모델의 하이퍼파라미터 값이다.


<br/>

---

# 🪄 Recommendation Flow

>![image](https://github.com/user-attachments/assets/b542715d-21db-41e4-95fa-53da128db43d)
그림 9. 취향 학습 후 콘텐츠 랭킹 변화

그림 9는 모델 학습을 마친 후, 특정 콘텐츠에 대한 유사도 기반 랭킹 추출 결과이다. 실제 출력값을 재구성하였으며, 좌측은 기존 내용 기반의 콘텐츠 벡터 기반의 추천 결과, 우측은 PAIR RANKING 취향 학습 후 추천 결과이다. 새롭게 올라온 콘텐츠와 삭제된 콘텐츠도 있고, 기존 추천결과에서 순서만 섞인 것도 존재하는 것을 확인할 수 있었다.


# 🧑‍💻 Code

[🗂️ Github - Recommender system for Korean variety show "Infinite Challenge"](https://github.com/uoahvu/infinite-challenge-contents-recommendation)