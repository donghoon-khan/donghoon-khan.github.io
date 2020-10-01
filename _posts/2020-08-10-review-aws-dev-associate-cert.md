---
title: AWS DVA 취득 후기
category: 
- DevOps
tags:
- aws
summary: Review AWS Certified Developer - Associate Certification
thumbnail: /assets/img/thumbnail/aws.png
---
회사에서 진행하는 프로젝트 덕분에 약 1년간 AWS를 접할 기회가 있었다. 주로 EKS를 구축하고 사용하는 과정에서 IAM이나 EC2, LB, EBS등을 접할 수 있었다. 말 그대로 접해본 수준이지 깊은 지식이 있던 상태가 아니었는데, DevOps에 흥미가 생기면서 클라우드를 제대로 공부해보자라는 생각이 들었다. 이후 관련 자격을 알아보던 중 `AWS Developer Associate` 자격이 눈에 들어왔고 취득하기로 마음 먹었다. 약 한달정도 공부 후 합격 했는데 간단한 후기를 적어보자.

## AWS Certification
AWS는 다양한 교육프로그램 뿐만 아니라 다양한 자격 검증을 지원한다.  
![AWS Cert list](/assets/img/posts/2020-08-10-review-aws-dev-associate-cert-list-aws-cert.png)  

Foundational 부터 Specialty까지 난이도도 다양하고 Architect, Developer등 분야도 다양하다. [AWS Certification 설명](https://aws.amazon.com/certification/certification-prep/)을 보고 자신한테 맞는 자격증을 선택하면 된다. 나는 DevOps에 관심이 있어서 중간 과정으로 `AWS Developer Associate`를 선택했다.

## 시험 접수
시험 접수는 간단하다.  AWS 계정과 해외결제가 가능한 카드를 준비한 이후 [시험 접수 페이지](https://www.certmetrics.com/amazon/)에서 원하는 자격의 스케줄을 보고 신청하면 된다. 시험비용은 150달러다. 환율 + 해외결제 수수료 하니까 19만원 조금 넘게 결제됬던거 같은데 너무 비싸다고 생각한다.  
시험 접수 시 비영어권 화자들이 영어로 시험치면 추가시간을 요청할 수 있다고 하는데, 나는 한글로 시험을 신청했고 시간도 한시간 넘게 남기고 나왔다. 관심있는 사람들은 신청하자.

## 시험 준비
`AWS Developer Associate` 관련 구글링을 해보니 자격 시험 후기글이 많이 있었다. 하나같이 Udemy에서 [Ultimate AWS Certified Developer Associate 2020](https://www.udemy.com/course/aws-certified-developer-associate-dva-c01/)강의를 수강했다고 하길래 나도 따라서 결제했다. 
강의를 듣다보면 상수 값들이 자주 나오는데(SQS 내 Message 보관 기간, dynamoDB 보조 인덱스 최대 할당량 등)문제 풀면서 외우자는 마음으로 가볍게 듣고 넘어갔다. 강의를 들으면서는 외운다는 생각보다는 어떤 서비스가 어떤 목적으로 만들어졌고 어떤 문제를 해결할 수 있다 라는 정도만 알고 넘어갔다.  
이후 [Practice Exams | AWS Certified Developer Associate 2020](https://www.udemy.com/course/aws-certified-developer-associate-practice-tests-dva-c01/)를 구매해서 풀었는데, 이 연습문제가 도움이 많이 되었다. 만약 시간이 부족해 강의를 다 듣지 못하면 연습문제라도 풀어보자. 문제의 답변 부분에 해설과 더불어서 친절하게 AWS Docs 링크까지 걸려 있어서 이해하는데 어려움은 없었다.
연습문제는 보너스 문제 포함 5셋트가 있는데 처음 풀 때는 60% ~ 65% 사이에서 점수가 왔다갔다 했다. 오답노트 정리 후 2회차 때부터는 80% 이상으로 점수가 나와서 틀린 부분만 중점적으로 공부하고 시험을 쳤다.

## 시험
시험은 집에서 가까운 송파 SRTC시험장에서 쳤다. 꼭 `신분증`과 `신용카드`를 준비해서 가야 한다. 엄격하게 신분확인을 하니까 무조건 준비해서 가자.  
나는 여유롭게 시험시간보다 30분 정도 전에 갔는데, 바로 시험을 응시할 수 있다고 해서 바로 시험을 쳤다. 연습문제와 비슷한 문제는 많은데 완전 똑같은 문제는 몇 없었던 걸로 기억한다. 단순히 문제와 답만 외운다면 불합격할 확률이 높다.  
한국어로 신청했어도 영어로 변환이 가능 해서 이해가 안되거나 낯선 지문, 선택지는 영어로 변환해서 확인했다.  
플래그 기능도 제공하기 때문에 헷갈리는 문제는 플래그 처리하고 나중에 다시 풀 수 있었다.  

## 시험결과
시험 결과는 모든 시험문제를 제출하고, 설문까지 완료하면 바로 나온다. pass가 적혀 있으면 합격인데, 3 ~ 5일 내로 메일로 점수를 알려준다고 적혀있었던 것 같다. pass확인 후 바로 퇴실해서 제대로 기억이 나지 않는다.  
시험치고 다음날 점수를 확인할 수 있다는 내용의 메일을 받았는데, [AWS Certmetrics](https://www.certmetrics.com/amazon/)에서 취득 점수와 인증서를 다운받을 수 있었다.
![dhkang-aws-dva-cert](/assets/img/posts/2020-08-10-review-aws-dev-associate-cert-dhkang-cert.png)  

페이스북이랑 링크드인에 AWS 뱃지를 추가할 수 있다고도 하던데 나중에 추가해야겠다.  
이제 [Certified Kubernetes Application Developer](https://www.cncf.io/certification/ckad/)를 준비해야겠다. CKAD는 300달러던데... 더 열심히 준비해야겠다.
