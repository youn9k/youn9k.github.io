---
title: "[Server]AWS EC2 npm install 멈추는 현상"
date: 2025-05-19 12:00 +0900
categories:
  - ⚙️ Automation
  - Workflow
tags:
  - AWS
  - EC2
  - npm
---

개인 프로젝트 서버를 AWS EC2 인스턴스에 배포하는 과정에서, 서버가 중간에 멈추는 문제를 마주했습니다.

제가 사용 중인 인스턴스는 t2.micro 로, 즉 프리티어에서 제공되는 최소 사양 인스턴스였습니다.

- vCPU: 1개
- RAM: 1GB

서버를 실행했을 때 반응이 없어 AWS 대시보드에서 확인해보니, CPU 사용량이 100%에 가까운 상태로 장시간 유지됨을 확인할 수 있었습니다.

![대시보드 CPU 사용량 이미지](/assets/img/post/2025/021.png)

또한 인스턴스에 접속해 `top` 명령어로 리소스 사용량을 확인한 결과, DB 서버(mysqld)가 메모리의 40% 이상을 점유하고 있는 상황으로 가용 메모리가 약 150MB 밖에 되지 않는 상황이었습니다.

![top 명령어](/assets/img/post/2025/022.png)

이러한 리소스 부족으로 인해 Nest.js 서버를 실행하는 명령어인 `npm run start` 나, `npm run build` 시 서버가 멈추거나 오래 걸리는 문제가 발생하고 있었습니다. 

Nest.js는 Node.js 기반 서버사이드 프레임워크로, npm install 시 많은 패키지들을 의존성으로 가져오고, 실행 시에도 런타임/컴파일 메모리 요구량이 존재합니다.

t2.micro는 RAM이 1GB뿐이라 여유 메모리가 거의 없어, 스레드가 바빠지고 blocking이 일어나 서버가 멈춘다고 예상했습니다.

가장 쉬운 해결 방법으로는 인스턴스 스펙을 올리는 것이지만, 프로젝트 규모가 작기에 t2.micro를 유지하면서 해결 방법을 찾기로 했습니다.

비슷한 문제를 겪는 사례들이 있었고, 가상 메모리를 활용해 문제를 해결하는 방식을 사용할 수 있었습니다.

아래는 Ubuntu 에서 가상메모리를 할당하는 명령어입니다. 

```bash
# 1GB 스왑공간 생성
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
# 스왑 공간 활성화
sudo mkswap /swapfile
sudo swapon /swapfile
# 스왑 공간 확인
sudo swapon  --show
sudo free -h
# 부팅 시 자동 적용
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

재할당 하려면 아래와 같이 기존 스왑 공간을 제거하고 재할당할 수 있습니다.
```bash
# 기존 스왑 공간 비활성화, 제거
sudo swapoff /swapfile
sudo rm /swapfile

# 새 용량으로 할당
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile

# 스왑 공간 활성화
sudo mkswap /swapfile
sudo swapon /swapfile
```



## 결과

Nest.js + MySQL 띄웠을 때 물리 메모리 400MB, 가상 메모리 400MB 정도를 사용하고 있는 걸 알 수 있습니다.

![](/assets/img/post/2025/023.png)



## 참고

https://stackoverflow.com/questions/66693201/npm-install-hangs-forever-in-ec2
https://marklee1117.tistory.com/163