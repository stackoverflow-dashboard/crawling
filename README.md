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

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bedd02c3-c28d-4bc7-afcd-ddb06d97b3ca/f004f311-3839-4595-bd34-a98c5774b909/Untitled.png)

<aside>
💡 `raw_data` : 외부에서 가져온 데이터들을 테이블에 저장하는 곳
`analytics` : raw_data를 가공해서 summary 하는 등의 결과 테이블 저장하는 곳

* 강의 보시면 `raw_data`에 기본 정보들 넣고 이를 가공하여 저장한 곳이 `analytics` !!!

</aside>

## 3. 데이터 크롤링 - selenium

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bedd02c3-c28d-4bc7-afcd-ddb06d97b3ca/791d779e-de52-484f-a76d-9f988ad5014c/Untitled.png)

<aside>
💡 수집 항목 : 질문 제목, 태그, 답변 수, 조회 수, 시간
수집 일자 : 2024.2.29 ~ 2024.5.15
총 데이터 수 : 126,351

</aside>

## 4. 데이터 적재 - S3

![스크린샷 2024-05-17 오전 10.07.38.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/bedd02c3-c28d-4bc7-afcd-ddb06d97b3ca/add5f458-3336-481a-a54d-6c18be79fad4/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2024-05-17_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_10.07.38.png)

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
    
    ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bedd02c3-c28d-4bc7-afcd-ddb06d97b3ca/30a58e0d-9c0a-4805-a698-dfea223df0d7/Untitled.png)
    
- QUESTION 테이블
    
    ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bedd02c3-c28d-4bc7-afcd-ddb06d97b3ca/80e22697-1a32-4af9-803a-b5d132cf352c/Untitled.png)
    
- TAG 테이블
    
    ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bedd02c3-c28d-4bc7-afcd-ddb06d97b3ca/f52af62e-ed2a-4064-b62f-7864ce8ced4c/Untitled.png)
    

## 6. 차트 생성 및 대시보드 구성 - preset

- **차트 생성**
    - 전체 질문수 - big number
        - 총 질문 합산 수
        
        ![전체-질문수-2024-05-17T01-15-04.980Z.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/bedd02c3-c28d-4bc7-afcd-ddb06d97b3ca/d5485810-028d-4275-a4b5-fa9cd00bc26d/%E1%84%8C%E1%85%A5%E1%86%AB%E1%84%8E%E1%85%A6-%E1%84%8C%E1%85%B5%E1%86%AF%E1%84%86%E1%85%AE%E1%86%AB%E1%84%89%E1%85%AE-2024-05-17T01-15-04.980Z.jpg)
        
    - 일별 질문수 - line chart
        - 하루동안 등록된 질문 합산 수
        
        ![-2024-05-17T01-15-11.250Z.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/bedd02c3-c28d-4bc7-afcd-ddb06d97b3ca/d541866d-67d9-421b-86e2-9d7657c843ea/-2024-05-17T01-15-11.250Z.jpg)
        
    - 일별 질문/답변 비율 - bar chart
        - 일별로 질문 수, 답변 수를 한눈에 비교
        - 약 3달 동안의 질문량 추이 파악 가능
        
        ![질문-답변-비율-2024-05-17T01-17-45.914Z.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/bedd02c3-c28d-4bc7-afcd-ddb06d97b3ca/885db1ea-b524-41ff-b621-73dd1831df37/%E1%84%8C%E1%85%B5%E1%86%AF%E1%84%86%E1%85%AE%E1%86%AB-%E1%84%83%E1%85%A1%E1%86%B8%E1%84%87%E1%85%A7%E1%86%AB-%E1%84%87%E1%85%B5%E1%84%8B%E1%85%B2%E1%86%AF-2024-05-17T01-17-45.914Z.jpg)
        
    
    - 요일별 질문/답변 추이 - area chart
        - 어떤 요일에 질문/답변이 활발히 이뤄졌는지 파악 가능
        
        ![요일별-질문-답변-추이-2024-05-17T01-19-24.639Z.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/bedd02c3-c28d-4bc7-afcd-ddb06d97b3ca/f3faf77a-9a96-4a90-be81-1fde3f47f7ea/%E1%84%8B%E1%85%AD%E1%84%8B%E1%85%B5%E1%86%AF%E1%84%87%E1%85%A7%E1%86%AF-%E1%84%8C%E1%85%B5%E1%86%AF%E1%84%86%E1%85%AE%E1%86%AB-%E1%84%83%E1%85%A1%E1%86%B8%E1%84%87%E1%85%A7%E1%86%AB-%E1%84%8E%E1%85%AE%E1%84%8B%E1%85%B5-2024-05-17T01-19-24.639Z.jpg)
        
    - 시간대별 질문/답변 추이 - area chart
        - 어떤 시간대에 질문/답변이 활발히 이뤄졌는지 파악 가능
        
        ![시간대별-질문-답변-추이-2024-05-17T01-19-30.180Z.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/bedd02c3-c28d-4bc7-afcd-ddb06d97b3ca/cd26932f-a038-41b2-b7f3-34a147c6a9bf/%E1%84%89%E1%85%B5%E1%84%80%E1%85%A1%E1%86%AB%E1%84%83%E1%85%A2%E1%84%87%E1%85%A7%E1%86%AF-%E1%84%8C%E1%85%B5%E1%86%AF%E1%84%86%E1%85%AE%E1%86%AB-%E1%84%83%E1%85%A1%E1%86%B8%E1%84%87%E1%85%A7%E1%86%AB-%E1%84%8E%E1%85%AE%E1%84%8B%E1%85%B5-2024-05-17T01-19-30.180Z.jpg)
        
    - 상위 태그 30 - pie chart
        - 가장 많이 질문된 태그 30위까지 표현
        - 질문별로 사용된 모든 태그 합산
        
        ![상위-태그-30-2024-05-17T01-21-47.413Z.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/bedd02c3-c28d-4bc7-afcd-ddb06d97b3ca/093b05ba-8a22-47ec-990c-6d1c08229f2a/%E1%84%89%E1%85%A1%E1%86%BC%E1%84%8B%E1%85%B1-%E1%84%90%E1%85%A2%E1%84%80%E1%85%B3-30-2024-05-17T01-21-47.413Z.jpg)
        
    - 상위 태그 250 - wordcloud
        - 상위 250개 태그 워드 클라우드로 표현
        
        ![-2024-05-17T01-21-52.189Z.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/bedd02c3-c28d-4bc7-afcd-ddb06d97b3ca/f75cd255-a51f-4284-8105-a8f6035666a4/-2024-05-17T01-21-52.189Z.jpg)
        
    - 상위 태그 질문/조회/답변 수 비교
        - 질문/조회/답변 별 상위 태그 10개 선별
        - 각 태그별로 주별 질문/조회/답변 추이를 표현
        
        ![상위-태그-답변-수-비교-2024-05-17T01-23-59.591Z.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/bedd02c3-c28d-4bc7-afcd-ddb06d97b3ca/c0f5472c-e513-4217-9495-5d8725f6a946/%E1%84%89%E1%85%A1%E1%86%BC%E1%84%8B%E1%85%B1-%E1%84%90%E1%85%A2%E1%84%80%E1%85%B3-%E1%84%83%E1%85%A1%E1%86%B8%E1%84%87%E1%85%A7%E1%86%AB-%E1%84%89%E1%85%AE-%E1%84%87%E1%85%B5%E1%84%80%E1%85%AD-2024-05-17T01-23-59.591Z.jpg)
        
        ![상위-태그-조회-수-비교-2024-05-17T01-23-55.591Z.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/bedd02c3-c28d-4bc7-afcd-ddb06d97b3ca/525a251e-9288-46ff-aa1b-77283a37079f/%E1%84%89%E1%85%A1%E1%86%BC%E1%84%8B%E1%85%B1-%E1%84%90%E1%85%A2%E1%84%80%E1%85%B3-%E1%84%8C%E1%85%A9%E1%84%92%E1%85%AC-%E1%84%89%E1%85%AE-%E1%84%87%E1%85%B5%E1%84%80%E1%85%AD-2024-05-17T01-23-55.591Z.jpg)
        
        ![상위-태그-질문-수-비교-2024-05-17T01-23-50.939Z.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/bedd02c3-c28d-4bc7-afcd-ddb06d97b3ca/563bf870-4b64-43d4-8e80-da0e7cb86ed0/%E1%84%89%E1%85%A1%E1%86%BC%E1%84%8B%E1%85%B1-%E1%84%90%E1%85%A2%E1%84%80%E1%85%B3-%E1%84%8C%E1%85%B5%E1%86%AF%E1%84%86%E1%85%AE%E1%86%AB-%E1%84%89%E1%85%AE-%E1%84%87%E1%85%B5%E1%84%80%E1%85%AD-2024-05-17T01-23-50.939Z.jpg)
        
