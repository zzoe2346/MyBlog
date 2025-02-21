---
title: Spring Rest Docs 도입기
tags:
  - Spring
  - Troubleshooting
  - RestDocs
date: 2024-08-19
categories: 트러블슈팅
---

## 선택 이유

`Swagger`와 `Spring Rest Docs` 중 무엇을 사용할지 고민하던 중, `Spring Rest Docs`가 운영 코드에 문서를 위한 코드를 추가안해도 된다는 것이 매력적이라서 선택하게 되었습니다.
또한, `mockMvc`를 사용해야하고 `build`를 해야하기 때문에 **테스트 기반으로 신뢰할수 있는 문서가 작성**된다는것도 좋았습니다.

## Spring Rest Docs 사용 방법

### 1. build.gradle 수정하기

```gradle
plugins { 
	id "org.asciidoctor.jvm.convert" version "3.3.2" 
}

configurations {  
    asciidoctorExt  
}

dependencies { 
	asciidoctorExt 'org.springframework.restdocs:spring-restdocs-asciidoctor: 3.0.1' 
	testImplementation 'org.springframework.restdocs:spring-restdocs-mockmvc: 3.0.1' 
}

ext {  
    snippetsDir = file('build/generated-snippets')  
}

test {  
    outputs.dir snippetsDir  
}

asciidoctor {  
    inputs.dir snippetsDir  
    configurations 'asciidoctorExt'  
    dependsOn test  
}
여기까지는 공식문서대로 설정
--------------------------------------------------------
여기는 내가 추가한 설정

//asciidoctor실행시 해당 경로 패키지 내부 싹 제거
asciidoctor.doFirst() {  
    delete file('src/main/resources/static/docs')  
}  

//asciidoctor가 끝나고 from 경로의 파일을 into 경로의 패키지로 복사
task copyDocument(type: Copy) {  
    dependsOn(asciidoctor)  
    from file("build/docs/asciidoc")  
    into file("src/main/resources/static/docs")  
}  

//build 가 copyDocument작업에 의존. 즉, copyDocument가 끝나고 build가 진행됨
build {  
    dependsOn copyDocument  
}

```

### 2. Controller 단위 테스트 수행

> [!NOTE]
> `RestDocumentationRequestBuilders`를 사용해야 `path parameter`까지 문서화가 가능합니다. 만약 `MockMvcRequestBuilders` 를 사용한다면 테스트 통과가 실패
> 할 수 있습니다.

이 부분은 공식문서를 참고 https://docs.spring.io/spring-restdocs/docs/current/reference/htmlsingle/#documenting-your-api

### 예시

```java

@WebMvcTest(controllers = WishController.class)
@AutoConfigureRestDocs
@DisplayName("위시 컨트롤러 단위테스트")
class WishControllerTest {

    private static final String URL = "/api/wishes";
    @MockBean
    private TokenService tokenService;
    @MockBean
    private JpaMetamodelMappingContext jpaMetamodelMappingContext;
    @MockBean
    private AuthInterceptor authInterceptor;
    @Autowired
    private MockMvc mockMvc;
    @Autowired
    private ObjectMapper objectMapper;
    @MockBean
    private WishService wishService;

    @Test
    @DisplayName("위시리스트 상품 추가")
    void addProductToWishList() throws Exception {
        //Given  
        when(authInterceptor.preHandle(any(), any(), any())).thenReturn(true);
        WishRequest request = new WishRequest(1L, null);

        //When  
        mockMvc.perform(RestDocumentationRequestBuilders.post(URL)
                        .header(HttpHeaders.AUTHORIZATION, "Bearer validTokenValue")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                //Then  
                .andExpect(
                        status().isOk()
                )
                .andDo(document("wish-add",
                                Preprocessors.preprocessRequest(Preprocessors.prettyPrint()),
                                Preprocessors.preprocessResponse(Preprocessors.prettyPrint()),

                                requestHeaders(
                                        headerWithName("Authorization").description("Authorization: Bearer ${ACCESS_TOKEN} +\n인증방식, 액세스 토큰으로 인증요청")
                                ),
                                requestFields(
                                        fieldWithPath("productId").type(JsonFieldType.NUMBER).description("위시에 추가할 상품 ID"),
                                        fieldWithPath("quantity").type(JsonFieldType.NULL).description("이거 무시하시면 됩니다").optional()
                                )
                        )
                );
    }
```

### 3. 본격적인 API 문서 작성

### 3.1 snippets 생성

테스트에 통과하면 자동으로 `/Users/seonghunjeong/spring-gift-point/build/generated-snippets` 이 경로에 `snippets` 가 생성됩니다.
![](스크린샷%202024-08-14%2017.05.00.png)

### 3.2 snippets으로 \*.adoc 문서 작성

adoc이 Asciidoctor의 줄임말.
https://docs.spring.io/spring-restdocs/docs/current/reference/htmlsingle/#working-with-asciidoctor 이 문서를 참고하면 된다.

### 예시

아래와 같이 adoc문서를 작성하면

```adoc
== 위시 조회  
=== 기본정보  
|===  
|Method | URL |Authentication method  
  
|GET  
|/api/wishes  
|AccessToken  
|===  
**설명**: 사용자의 위시리스트를 조회합니다. 각 위시 항목의 상품 ID, 이름, 가격, 이미지 URL, 수량 등을 반환합니다.  
  
=== 요청  
operation::wish-get[snippets='request-headers,request-body']  
=== 응답  
operation::wish-get[snippets='response-fields,response-body']  
=== 예제  
operation::wish-get[snippets='http-request,http-response']
```

아래 스크린샷 같이 웹문서를 자동으로 만들어줍니다.
![](스크린샷%202024-08-14%2017.14.30.png)
![](스크린샷%202024-08-14%2017.15.31.png)
![](스크린샷%202024-08-14%2017.15.50.png)
## References

- [Spring Rest Docs공식문서](https://docs.spring.io/spring-restdocs/docs/current/reference/htmlsingle/#introduction)
- [Asciidoctor 공식문서](https://docs.asciidoctor.org/asciidoc/latest/syntax-quick-reference/)
- [도움 받은 유튜브](https://www.youtube.com/watch?v=LEqaUEcWH7M)
