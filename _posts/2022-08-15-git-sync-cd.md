---
layout: article
title: git sync를 활용한 CD 구축
tags: kubernetes k8s sidecar 쿠버네티스 cicd git 
mode: immersive
article_header:
  type: overlay
  theme: dark
  background_color: '#123'
  background_image: false
aside:
  toc: true
author: author
---

# Init
### 글의 주제:
-  쿠버네티스 side car 패턴을 이해하고, git sync 오픈소스와 리눅스 inotify-tools 패키지를 사용해서 간결하고 빠른 cd 자동화를 구축해본다. <br>
<br>

<!--more-->

### 글 작성 배경: 
- 코드 테스트없이 코드의 실행 결과를 바로 배포하는 목적의 브랜치가 있었다. 간단한 파이프라인이므로 굳이 jenkins 웹 훅 트리거 ~ 도커 빌드 ~ argocd 배포의 과정을 거치는 게 의미가 있나? 하는 생각이 들어서 이 시간을 더 단축시킬 수 있을 거 같은 단순한 호기심과 도전 욕구로 인해 시작한 미니 프로젝트.

<br>

### 총평:
- (good) side car 패턴을 처음 사용해본 경험. 아주아주 좋은 기능임을 경험할 수 있었다.
- (good) inotify-tools를 알게 되면서 리눅스 활용을 더 잘하게 된 점.
- (good) 상상을 통해 여러 오픈 소스를 조합해서 나만의 상품을 만드는데 성공했다는 점.

-> git sync는 git 서버로 주기적인 핑을 날리므로 커넥션 수가 많아질 수록 서버 부하가 증가하는 단점이 발생함. 
sync 주기를 1초로 설정한 서버 5대를 gitlab 서버에 연결한 결과 서버 부하가 100%에 가깝게 올라가며 서버 UI에 접속하는 것 조차 힘들어졌다. 
서버 스펙은 vCPU 2core, RAM 8GB이다. 서버의 평상시 CPU 부하는 10% 수준으로 아주 안정적인 상태였다. 이걸 해결하기 위해 sync 주기를 10초로 낮추니까 부하가 25% 수준으로 떨어졌다.

<br>

# 아키텍쳐 설명
| `코드 흐름` | `설명` |
| ---  |  --- |
| ![아키텍쳐 코드 흐름](https://raw.githubusercontent.com/protocol-jylee/protocol-jylee.github.io/master/assets/images/git-sync-architecture.png) | git sync 컨테이너가 gitlab 서버에 연결됩니다. <br> git lab 서버로 커밋이 발생하면 git sync는 해당 코드를 가져와서 empty dir에 덮어쓰기 합니다. <br> 그렇게 empty dir에는 늘 최신 코드가 존재합니다. <br> 백엔드 서버는 empty dir에서 코드를 가져온 뒤 서버를 재실행합니다. |

<br>

# git sync
## git sync 사용
git sync 오픈소스는 [여기](https://github.com/kubernetes/git-sync/tree/v3.3.5)에서 다운받을 수 있습니다. 저는 환경변수를 사용하기 위해 `v3.3.5` 버전을 적용했습니다.

컨테이너 1과 컨테이너 2가 하나의 empty dir(git-sync-volume)를 바라보도록 볼륨 마운트 되어 있습니다.
```YAML
spec:
  containers:
    - name: backend-server # 컨테이너 1
      image: <my-image-repository>/<my-image>:1
      command: ['/entrypoint/entrypoint.sh']
      args: ["start"]
      tty: true
      stdin: true
      ports:
        - name: http
          containerPort: 8000
          protocol: TCP
      volumeMounts:
        - name: git-sync-volume
          mountPath: /git

    - name: git-sync # 컨테이너 2
      image: <my-image-repository>/git-sync:v3.3.5__linux_amd64
      args:
        - --repo=https://gitlab.kismi.re.kr/platform/cloud_automation.git
        - --branch=develop
        - --depth=1
        - --root=/git
        - --dest=synchronized
      volumeMounts:
      - name: git-sync-volume
        mountPath: /git
        
  volumes:
    - name: git-sync-volume
      emptyDir: {}
```

## git 서버 액세스 설정
git 서버에 read 및 write 접근 권한이 필요할 경우 계정 인증을 거쳐야합니다.
공식문서에서 제시하는 방법입니다.
| Environment | Flag | Description | Default |
| --- | --- | --- | --- |
| GIT_SYNC_USERNAME |	--username |	the username to use for git auth | "" |
| GIT_SYNC_PASSWORD | --password | the password or personal access token to use for git auth. (users should prefer --password-file or env vars for passwords)	 | "" |
| GIT_SYNC_PASSWORD_FILE | --password-file | the path to password file which contains password or personal access token (see --password) | "" |

### 1. kubernetes secret 사용
### 2. 도커파일 수정

<br>

# inotify-tools
## 소개
## 활용 방법

<!--more-->

---

이 글이 마음에 든다면 별 하나 쏴주세요! 글 작성에 큰 즐거움이 됩니다. :star2:

[![Star This Project](https://img.shields.io/github/stars/protocol-jylee/protocol-jylee.github.io.svg?label=Stars&style=social)](https://github.com/protocol-jylee)
