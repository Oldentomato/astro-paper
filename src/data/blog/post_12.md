---
author: Jowoosung
title: EKS 인프라 구조  
featured: true
draft: false
slug: EKS-인프라-구조  
pubDatetime: 2025-08-28T15:22:00Z
modDatetime: 2025-08-28T16:52:45.934Z
description: 최종 프로젝트 인프라 구조 소개 
tags: 
  - infra
---  

## Table of contents

## aws eks 구조  
![img1](https://github.com/Oldentomato/PortFolio_Next/blob/main/postsimg/post_12/sos_aws_architecture.png?raw=true)  

현재 구성된 전체적인 aws구조이다. 활용한 aws의 서비스들은 eks, vpc, vpn, alb, ecr, efs이다.  
서브넷은 public, private 각각 3개씩 사용하고 있고, public에는 ALB만 두고 나머지 서비스들은 private 서브넷에만 구성했다.  
ALB도 IGW용과 NAT용으로 2개를 만들어, 외부접속과, 내부접속용으로 나누어 구축했다.  
각 서비스의 설명은 아래와 같다.  

### VPC  
![img2](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*hZGJeN-4F6fLtus5XBJC_w.png)  
VPC는 자체 데이터 센터에서 운영하는 기존 네트워크와 아주 유사한 가상 네트워크이다.  
VPC가 없다면 EC2 인스턴스들이 서로 거미줄처럼 연결되고 인터넷과 연결된다.  
이런 구조는 시스템의 복잡도를 엄청나게 끌어올릴뿐만 아니라 하나의 인스턴스만 추가되도 모든 인스턴스를 수정해야하는 불편함이 생긴다.  
마치 인터넷 전용선을 다시까는것과 같다.  
![img3](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*Ehn4uEQMtbmdPsU6MxVc3Q.png)  
VPC를 적용하면 위 그림과같이 VPC별로 네트워크를 구성할 수 있고 각각의 VPC에따라 다르게 네트워크 설정을 줄 수 있다.  
또한 각각의 VPC는 완전히 독립된 네트워크처럼 작동하게된다.  

### VPN  
VPN은 한국말로 가상 사설망이라고 한다.  
쉽게 설명하면 인터넷 망에서 논리적으로 직접 연결한 것과 같은 전용망을 만들어 연결하는 기술이다.  
이렇게 직접 연결한 것같이 만드는 통로를 터널(Tunnel)이라고 한다.  
인터넷은 모두가 사용하는 공간이기 때문에 보안에 취약하다.  
그래서 다른 서버에 접속할 경우 VPN 기술을 통해 암호화한 터널을 만들어서 통신하면 보안을 강화시킬 수 있다.  
자세한 내용 및 구성방법은 아래 링크에 있다.  
[이동하기](https://wsportfolio.vercel.app/blog/post_10)  

### ALB  
ALB는 AWS에서 제공하는 4가지 로드밸런서 중 하나로 OSI 7Layer에서 일곱번째 계층에 해당하는 Application Layer를 다루는 로드밸런서이다.  
주요 특징으로는 다음과 같다.  
- HTTP를 활용한 라우팅(부하분산)  
- 로드밸런싱(부하 분산) 방식  
- SSL 인증서 탑재 가능  
- 프록시 서버로서의 역할  
- AWS Web Application Firewall(WAF) 연동  


### ECR  
AWS에서 제공하는 Docker Hub와 비슷한 개념으로, Amazon Elastic Container Registry의 약자로 안전하고 확장 가능하고 신뢰할 수 있는 AWS 관리형 컨테이너 이미지 레지스트리 서비스이다.  
Docker Hub와 동일하다고 볼 수 있지만 장점으로는 S3로 Docker Image를  관리하므로 고가용성을 보장하고, AWS IAM 인증을 통해 이미지 push/pull에 대한 권한 관리가 가능하다.  


### EFS  
![img4](https://blog.kakaocdn.net/dna/bfOSCy/btsFK5HHuXo/AAAAAAAAAAAAAAAAAAAAABf_xiW5DSsqRYltfC2fkRQIBgL46RKl4IKggGmjH49J/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1756652399&allow_ip=&allow_referer=&signature=fy8YOWKGsr4wdxkYwvmKxburCcE%3D)  
AWS 클라우드 서비스와 온프레미스 리소스에서 사용할 수 있는 간단하고 확장 가능하며 탄력적인 파일 스토리지를 제공하는 서비스이다.  
EFS는 리눅스 인스턴스를 위한 확장성, 공유성 높은 파일 스토리지로, EC2 Linux 인스턴스에 마운트된 Network File System을 통해 VPC에서 필요한 파일에 접근하거나 AWS Direct Connect로 연결된 온프레미스 서버의 파일에 접근할 수 있다.  

## aws infra 구조  
![img5](https://github.com/Oldentomato/PortFolio_Next/blob/main/postsimg/post_12/sos_infra_architecture.drawio.png?raw=true)  
인프라 구성도이다.  
사용된 기술스택은 다음과 같다.  
- kafka, zookeeper, kafkaUI
- elastic_search, kibana
- postgres
- redis
- spring_boot
- fastAPI
- airflow 
- react
- prometheus, grafana
- redmine
- adminer

위 시스템들은 ALB를 제외하고 모두 private subnet을 통해 서로 통신하고 있고, ALB는 NAT Gateway로 두어 내부에서 외부로만 Internet이 연결이 되도록 구성했다.  
그리고 다른 하나의 ALB는 IGW로 두고 FrontEnd만 라우팅하여 다른 시스템에 접근 못하도록 구성했다.  
Kibana나 Potainer 처럼 개발자들만 접근해야하는 서비스들은 Nat Gateway가 연결된 ALB에 라우팅을 설정하고 VPN으로 터널링 후에 접속할 수 있는 형태이다.  

## 데이터 수집 매커니즘  

## 사용자 요청 매커니즘  

## 후기  



