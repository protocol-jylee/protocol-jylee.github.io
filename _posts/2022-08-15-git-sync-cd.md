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

| **코드 흐름** | **설명** |
| ---  |  --- |
| ![아키텍쳐 코드 흐름](https://raw.githubusercontent.com/protocol-jylee/protocol-jylee.github.io/master/assets/images/git-sync-architecture.png) | git sync 컨테이너가 gitlab 서버에 연결됩니다. <br> git lab 서버로 커밋이 발생하면 git sync는 해당 코드를 가져와서 empty dir에 덮어쓰기 합니다. <br> 그렇게 empty dir에는 늘 최신 코드가 존재합니다. <br> 백엔드 서버는 empty dir에서 코드를 가져온 뒤 서버를 재실행합니다. |

<br>

# git sync
## git sync 사용
git sync 오픈소스는 [여기](https://github.com/kubernetes/git-sync/tree/v3.3.5)에서 다운받을 수 있습니다. 저는 환경변수를 사용하기 위해 `v3.3.5` 버전을 적용했습니다.

컨테이너 1과 컨테이너 2가 하나의 empty dir(git-sync-volume)를 바라보도록 볼륨 마운트 되어 있습니다.
```yaml
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
      image: k8s.gcr.io/git-sync/git-sync:v3.3.5
      args:
        - --repo=<my git project repository>
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
공식문서에서 제시하는 방법입니다. <br>

| **Environment** | **Flag** | **Description** | **Default** |
| --- | --- | --- | --- |
| GIT_SYNC_USERNAME |	--username |	the username to use for git auth | "" |
| GIT_SYNC_PASSWORD | --password | the password or personal access token to use for git auth. (users should prefer --password-file or env vars for passwords)	 | "" |
| GIT_SYNC_PASSWORD_FILE | --password-file | the path to password file which contains password or personal access token (see --password) | "" |

이 값을 적용하는 방법은 아래 2가지가 있습니다.

### 1. kubernetes secret 사용
시크릿을 사용할 경우 시크릿을 만듭니다. 파드에 마운트해서 바로 쓰기 위해서 키 이름은 공식문서에서 지정한 키로 설정합니다.
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-sync-auth
  namespace: <my-namepsace>
type: Opaque
stringData:
  GIT_SYNC_USERNAME: "jylee"
  GIT_SYNC_PASSWORD: "mypassword" # or access token
```

위 시크릿을 git sync 컨테이너에 환경변수로 마운트 해줍니다.
```yaml
- name: git-sync # 컨테이너 2
  image: k8s.gcr.io/git-sync/git-sync:v3.3.5
  args:
    - --repo=<my git project repo>
    - --branch=develop
    - --depth=1
    - --root=/git
    - --dest=synchronized

  ## 추가 ##
  envFrom: 
  - secretRef:
      name: git-auth-auth
  ## 추가 ##

  volumeMounts:
  - name: git-sync-volume
    mountPath: /git
```

### 2. git sync 이미지 빌드 시 access token 박제하기
먼저 golang을 설치합니다. (얼마 안 걸립니다!!)
```bash
wget https://go.dev/dl/go1.18.3.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.18.3.linux-amd64.tar.gz
vi ~/.bashrc
   export GOROOT=/usr/local/go
   export GOPATH=$HOME/go
   export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
source .bashrc
go version
```

git 서버에 access token을 발급받습니다.
저의 경우 최소 권한 원칙에 의거하여 read 권한만 할당한 토큰을 사용했습니다.
```Dockerfile
# Dockerfile.in
FROM {ARG_FROM} as prep
...
ENV GIT_SYNC_USERNAME="root"
ENV GIT_SYNC_PASSWORD="2V***************"

ENTRYPOINT ["/{ARG_BIN}"]
```

이제 make 명령으로 이미지를 빌드합니다.
({ARG_FORM} 과 같은 변수는 사용자가 할당하지 않을 시 기본값이 할당됩니다.)
```bash
make container REGISTRY=<my.image.repository> VERSION=v3.3.5
```

빌드가 완료된 뒤, 이 이미지로 git sync를 실행합니다.
```yaml
- name: git-sync # 컨테이너 2
  image: <my.image.repository>/git-sync:v3.3.5__linux_amd64 # 내 이미지
  args:
    - --repo=<my git project repo>
    - --branch=develop
    - --depth=1
    - --root=/git
    - --dest=synchronized

  volumeMounts:
  - name: git-sync-volume
    mountPath: /git
```
여기까지 git sync 적용을 알아보았습니다. <br>
다음은 git sync로 가져온 코드를 백엔드 서버가 어떻게 가져갈 지 설명합니다.
<br>

# inotify-tools
## 소개
파일 및 디렉터리에 대한 변경 사항을 감지하는 리눅스 패키지 입니다.
```bash
apt-get install inotify-tools -y
```
이 패키지의 옵션이 참 많아서 전부 설명하는 건 시간상 무리네요.. <br>
참고할 수 있는 [링크](https://hmjkor.tistory.com/502)를 남깁니다.
저는 제가 적용한 부분만 설명하고 포스팅을 마치겠습니다.

## 활용 방법
git sync로 코드 업데이트 이벤트 발생시 해당 디렉터리가 지워지고 같은 이름의 디렉터리가 생성됩니다. <br>
저는 `/git/synchronized` 경로에 코드 업데이트를 지정했으니까 `synchronized` 디렉터리가 지워졌다가 새로 생기는 방식입니다.
`synchronized`의 변경사항을 감지하려면 `/git` 디렉터리 전체를 감시하면 됩니다. <br>

제가 알아낸 바에 의하면 `synchronized` 디렉터리의 변경사항을 감지하면 프로젝트 파일 수만큼 변경사항이 감지되어 수천개의 이벤트가 발생했습니다. <br>

반면 `/git` 디렉터리의 변경사항을 감지하면 `synchronized` 디렉터리 삭제에 대한 이벤트만(1개) 감지돼서 핸들링하기 수월했습니다. 아직까지 이 방법으로 잘 쓰고 있기 때문에 더 많은 테스트는 하지 않았습니다.

이제 아래 코드를 스크크립트 파일로 만들어서 백엔드 서버에서 실행해주면 empty dir에서 최신코드가 생길 때 마다 백엔드 서버 프로젝트 디렉터리로 가져오게 됩니다.
```bash
#!/bin/sh
set -e

inotifywait -m -e delete "/git" |
while read dirname eventlist filename
  do
    cp -fR /git/synchronized/tests /<backend-project-dir>
    cp -fR /git/synchronized/app /<backend-project-dir>
    cp -fR /git/synchronized/Pipfile /<backend-project-dir>

    cd /<backend-project-dir>

    <your project build or start code>
  done
```
<!--more-->

---

이 글이 마음에 든다면 별 하나 쏴주세요! 글 작성에 큰 즐거움이 됩니다. :star2:

[![Star This Project](https://img.shields.io/github/stars/protocol-jylee/protocol-jylee.github.io.svg?label=Stars&style=social)](https://github.com/protocol-jylee)

<br>
글을 읽다 궁금한 점은 여기에 문의해주시면 빠르게 답변해드리고 있습니다. :star2:

[![재윤이 카카오톡 오픈채팅방](https://raw.githubusercontent.com/protocol-jylee/protocol-jylee.github.io/master/assets/images/kakao_logo.png)](https://open.kakao.com/o/gVTq5Pve)
