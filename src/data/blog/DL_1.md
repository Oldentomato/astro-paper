---
author: Jowoosung
title: 딥러닝 모델 확장자
featured: false
draft: false
slug: 딥러닝-모델-확장자
pubDatetime: 2023-01-19T15:22:00Z
modDatetime: 2023-01-19T16:52:45.934Z
description: h5, ckpt, pb 의 차이점에 대하여
tags: 
  - ai
---  

## Table of contents

## ckpt 파일  
일반적으로 이야기하는 ckpt 파일은 ckpt-data와 같으며, 딥러닝 모델을 제외한 학습한 가중치(weight)만 있는 파일. 모델 구조(graph)는 저장하지 않는다.  
- .ckpt-meta : 모델(graph)만 있는 파일  
- .ckpt-data : 딥러닝 모델을 제외한 학습한 가중치(weight)만 있는 파일. 모델 구조(graph)는 저장하지 않는다.  

## pb 파일  
모델 구조와 가중치(weight) 모두 저장된 파일. freeze_graph.py를 통해서 만들 수 있고,'그래프를 프리징시킨다.'라고 하면 pb파일을 만들 것이라는 뜻이다.  

## h5 파일  
Hierarchical Data Format (HDF)형식으로 저장되는 데이터. Keras에서는 모델 및 가중치(weight) 모두를 가지고 있는 파일이다.  

### 참조  
[참조](https://i-am-eden.tistory.com/5)