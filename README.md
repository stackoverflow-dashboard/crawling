# Stack Overflow 질문 분석을 위한 대시보드

# 목차
1. 목적
2. 기대효과
3. 프로젝트 진행
    1. 주제선정
    2. ERD 작성
    3. 데이터 크롤링 - selenium
    4. 데이터 적재 - S3
    5. 테이블 생성 - Snowflake
    6. 차트 생성 및 대시보드 구성 - preset

# 목적

stackoverflow 질문 데이터를 활용하여 다양한 기술 트렌드 및 활동 상황을 분석하고 시각화하여 제공합니다.

- 다양한 기술 트렌드 및 인기도에 대한 인사이트 제공
- stackoverflow 질문수와 답변수로 개발자 간의 교류가 활발한지 파악

# 기대효과

- 현재 기술 트렌드와 지속적인 인기도 파악
- 일별 질문수와 답변수로 stackoverflow의 활성도 확인
- 요일별 질문 대비 답변 비율을 확인하여 답변 받을 확률 확인
- 특정 기술들에 대한 질문, 답변, 조회 수 추이를 확인하여 기술별로 참여자의 흥미도 확인 가능
- 자주 사용되는 태그들을 확인하여 해당 기술에 대한 지속적인 인기도 파악

# 프로젝트 진행 순서

1. 주제 선정
2. ERD 작성
3. 데이터 크롤링 - selenium
4. 데이터 적재 - S3
5. 테이블 생성 - snowflake
6. 차트 생성 및 대시보드 구성 - preset

## 1. 주제 선정

### **Stackoverflow 질문 데이터 분석을 위한 대시보드**

## 2. ERD 작성

![image](https://github.com/stackoverflow-dashboard/crawling/assets/66053902/ac60e504-6bfa-4502-8dec-804b88653f3b)


<aside>
💡 `raw_data` : 외부에서 가져온 데이터들을 테이블에 저장하는 곳
`analytics` : raw_data를 가공해서 summary 하는 등의 결과 테이블 저장하는 곳

* 강의 보시면 `raw_data`에 기본 정보들 넣고 이를 가공하여 저장한 곳이 `analytics` !!!

</aside>

## 3. 데이터 크롤링 - selenium
<img width="747" alt="image" src="https://github.com/stackoverflow-dashboard/crawling/assets/80039556/8c24527c-6f74-4f7c-a381-d28b4fff99ea">


💡 수집 항목 : `질문 제목`, `태그`, `답변 수`, `조회 수`, `일시(날짜)`

수집 일자 : 2024.2.29 ~ 2024.5.15
총 데이터 수 : 126,351

## 4. 데이터 적재 - S3

![image](https://github.com/stackoverflow-dashboard/crawling/assets/66053902/65d48e3c-02a6-471e-88b8-a49b610b5313)


## 5. 테이블 생성 - Snowflake

- SQL 구문
    
    ```sql
    -- 크롤링한 데이터를 적재할 SOF_INFO 테이블 생성
    CREATE OR REPLACE TABLE dev.raw_data.sof_info (
     question_id int primary key,
     title varchar(200),
     date timestamp,
     tags array,
     answer_cnt int,
     view_cnt int
    );
    
    -- 크롤링한 데이터 파일인 CSV를 S3에서 가져와서 SOF_INFO 테이블에 적재
    COPY INTO dev.raw_data.sof_info
    FROM 's3://{s3 주소}/sof_info.csv'
    credentials=(AWS_KEY_ID='{AWS KEY 넣기}' AWS_SECRET_KEY='{AWS SECRET KEY 넣기}')
    FILE_FORMAT = (type='CSV' skip_header=1 FIELD_OPTIONALLY_ENCLOSED_BY='"')
    ON_ERROR = CONTINUE;
    
    -- QUESTION 테이블 생성 (CTAS 방식) 
    CREATE OR REPLACE TABLE dev.raw_data.question AS
    SELECT question_id, title, date, answer_cnt, view_cnt
    FROM dev.raw_data.sof_info;
    
    -- TAG 테이블 생성 (CTAS 방식)
    CREATE OR REPLACE TABLE dev.raw_data.tag AS
    SELECT
    	ROW_NUMBER() OVER (ORDER BY date) AS tag_id,
    	question_id,
    	t.value::string AS name
    FROM
    	dev.raw_data.sof_info,
    	LATERAL FLATTEN(input => tags) t;
    
    -- 특정 날짜 삭제 - 4월18일과 5월14일 삭제
    --   1. TAG 테이블부터 데이터 삭제
    DELETE FROM raw_data.tag
    WHERE question_id IN (
      SELECT q.question_id
      FROM raw_data.question q
      WHERE TO_DATE(q.date) IN ('2024-04-18', '2024-05-14')
    );
    
    --   2. QUESTION 테이블의 데이터 삭제
    DELETE FROM raw_data.question
    WHERE TO_DATE(date) IN ('2024-04-18', '2024-05-14');
    
    -- 4/18과 5/14일 삭제됐는지 확인 조회
    select q.question_id, t.name, TO_DATE(date)
    from raw_data.question q
    join raw_data.tag t on q.question_id = t.question_id
    WHERE TO_DATE(date) IN ('2024-04-18', '2024-05-14');
    ```
    
    <aside>
    💡 대량 데이터를 적재할 때는 `INSERT INTO ..` 보다는 CTAS 방식이 더 효율적이며 성능이 더 좋으므로 처음에는 CTAS로 생성하기
    
    </aside>
    

- SOF_INFO 테이블
    
![image](https://github.com/stackoverflow-dashboard/crawling/assets/66053902/c1753883-c2bc-47ff-a597-21d72fe82751)

    
- QUESTION 테이블
    
![image](https://github.com/stackoverflow-dashboard/crawling/assets/66053902/b4668750-1dc5-4643-a6c0-bb5f7f31ca7e)

    
- TAG 테이블
    
![image](https://github.com/stackoverflow-dashboard/crawling/assets/66053902/530f8f10-c538-4333-ba06-ab63ae1324eb)

    

## 6. 차트 생성 및 대시보드 구성 - preset

- **차트 생성**
    - 전체 질문수 - big number
        - 총 질문 합산 수
        
![image](https://github.com/stackoverflow-dashboard/crawling/assets/66053902/b331ead4-da38-4706-a8f6-724dee5ee076)

        
    - 일별 질문수 - line chart
        - 하루동안 등록된 질문 합산 수
        
![image](https://github.com/stackoverflow-dashboard/crawling/assets/66053902/29c80a40-03a8-4b76-84ef-de5bbdc07f0e)

        
    - 일별 질문/답변 비율 - bar chart
        - 일별로 질문 수, 답변 수를 한눈에 비교
        - 약 3달 동안의 질문량 추이 파악 가능
        
![image](https://github.com/stackoverflow-dashboard/crawling/assets/66053902/7cc840be-94ee-4e21-b0d3-898bd308fd3c)

        
    
    - 요일별 질문/답변 추이 - area chart
        - 어떤 요일에 질문/답변이 활발히 이뤄졌는지 파악 가능
        
![image](https://github.com/stackoverflow-dashboard/crawling/assets/66053902/2a9aea2f-00ed-41f6-9f78-0d933827a8e8)

        
    - 시간대별 질문/답변 추이 - area chart
        - 어떤 시간대에 질문/답변이 활발히 이뤄졌는지 파악 가능
        
![image](https://github.com/stackoverflow-dashboard/crawling/assets/66053902/be56b392-64b3-4d3e-8337-f63ec052c3af)

        
    - 상위 태그 30 - pie chart
        - 가장 많이 질문된 태그 30위까지 표현
        - 질문별로 사용된 모든 태그 합산
        
![image](https://github.com/stackoverflow-dashboard/crawling/assets/66053902/7d5e449f-6e69-49fa-a25d-611b0136cc24)

        
    - 상위 태그 250 - wordcloud
        - 상위 250개 태그 워드 클라우드로 표현
        
![image](https://github.com/stackoverflow-dashboard/crawling/assets/66053902/4bc95572-d5b6-4bd0-b0ff-c04d60c83f28)

        
    - 상위 태그 질문/조회/답변 수 비교
        - 질문/조회/답변 별 상위 태그 10개 선별
        - 각 태그별로 주별 질문/조회/답변 추이를 표현
        
![image](https://github.com/stackoverflow-dashboard/crawling/assets/66053902/4ad0a8d6-970d-4dd8-a18e-e0ea635d8ad1)

        
![image](https://github.com/stackoverflow-dashboard/crawling/assets/66053902/d115b32a-402f-4b81-996d-ff6276c73503)

        
![image](https://github.com/stackoverflow-dashboard/crawling/assets/66053902/0dd9321d-b186-47f7-9fb8-838e2262d85b)

        
