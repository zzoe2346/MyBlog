---
title: RESTful 이란 무엇일까
tags:
  - Web
date: 2024-10-31
---
## 의문
Job Description 에 보면 항상 나오는게 RESTful API 개발경험입니다. 여태까지 `GET: {url}/document/1` 이런식으로 HTTP method 잘쓰고 URL 잘쓰는게 RESTful 한거겟지하고 막연히 생각했었는데, 이번기회에 한번 정리해 보려합니다. RESTful 이게 뭔데 맨날 공고에 있을까요? 별거 아닌거 같은데 항상 있는거 보면 뭔가 중요한게 있는것이 아닐까요?

## 해소
### 그런 REST API로 괜찮은가
https://www.youtube.com/watch?v=RP_f5dMoHFc&ab_channel=NAVERD2
> 강연덕분에 이해가 너무 잘되었다. 너무 좋은 강연...
- 기존의 인터넷 질서를 망가뜨리지 않고 HTTP 를 발전시킬 수 있는 방법을 고민하다가 등장
- REST 아키텍처 스타일을 따르는 api
- 제약조건의 집합
- http만따라도 앵간한거 다 잘지켜진다
- 단 uniform inteface 충족이 어려움
- 왜?
	- 독립적 진화를 위함
	- 서버의 기능이 변경되어도 클라를 업뎃할 필요 없다
	- REST 계기:"How do I improve HTTP without breaking the Web."
- HTTP 를 활용한다면 그걸 풍부하게 하면되네
- 결국 이거 캡슐화로 생각하면 되겠는데?
- 결국 중요한건 HTTP고 근본이다.
- REST 아키텍처가 HTTP 에 지속적으로 영향을 줌으로서 HTTP 가 진화됨.
- 웹의 독립적 진화를 위함
- self descriptive, HATEOAS
- REST를 구성하는 스타일
	- client - server
	- stateless
	- cache
	- uniform interface
	- layered system
	- code-on-demand(optional)

![[SCR-20241031-qqou.png|400]]
### HATEOAS가 왜 필요할까?

GPT 한테서 받은 자료

HATEOAS(Hypermedia as the Engine of Application State)는 REST API 설계 원칙 중 하나로, 클라이언트와 서버 간의 상호작용을 더욱 유연하고 확장성 있게 만들기 위한 방법입니다. HATEOAS를 사용하면 클라이언트는 서버가 제공하는 링크를 통해 현재 리소스와 관련된 액션을 탐색할 수 있습니다.

예를 들어, 사용자가 특정 리**소스의 상태를 조회할 때 HATEOAS가 적용된 API에서는 응답에 관련 작업(예: 리소스 수정, 삭제 등)에 대한 링크도 포함**됩니다. ==이로 인해 클라이언트는 별도의 엔드포인트 경로를 몰라도 리소스의 상태에 따라 제공된 링크를 따라 원하는 작업을 수행==할 수 있게 됩니다.

### HATEOAS의 장점
1. **확장성**: 클라이언트는 사전에 서버의 구조를 몰라도 동적으로 탐색하며 액션을 수행할 수 있습니다.
2. **자체 문서화**: 리소스와 관련된 링크들이 API 응답에 포함되어 있으므로, 클라이언트는 링크를 참조하며 자동으로 사용 방법을 이해할 수 있습니다.
3. **유연한 변경**: 서버의 엔드포인트가 변경되더라도 클라이언트는 ==링크 기반으로 동작==하므로 변경에 덜 민감합니다.

### 예시
```json
{
  "id": 1,
  "name": "정성훈",
  "points": 1000,
  "_links": {
    "self": { "href": "/users/1" },
    "add_points": { "href": "/users/1/add-points", "method": "POST" },
    "deduct_points": { "href": "/users/1/deduct-points", "method": "POST" }
  }
}
```

위 예시에서 `"_links"` 객체는 현재 리소스와 관련된 작업 링크를 제공합니다. HATEOAS는 REST API의 자율성을 높이고 유지보수를 쉽게 만들어주는 중요한 요소입니다.
## 생각 정리
RESTful 설계에 대해 공부하고 나서, REST의 모든 제약 사항을 철저히 지키는 것이 내부 API에 항상 필요한 것은 아니라는 생각이 들었습니다. 특히 클라이언트-서버 간에 긴밀한 통제가 가능한 내부 시스템이라면, 완벽하게 RESTful하게 구성하지 않아도 개발 효율성을 유지할 수 있을 것 같습니다.  

REST의 제약 사항 중에서도 특히 **self-descriptive 메시지**와 **HATEOAS가**는 가장 많이 간과되는 부분입니다. 물론, 이 두 가지 원칙을 잘 지킨다면 API의 유연성과 자율성, 그리고 확장성 측면에서 장점이 많겠지만, 내부 환경에서는 오히려 개발 복잡도와 부담이 늘어나는 측면도 있습니다. 

이러한 RESTful 원칙 대신, **회사 내부에서만 쓰이는 API라면 문서화에 집중**하는 것도 좋은 접근이라고 생각됩니다. 철저히 self-descriptive하거나 HATEOAS를 지키기보다는, **명확하고 쉽게 접근할 수 있는 문서**가 오히려 개발 속도와 커뮤니케이션에 긍정적 영향을 줄 수 있습니다. RESTful의 기본 원칙을 유연하게 적용하면서, 필요한 경우 문서화로 보완하는 것이 실용적이고, 클라이언트와 서버 간 자율성과 협업도 균형 있게 유지할 수 있다고 봅니다.
- 클라이언트-서버간 약속된 사항이기 때문에 개발/유지보수 과정에서 많은 의사소통 비용 절약
- 협업에 유리


## 참고자료
- https://www.redhat.com/en/topics/api/what-is-a-rest-api
- https://www.redhat.com/en/topics/integration/whats-the-difference-between-soap-rest
- https://aws.amazon.com/what-is/restful-api/