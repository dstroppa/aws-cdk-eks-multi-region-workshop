---
title: 실습 1, EKS 클러스터 생성하기
weight: 400
pre: "<b>1-4. </b>"
---

![](/images/20-single-region/intro.svg)

CDK로 반복 재사용 가능한 Construct를 작성해서 하나의 리전에 EKS 클러스터를 배포하고 그 위에 K8S 자원을 생성합니다.  
이를 통해 기존에는 별개로 관리 해왔던 인프라 정의 코드와, 라이프사이클이 긴 컨테이너를 한 번에 정의할 수 있습니다.