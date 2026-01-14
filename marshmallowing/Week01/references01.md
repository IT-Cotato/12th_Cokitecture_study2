# **SpringBoot Batch를 이용한 주소DB 구축**

> 배달 어플에서는 픽업지와 배달지의 위치가 매우 중요하다. 따라서 해당 위치의 변경 및 확인을 위한 주소 검색서버가 필요했고, 직접 DB에 데이터를 쌓고 서버를 배포하는 방식을 선택했다.
>

---

## **주소DB를 따로 구축한 이유**

> [www.juso.go.kr](http://www.juso.go.kr/) 은 정부가 관리하는 곳으로, 주소검색 솔루션과 주소검색 API 그리고 txt형식의 주소 데이터를 제공해준다.
>
- **최신 주소 데이터 자동동기화의 불필요**
- **테스트용 주소에 대한 처리**
- **SLA(Service Level Agreement) 미보장**
    - 트래픽 대응, 장애대응 등 직접적으로 운영, 관리할 수 있는 주소 서버가 필요

## **Spring Batch란?**

> Spring Batch는 로깅/추적, 트랜잭션 관리, 작업 처리 통계, 작업 재시작, 건너뛰기, 리소스 관리 등 대용량 레코드 처리에 필수적인 기능을 제공한다. 또한 최적화 및 파티셔닝 기술을 통해 대용량 및 고성능 배치 작업을 가능하게 하는 고급 기술 서비스 및 기능을 제공한다.
>

```html
[ 실행 프레임워크 ]
   ├─ Job / Step (처리 흐름 정의)
   ├─ Reader / Processor / Writer (데이터 처리)
   ├─ Transaction / Chunk 관리
   ├─ JobRepository (상태 저장: DB)
```

![Spring Batch에서의 Job은 여러가지 Step의 모음으로 구성되어 있으며 Job은 순차적인 Step을 수행하며 Batch를 수행](attachment:01cfeba5-a688-4ed9-b104-1457b4d59c78:image.png)

Spring Batch에서의 Job은 여러가지 Step의 모음으로 구성되어 있으며 Job은 순차적인 Step을 수행하며 Batch를 수행

- JobRepository: 현재 실행 중인 프로세스의 meta data를 저장
- JobLauncher: Client로부터 요청을 받아 Job을 실행하는 객체
- Job: Step들의 컨테이너, Step들의 단계 혹은 재시작 가능성과 같은 모든 단계에 대한 전역 속성을 구성

![image.png](attachment:c2149b1e-6f73-420e-9e96-798939c0924f:image.png)

- Step: 실제 batch처리를 정의, 제어하는 정보가 들어있는 도메인 객체
- ItemReader: 한 Step안에서 FlatFile, XML, DB 등 여러 input에서 Item을 읽어 들임
- ItemProcessor: ItemReader로부터 읽어들인 Item을 DB에 Write하기 전에 필요한 로직을 처리
- ItemWriter: ItemReader로부터 읽어 들인 Item을 Insert, Update 처리

## **Spring Batch를 사용한 이유**

> 대량 데이터를 읽고 → 변환하고 → 저장하는 작업을 **대규모로, 실패해도 이어서 운영 가능하게** 만드는 프레임워크
>
1. **실패 시점부터 재시작 가능**

   주소 txt파일을 읽어 주소데이터를 넣을 때 예상치 못한 에러로 Batch Job이 중단 될 수도 있다.

   Spring Batch를 통해 Application을 실행하면 BATCH_JOB_EXECUTION_CONTEXT, BATCH_STEP_EXECUTION_CONTEXT 라는 테이블이 생성되고, 이 테이블에는 특정 Job or Step의 실행을 지속할 수 있게하는 데이터가 업데이트된다.

   이를 통해 중간에 batch job이 중단되더라도 문제파악 후 실패시점부터 다시 실행할 수 있다.

2. **기본적인 ItemReader, ItemWriter interface 및 구현체 제공**

   Spring Batch에서는 여러 형태의 데이터들(DB, 플랫파일)을 하나의 인터페이스(ItemReader, ItemWriter)로 Step을 구현할 수 있게 제공해준다.


## **주소 Batch Job 적용**

1. 각 데이터 별(주소, 지번, 부가, 도로명) 주소데이터 txt파일을 읽는 ItemReader, 주소DB에서 시도, 시군구 등 데이터를 읽는 ItemReader 구현합니다.
2. ItemReader로 읽은 Item(도메인객체)들을 새로운 주소DB에 Insert할 ItemWriter를 구현합니다.
3. ItemReader와 ItemWriter, 필요에 따라서는 ItemProcessor를 추가하여 **데이터별 Step을 구현**합니다.
4. 각 Step들을 플로우에 맞게 실행할 Job을 생성하고
5. 해당 Job name을 parameter로 JobLauncher에게 실행요청을 합니다.

### **주소 txt파일을 라인단위로 읽는 ItemReader**

```java
@Bean
   public FlatFileItemReader<Juso> jusoItemReader() {
       FlatFileItemReader<Juso> reader = new FlatFileItemReader<>();
       reader.setEncoding(CP949); // 주소 txt파일 한글인코딩
       reader.setLineMapper(new DefaultLineMapper<Juso>() {{
           setLineTokenizer(new DelimitedLineTokenizer("|") {{
               setNames(new String[]{
                       "id",
                       "jusoName",
                       "jusoCol1",
                       "jusoCol2"
               });
           }});
           setFieldSetMapper(new BeanWrapperFieldSetMapper<Juso>() {{
               setTargetType(Juso.class);
           }});
       }});
       return reader;
   }
```

- FlatFileItemReader를 이용하여 FlatFile(.txt)을 읽는 ItemReader Bean을 생성
- 플랫파일을 라인단위로 읽은 후, read한 각 라인을 도메인객체로 리턴받게 구현

### **read한 Item(도메인객체)을 DB에 저장할 ItemWriter**

```java
@Bean
    public JdbcBatchItemWriter<Juso> jusoItemWriter() {
        JdbcBatchItemWriter<Juso> writer = new JdbcBatchItemWriter<>();
        writer.setAssertUpdates(false);
        writer.setItemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<>());
        writer.setJdbcTemplate(new NamedParameterJdbcTemplate(dataSource));
        writer.setSql("INSERT INTO `juso_info`n" +
                "(`id`,n" +
                "`juso_name`,n" +
                "`juso_col1`,n" +
                "`juso_col2`)n" +
                "VALUESn" +
                "(:id,n" +
                ":jusoName,n" +
                ":jusoCol1,n" +
                ":jusoCol2)"); // Item을 read할 때 mapping해준 named paramter로 set value
        return writer;
    }
```

- JdbcBatchItemWriter를 이용하여 Jdbc모듈 형식으로 mysql에 접근
- NamedParameterJdbcTemplate을 이용하여 read한 Item (도메인객체)을 namedParameter로 가져와 insert할 value를 셋팅

### **ItemReader와 ItemWriter를 이용하여 여러 데이터의 Step 구현**

```java
@Bean
public Step jusoStep() {
    return stepBuilderFactory.get("jusoStep")
            .<Juso, Juso>chunk(CHUNK_SIZE)
            .reader(new MultiResourceItemReader<Juso>() {{
                setResources(resources);
                setDelegate(jusoItemReader());
            }})
            .writer(jusoItemWriter())
            .build();
}
```

- 각 데이터에 맞는 도메인 객체와, txt파일을 Resource객체로 변환 후 set
- step에는 chunk size를 지정해 chunk size 만큼 ItemReader와 ItemWriter가 동작하여, 원하는 단위로 트랜잭션 커밋을 할 수 있다

### **주소DB에서 시도, 시군구, 행정동을 뽑아 저장하는 Step 구현**

```java
public <T> Step createStep(String stepName, int chunkSize, String readSql, RowMapper<T> readRowMapper) {
    return stepBuilderFactory.get(stepName)
            .<T, T>chunk(chunkSize)
            .reader(new JdbcCursorItemReader<T>() {{
                setDataSource(dataSource);
                setSql(readSql);
                setRowMapper(readRowMapper);
            }})
            .writer(new JpaItemWriter<T>() {{
                setEntityManagerFactory(entityManagerFactory);
            }}).build();
}
```

### **각 Step들의 순서를 지정할 Job 생성**

구현한 Step들을 Job단위로 묶어 원하는 순서대로 실행할 Job을 만든다

```java
@Bean
public Job initDataJob() throws IOException {
    return jobBuilderFactory.get("initDataJob")
            .incrementer(new RunIdIncrementer())
            .start(batchConfiguration.jusoStep0())
            .next(batchConfiguration.jusoStep1())
            .next(batchConfiguration.jusoStep2())
            .next(batchConfiguration.initSidoStep())
            .next(batchConfiguration.initSigunguStep())
            .next(batchConfiguration.initDongOfAdminStep())
            .build();
}
```

jusoStep을 먼저 실행하여 주소txt 파일로부터 DB에 데이터를 쌓고, 그 후에 시도, 시군구 등의 데이터를 다시 주소DB로 부터 추출한다.

## 근접 서비스 관점에서 본 Spring Batch의 역할

근접 서비스(LBS)는 GeoHash 계산과 Redis 기반 조회를 중심으로 사용자의 요청에 즉시 응답한다.

이러한 조회가 빠르고 안정적이게 동작하기 위해서는, Redis나 DB에 저장된**`지오해시 → 사업장 ID 목록`과 같은 인덱스 데이터가 미리 정확하게 준비되어 있어야 한다**는 전제가 필요하다.

이 인덱스와 기초 데이터(주소, 좌표, 행정구역 정보 등)를 생성·정제·갱신하는 작업은 대량의 데이터를 다루고, 실패 시 복구가 가능해야 하므로 오프라인 배치 처리가 쓰인다.

**즉, 근접 서비스 아키텍처에서 Spring Batch는 빠른 조회를 가능하게 만드는 데이터와 인덱스를 미리 준비하는 역할을, LBS 서빙 로직은 준비된 데이터를 바탕으로 실시간 근접 검색을 수행하는 역할을 맡는다.**

이러한 역할 분담을 통해, 대규모 위치 기반 서비스는 데이터 갱신 안정성과 실시간 응답 성능을 동시에 확보할 수 있다.

[주소검색서버(woowahan-juso) 개발기(上) | 우아한형제들 기술블로그](https://techblog.woowahan.com/2575/?utm_source=chatgpt.com)

[Spring Batch란? 이해하고 사용하기(예제소스 포함)](https://khj93.tistory.com/entry/Spring-Batch%EB%9E%80-%EC%9D%B4%ED%95%B4%ED%95%98%EA%B3%A0-%EC%82%AC%EC%9A%A9%ED%95%98%EA%B8%B0#google_vignette)