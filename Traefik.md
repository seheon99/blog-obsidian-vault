---
title: 트래픽을 다루는 Traefik
publish: false
tags:
  - Traefik
description:
---
## Traefik

> [Traefik GitHub](https://github.com/traefik/traefik)
> Traefik (pronounced *traffic*) is a modern HTTP reverse proxy and load balancer that makes deploying microservices easy. Traefik integrates with your existing infrastructure components (Docker, Swarm mode, Kubernetes, Consul, Etcd, Rancher v2, Amazon ECS, ...) and configures itself automatically and dynamically. Pointing Traefik at your orchestrator should be the *only* configuration step you need.

트래픽은 그 이름에서도 알 수 있다시피 인터넷 트래픽을 다루는 프로그램입니다. 트래픽은 컴퓨터에 들어오는 트래픽을 잡아 분석해 적절한 서비스로 라우팅하는 역할을 합니다. 특히 트래픽을 다루는 다른 프로그램과는 달리 적절한 서비스를 판단하는 설정을 자동으로 판단해 동적으로 업데이트 할 수 있습니다.

![[1.png]]