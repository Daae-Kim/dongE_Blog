---
title: "Redis 캐싱전략 - Cache Aside, Write Around"
datePublished: Sun Jun 01 2025 07:52:33 GMT+0000 (Coordinated Universal Time)
cuid: cmbdd57dx000509jvc23k7e9i
slug: redis-cache-aside-write-around
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1748764331311/75197ce5-3ce6-4da9-a113-e24635bd3d6c.png
tags: redis, redis-caching

---

## 캐시(Cache), 캐싱(Caching)이란?

캐시란, 원본 저장소보다 빠르게 가져올 수 있는 임시 데이터 저장소

### 캐싱이란?

캐싱이란, 캐시(임시저장소)에 접근해서 데이터를 빠르게 가져오는 방식을 의미한다.

현업예시

> “이 API는 응답 속도가 너무 느린데? 이 응답 데이터는 **캐싱(Cahing)** 해두고 쓰는 게 어때?’

## 데이터를 캐싱할때 사용하는 전략(Cache Aside, Write Around)

레디스를 캐스로 쓸때 어떤 방식으로 사용할지 전략이 다양하다. 그중에서 현업에서 가장 많이 사용되고 있는 전략 2가지를 배운다.

### Cache Aside(=Look Aside, Lazy Loading) 전략

데이터를 조회할 때 주로 사용하는 전략이 Cache Aside 이다.

> Cache Aside 전략은 캐시에 데이터를 확인하고, 없다면 DB를 통해 조회해오는 방식이다.

사용자가 데이터 조회를 요청했을때, 데이터 베이스로부터 바로 가져오지 않고, 조회전에 레디스에 있는지 먼저 확인한다.

레디스에 데이터가 없는것을 확인하고, 데이터베이스로부터 데이터를 조회해서 응답한다.

데이터베이스로부터 조회한 데이터를 응답한 뒤에 레디스에도 데이터를 저장해둔다.

이후 레디스에 조회하고자 하는 데이터가 있는지 확인하고, 데이터가 존재하면 레디스로부터 데이터를 바로 가져온다.

1. **Cache Hit : 캐시에 데이터가 있는 경우**
    
    데이터를 요청했을때 데이터가 있는 경우
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1748764455091/ec703deb-cf64-412e-a57c-8c2e3027bb61.png align="center")
    
2. **Cache Miss : 캐시에 데이터가 없는 경우**
    
    데이터를 요청했을때 캐시에 데이터가 없는 경우
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1748764482610/32028d6d-0826-45bd-8909-ff89f1803bd1.png align="center")
    

### Write Around 전략

데이터를 어떻게 사용할지(저장, 수정, 삭제)에 대한 전략이다. cache aside 전략과 함께 자주 활용되는 전략이다.

데이터를 저장할때 레디스에는 저장하지 않고, 데이터베이스에만 저장하는 방식이다. 그러다 데이터를 조회할때 레디스에 데이터가 없으면 데이터베이스로부터 데이터를 조회해와서 레디스에 저장시켜주는 방식이다.

> **Write Around 전략**은 **쓰기 작업(저장, 수정, 삭제)을 캐시에는 반영하지 않고, DB에만 반영하는 방식**을 뜻한다.