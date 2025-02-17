---
title: AWS S3를 활용한 비정형 데이터 관리 (서버 부담 최소화)
date: 2025-02-13
categories: 트러블슈팅
tags:
  - S3
  - Optimization
  - Troubleshooting
  - AWS
---
## 문제 정의
기존 설계는 아래와  같은 설계를 염두해 두고 있었음
- MySQL에 SourceCode, Memo 같은 텍스트 비정형 데이터를 저장시키기
- 또는 DynamoDB활용 해보기

그런데 또 AWS에서 빌릴 서버의 사양이 1G Memory인걸 생각해보니 아래와 같은 우려가 생김. 

저 1G Memory 인스턴스에 OS도 실행하고, NGINX, Certbot 등 실행되는데 저런 예측할 수 없는 데이터가 서버에 로드되는것이 타당한 것인가?

- MySQL에 Code, Memo 등 비정형 문자열 데이터를 넣어도 될까?
- 그러면 DynamoDB에 비정형 데이터를 관리하도록 해도 될까?
- 코드 및 메모와 같은 비정형 데이터의 **크기를 예측하기 어렵고**, 애플리케이션 단과 데이터베이스 단에서 대용량의 비정형 데이터를 **메모리에 로딩시 메모리 부족 현상**에 대한 **우려**
## 문제 분석
- AWS S3를 활용하여 비정형 데이터를 효율적으로 저장하고 관리하도록 시스템을 설계
- MySQL에 Code, Memo 등 비정형 문자열 데이터 관리의 경우
	- 장점
		- 익숙하고 별로 따로 설정할게 없어 편하다
		- 기존 MySQL을 그대로 쓰면 된다
	- 단점
		- 사용 예정인 1G Memory 서버에서는 **메모리 부족 가능성** 존재(이게 큼)
		- 가장 저렴한 RDB 서버를 빌릴건데 용량이 15GB밖에 안된다. 이것도 한달에 15달러인데 비싼감이 있다.
- DynamoDB에 비정형 데이터를 관리
	- 장점
		- 텍스트는 외부 저장소 에서 관리 -> MySQL RDB Server 용량 우려 저하
		- Key-Value Store라서 빠름
	- 단점
		- FreeTier로 15GB정도 무료로 줘서 처음에는 저렴해서 좋다고 생각했지만, 과금 구조를 더 들여다 보니 쓰기/읽기 비용을 책정할때 용량기준으로하는것을 확인. 도저히 예상이 불가함
		- 학생신분이라 예측 불가한 과금 구조는 두려움
		- 단순 저장소로 쓰는데 Key-Value NoSQL DB는 필요 없다고 판단. 비용 낭비
		- 유저에게 Memo, Source Code정도만 보여주기위한것. 바로 로딩되는 것까지는 불필요 

## 문제 해결 과정
- AWS Simple Storage Service(S3) 이용하기로 함
	- 저장 비용 저렴
	- 조회, 수정, 삭제 등 API가 요청수대로 과금. 매우 저렴
	- 과금 모델이 단순하여 예측이 가능한 과금 구조
- 서버에 메모리 절약에 매우 도움되는 설계 떠올림
	- AWS에서 지원하는 API중 signed URL 활용하면 우리 서버에서는 클라이언트에 signed URL만 만들어주면 클라이언트와 S3가 자기들끼리 업로드/업데이트가 가능
	- !!! 서버의 메모리자원 SAVE 가능 !!!
- 제한된 시간 동안만 유효한 signed URL을 생성

`spring-cloud-aws-starter` 의 `S3Template` 으로 손쉽게 구현
- signed URL 생성 과정

1. 클라이언트(Chrome 확장 프로그램)는 서버로부터 signed URL을 요청
2. 서버는 클라이언트에게 S3에 직접 데이터를 업로드/다운로드할 수 있는 권한을 가진 signed URL을 생성하여 제공
3. 클라이언트는 signed URL을 사용하여 S3에 직접 데이터를 업로드/다운로드

다음 그림은 signed URL 생성하는 과정에대한 시퀀스 다이어그램
![](Pasted%20image%2020250213195300.png#center)


```java S3Service.java
@Service  
public class S3Service {  
  
    private final S3Template s3Template;  
    @Value("${aws.s3.bucketName}")  
    private String bucketName;  
    @Value("${aws.s3.urlExpirationSeconds}")  
    private int urlExpirationSeconds;  
  
    public S3Service(S3Template s3Template) {  
        this.s3Template = s3Template;  
    }  
  
    public String getCodeReadUrl(String submissionId, Long userId) {  
        String key = userId + "/codes/" + submissionId + ".txt";  
        return s3Template.createSignedGetURL(bucketName, key, Duration.ofSeconds(urlExpirationSeconds)).toString();  
    }  
  
    public String getCodeUpdateUrl(String submissionId, Long userId) {  
        String key = userId + "/codes/" + submissionId + ".txt";  
        return s3Template.createSignedPutURL(bucketName, key, Duration.ofSeconds(urlExpirationSeconds)).toString();  
    }  
  
    public String getMemoReadUrl(String reviewId, Long userId) {  
        String key = userId + "/memos/" + reviewId + ".txt";  
        if (s3Template.objectExists(bucketName, key)) {  
            return s3Template.createSignedGetURL(bucketName, key, Duration.ofSeconds(urlExpirationSeconds)).toString();  
        }  
        return "noMemo";  
    }  
  
    public String getMemoUpdateUrl(String reviewId, Long userId) {  
        String key = userId + "/memos/" + reviewId + ".txt";  
        return s3Template.createSignedPutURL(bucketName, key, Duration.ofSeconds(urlExpirationSeconds)).toString();  
    }  
  
  
    public void deleteCodes(List<String> submissionIdList, Long userId) {  
        for (String submissionId : submissionIdList) {  
            String key = userId + "/codes/" + submissionId + ".txt";  
            s3Template.deleteObject(bucketName, key);  
        }  
    }  
  
    public void deleteMemo(String reviewId, Long userId) {  
        String key = userId + "/memos/" + reviewId + ".txt";  
        s3Template.deleteObject(bucketName, key);  
    }  
}
```
## 개선 결과
- `bucketName`, `urlExpirationSeconds`같은 민감한 변수는 환경변수화
- key의 root path를 userId로 두어 유저별 객체 관리가 용이하도록 구현
- 기존 설계 대비하여 스프링 서버에는 긴 text 데이터를 다루지 않아도되도록 함으로써 메모리 부담 저하