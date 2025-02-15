---
title: AWS S3를 활용한 비정형 데이터 저장 및 관리 (서버 부담 최소화)
date: 2025-02-13
categories:
  - MyCodingTest
tags:
  - MCT
  - S3
---
## 구현 배경
- 코드 및 메모와 같은 비정형 데이터의 크기를 예측하기 어렵고, 애플리케이션 단과 데이터베이스 단에서 대용량의 비정형 데이터를 메모리에 로딩시 메모리 부족 현상에 대한우려
- AWS S3를 활용하여 비정형 데이터를 효율적으로 저장하고 관리하도록 시스템을 설계
- 클라이언트가 S3와 직접 통신하므로, 서버는 데이터 전송에 대한 부담 저하
- 제한된 시간 동안만 유효한 signed URL을 생성하여 안전

## 구현 내용

- `spring-cloud-aws-starter` 의 `S3Template` 으로 손쉽게 구현
- signed URL 생성 과정
1. 클라이언트(Chrome 확장 프로그램)는 서버로부터 signed URL을 요청
2. 서버는 클라이언트에게 S3에 직접 데이터를 업로드/다운로드할 수 있는 권한을 가진 signed URL을 생성하여 제공
3. 클라이언트는 signed URL을 사용하여 S3에 직접 데이터를 업로드/다운로드

다음 그림은 signed URL 생성하는 과정에대한 시퀀스 다이어그램
![](../../../images/Pasted%20image%2020250213195300.png)


```java S3Service.java
package com.mycodingtest.service;  
  
import io.awspring.cloud.s3.S3Template;  
import org.springframework.beans.factory.annotation.Value;  
import org.springframework.stereotype.Service;  
  
import java.time.Duration;  
import java.util.List;  
  
@Service  
public class S3Service {  
  
    private final S3Template s3Template;  
    @Value("${aws.s3.bucketName}")  
    private String bucketName;  
  
    public S3Service(S3Template s3Template) {  
        this.s3Template = s3Template;  
    }  
  
    public String getCodeReadUrl(String submissionId, Long userId) {  
        String key = userId + "/codes/" + submissionId + ".txt";  
        return s3Template.createSignedGetURL(bucketName, key, Duration.ofMinutes(2)).toString();  
    }  
  
    public String getCodeUpdateUrl(String submissionId, Long userId) {  
        String key = userId + "/codes/" + submissionId + ".txt";  
        return s3Template.createSignedPutURL(bucketName, key, Duration.ofMinutes(2)).toString();  
    }  
  
    public String getMemoReadUrl(String reviewId, Long userId) {  
        String key = userId + "/memos/" + reviewId + ".txt";  
        if (s3Template.objectExists(bucketName, key)) {  
            return s3Template.createSignedGetURL(bucketName, key, Duration.ofMinutes(2)).toString();  
        }  
        return "noMemo";  
    }  
  
    public String getMemoUpdateUrl(String reviewId, Long userId) {  
        String key = userId + "/memos/" + reviewId + ".txt";  
        return s3Template.createSignedPutURL(bucketName, key, Duration.ofMinutes(2)).toString();  
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
