---
layout: default
title: k8s 시리즈 01 - 도커 설치 
nav_order: 5
---

# 도커설치

[https://docs.docker.com/engine/install/ubuntu/](https://docs.docker.com/engine/install/ubuntu/)

기존 docker 삭제 

```bash
$ sudo apt-get remove docker docker-engine docker.io containerd runc
```

패키지 업데이트 및 레파지토리 https 허용

```bash
$ sudo apt-get update

$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```

도커 공식 GPG 키 추가 

```bash
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

추가된 키 확인 

```bash
$ sudo apt-key fingerprint 0EBFCD88

pub   rsa4096 2017-02-22 [SCEA]
      9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid           [ unknown] Docker Release (CE deb) <docker@docker.com>
sub   rsa4096 2017-02-22 [S]
```

도커 레파지토리 추가

```bash
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

도커 설치

```bash
$ sudo apt-get update
$ sudo apt install docker-ce
```

도커 확인

```bash
$ sudo docker run hello-world
```