# 🐶Pet_0Friendly_Facility_Recommendation_Chatbot


https://github.com/user-attachments/assets/6fca47a0-e042-4db2-b854-5f6ea19b8573


# 1. 프로젝트 소개
---
본 프로젝트는 사용자가 직접 검색하지 않아도 대화를 통해 **반려동물 동반 가능 시설의 정보를 제공**받을 수 있는 챗봇을 구현하는 것을 목표로 한다. 

이를 위해 **자연어 처리 기술**과 GPT 모델을 비롯한 **대형 언어 모델(LLM)** 의 동작 원리를 이해하고, 실제로 이를 다뤄보는 경험을 프로젝트의 핵심 과제로 삼았다.


　

 　
# 2. 배경 및 목적
---
국내 반려동물 가구가 꾸준히 증가함에 따라, 반려동물과 함께 여행하거나 외출할 수 있는 시설에 대한 수요도 높아지고 있다. 

하지만 이러한 시설 정보를 제공하는 서비스는 여전히 부족해, 많은 반려인이 직접 검색해야 하는 불편을 겪고 있다. 

이에 본 프로젝트는 대화형 챗봇을 통해 반려동물 동반 가능 시설에 대한 정보를 손쉽게 제공함으로써, 반려인들의 **정보 탐색 과정을 간소화하고 편의성을 높이는 것**을 목표로 한다.


　

 　
# 3. 활용데이터
---
[전국 반려동물 동반가능 문화시설 위치(한국문화정보원)](https://www.culture.go.kr/data/openapi/openapiView.do?id=585)


　

 　
# 4. 상세과정
---
프로젝트는 크게 **데이터 전처리** - **RAG 파이프라인 구축** - **웹 사이트 개발** 순으로 진행된다.

　
## 4-1. 데이터 전처리
- 파싱 : API 요청을 보내고 ElementTree를 사용하여 XML응답을 파싱한다. 각 item 요소에서 **필요한 정보를 추출**한다.
- ‘discription’에 **‘동반불가’가 포함된 데이터**는 동반 가능 시설이 아니므로 **제거**한다. 총 (687개)
- 모든 행이 NaN인 **‘category3’ 컬럼을 제거**한다
- llm이 해당 데이터를 기반으로 답해야 하므로, 결측치에 대해서는 ‘정보없음’으로 답하도록 하기 위해 **나머지 결측치를 ‘정보없음’으로 대체**한다
- 리스트를 df로 변환한다.

　
## 4-2. RAG 파이프라인 구축
Rag : 사용자의 질문을 파악하여 관련 데이터를 데이터베이스에서 검색하여 답변하도록. 할루시네이션을 해소하기 위함
#### 1) Rag data 생성
사용자의 질문과 rag data의 page_content 속성과의 유사도를 통한 검색이 진행된다. 즉, 검색에 필요한 정보가 모두 담긴 ‘하나’의 컬럼이 필요하므로, 생성여 page_content로 지정한다. **각 행의 정보를 요약한 텍스트인 ‘rag_text’ 컬럼을 추가**한다.
#### 2). Load Data
**‘rag_text’를  page_content로 지정**하고 DataFrameLoader를 사용하여 전체 데이터프레임을 Document 객체의 리스트로 **로드**한다. 
#### 3). Text Split
검색 효율성을 높이기 위해 **RecursiveCharacterTextSplitter**를 이용하여 로드한 **텍스트를 작은 조각으로 분할**한다. 이 과정은 **문서 검색 및 처리 속도를 향상**시키는 데 요한 역할을 한다.
#### 4). Indexing
**HuggingFaceEmbeddings**를 사용하여 분할한 텍스트를 **임베딩 한 후**, **Chroma Vector DB에 저장**한다. 
#### 5). Retrieval(검색)
**MultiQueryRetriever** 사용. MultiQueryRetriever는 사용자가 입력한 쿼리를 LLM을 활용해 여러 변형된 쿼리로 생성한 뒤, 이를 바탕으로 문서를 검색하고 중복된 항목을 제거하여 고유한 문서들을 결합해 결과로 반환하는 도구이다.

검색기는 이전에 저장한 **vector_db**를 사용하며, 검색 알고리즘으로는 **MMR**(Maximal Marginal Relevance)을 적용하고, LLM으로는 **GPT-4o-mini**를 사용한다. 또한, search_kwargs 매개변수에서 'k': 3으로 설정해 **상위 3개**의 정보를 반환하도록 한다.
#### 6). 생성
LangChain을 사용하여 구현한다. **ConversationalRetrievalChain**과 **ConversationBufferMemory**를 사용하고, Chat History와 retriever를 통해 질의응답 기억을 참고하여 답변하도록 한다.

LLM 모델로는 **GPT-4o-mini**를 사용하고, **temperature**값을 0으로 지정한다. 창의적인 답변보단 사실을 기반으로 정해진 응답을 해야 하므로 낮은 temperature 값을 사용한다.

<u>[Prompt]</u>
```ruby
prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            """
            You are a helpful assistant. 
            You are a chatbot that provides information about pet-friendly facilities. 
            Please respond based on the context.
            Answer questions using only the following context.
            모든 정보를 알려줘. 간결하게 답해줘.
            If you don't know the answer just say you don't know, 
            don't make it up:
            \n\n
            {context}
            """,
        ),
        ("human", "{question}"),
    ]
)
```
## 4-3. 웹 서비스 개발
앞서 제작한 챗봇을 실제로 사용해 보고자 **streamlit**을 사용하여 채팅 서비스를 구현하였다.
사용자가 **OpenAI API 키를 사이드바에 입력**하여 챗봇을 사용할 수 있도록 하였으며, session_state에 Chroma 데이터와 대화 기록(chat_histo ry)을 저장하여 더욱 효율적이고 원활한 서비스 구동이 가능하도록 최적화하였다.

또한 답변과 함께 **지도를 출력**해 시설의 위치 정보를 직관적으로 제공하였다. 
rag 검색 과정 중 반환된 답변에 활용할 문서의 **metadata에서 좌표 데이터를 추출**한다. 이 좌표 데이터는 해당 시설의 위치 정보를 담고 있으며, **folium**을 사용하여 해당 위치를 지도 위에 시각화하여 나타낸다.
