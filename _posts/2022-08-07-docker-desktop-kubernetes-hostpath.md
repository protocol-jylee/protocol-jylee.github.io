---
layout: article
title: docker desktop 쿠버네티스의 hostpath 마운트 경로 찾기
tags: kubernetes k8s 쿠버네티스 docker-desktop hostpath
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
- docker desktop 쿠버네티스(<strong>이하 docker 쿠베</strong>)의 hostpath 사용 시, 마운트 패스와 노드에 실제 마운트 된 경로가 서로 다르다는 걸 알자. <br>
<br>

<!--more-->

### 글 작성 배경: 
- jenkins helm chart를 배포해보려고 docker 쿠베를 써봤는데 mount path permission denied. 오류 때문에 시간 많이 잡아먹은게 열 받아서 기록을 남김. <br>
<br>

### 총평:
- (good) 처음 마주한 에러는 init 컨테이너에서 마운트 패스에 대한 쓰기 권한 부족이었다. PV로 hostpath를 설정한 뒤 마운트 접근 권한을 위해 Security Context를 적용해본 점은 좋은 시도였다고 생각한다. 
- (bad) 위 시도 후에도 같은 에러가 났는데, 방법을 바꾸지 않고 storageclass를 설정하고 hostpath의 다른 옵션이 있는지 찾아보고 에러 내용을 구글링 해보는 등 비슷한 방법을 계속 시도한 것은 시간 낭비 + 고집이었던거 같다. 
- (good) 바람 한 번 쐰 후 다시 노트북 앞에 앉았다. hostpath 오류가 났으나 배포 성공 이후에 난 오류라는 점을 통해 노드에 해당 경로가 있겠다는 추론을 한다. 노드에 쉘 접속 후 마운트 패스를 검색하여 진짜 마운트 패스를 알아내고 문제 해결.

-> 내가 파악한 범위에서 꼭 필요한 키워드만 뽑아서 검색하고 해결방법을 적용해본다. 그래도 안되면 아예 다른 접근을 시도해본다.

<br>

# 테스트에 사용된 hostpath YAML이에요.
[쿠버네티스 공식문서](https://kubernetes.io/ko/docs/concepts/storage/volumes/#hostpath-fileorcreate-example)에서 가져왔습니다!
```
apiVersion: v1
kind: Pod
metadata:
  name: test-webserver
  namespace: jenkins
spec:
  containers:
  - name: test-webserver
    image: k8s.gcr.io/test-webserver:latest
    volumeMounts:
    - mountPath: /var/local/aaa
      name: mydir
    - mountPath: /var/local/aaa/1.txt
      name: myfile
  volumes:
  - name: mydir
    hostPath:
      # 파일 디렉터리가 생성되었는지 확인한다.
      path: /var/local/aaa
      type: DirectoryOrCreate
  - name: myfile
    hostPath:
      path: /var/local/aaa/1.txt
      type: FileOrCreate
```
<br>

# AKS에서 hostpath는 마운트 패스가 직관적이다.
hostpath에서 지정한 마운트 패스와 노드에 생성된 경로는 <strong>일치</strong>한다.
```
## local - 노드 이름 검색
jylee@DESKTOP-P0DHH5B:~/workspace$ ktl get node aks-nodepool1-75262664-vmss000002
NAME                                STATUS   ROLES   AGE   VERSION
aks-nodepool1-75262664-vmss000002   Ready    agent   18d   v1.21.14

## local - 노드 쉘 접속
jylee@DESKTOP-P0DHH5B:~/workspace$ ktl node-shell aks-nodepool1-75262664-vmss000002
spawning "nsenter-h8wy8d" on "aks-nodepool1-75262664-vmss000002"
If you don't see a command prompt, try pressing enter.

## node shell - hostpath의 마운트 경로 확인
root@aks-nodepool1-75262664-vmss000002:/# ls
NOTICE.txt  bin  boot  dev  etc  home  initrd.img  initrd.img.old  lib  lib64  lost+found  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var  vmlinuz  vmlinuz.old

root@aks-nodepool1-75262664-vmss000002:/# ls /var/local/aaa
1.txt
```
<br>

# docker 쿠베에서 hostpath는 '여기' 밑에 설정된다.
```
## local - 노드 이름 검색
jylee@DESKTOP-P0DHH5B:~/workspace$ ktl config use-context docker-desktop 
Switched to context "docker-desktop".

## local - 노드 쉘 접속
jylee@DESKTOP-P0DHH5B:~/workspace$ ktl node-shell docker-desktop
spawning "nsenter-l9gujy" on "docker-desktop"
If you don't see a command prompt, try pressing enter.

## node shell - hostpath의 마운트 경로가 없다..
docker-desktop:/# ls /var/local/aaa
ls: /var/local/aaa: No such file or directory

## node shell - 여기 있다.
docker-desktop:/# ls /containers/services/docker/rootfs/var/local/aaa/1.txt 
/containers/services/docker/rootfs/var/local/aaa/1.txt
```
지나고보면 별거 아니지만, 처음보는 경우에는 닟설어서 알던것도 속이게 된다. 
전혀 예상 못한 문제였기 때문에 더 찬찬히 덤볐어야 했다. 다시한번 느낀다. 틀을 깨고 생각하자.

<!--more-->

---

이 글이 마음에 든다면 별 하나 쏴주세요! 글 작성에 큰 즐거움이 됩니다. :star2:

[![Star This Project](https://img.shields.io/github/stars/protocol-jylee/protocol-jylee.github.io.svg?label=Stars&style=social)](https://github.com/protocol-jylee)
