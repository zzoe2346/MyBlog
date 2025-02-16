---
title: squash merge 로 인한 git conflict 분석 및 해결
tags:
  - Git
  - SquashMerge
  - Github
  - Troubleshooting
date: 2024-09-29
categories: 트러블슈팅
summary:
---
## 문제 정의
- 팀의 브랜치 전략에 대해 우선 설명하자면
	- 일주일동안 작업한것을 Develop에 머지하고 문제 없으면 Master로 머지하는 방식
	- 작업 브랜치는 각자 생성하고 그 브랜치를 Weekly 브랜치에 PR후, 코드리뷰 후 Approve를 받아 머지시킴
	- 커밋 로그를 깔끔히 관리하기 위해 Squash Merge를 활용하기로 결정
- 문제 발생
	- Weekly -> Develop 머지시 All Conflict 가 발생했고 당연히 Develop -> Master 머지도 불가했다.

## 문제 분석
- Squash Merge 시행시 기존 커밋 히스토리가 하나의 커밋으로 합쳐지며 새로운 히스토리 하나로 통합된다
- 이러면 Weekly 에서 Squash Merge 된 커밋을 Develop 에서는 커밋 히스토리 추적이 불가하다. 커밋 ID로 히스토리가 추적되는데 Develop 입장에서는 Weekly에서 Squash 된  새로운 커밋은 생판 처음보기 때문
- 아래의 그림은 팀원에게 설명하기 위해 그린 그림.

![](Pasted%20image%2020241104013359.png)
## 문제 해결 과정
- Weekly -> Develop 은 Squash Merge
- Develop -> Master 는 일반 Merge 

![Pasted image 20241104013428.png](Pasted%20image%2020241104013428.png)
## 개선 결과
- **깔끔한 커밋 로그를 그대로 유지**하면서 **충돌상황이 더이상 발생 안하게 됨**
- 팀원들과 본인들이 가진 두루뭉실 했던 커밋개념을 명확히 하는 계기가 됨