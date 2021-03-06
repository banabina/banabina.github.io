---
title: 라즈베리 파이 커널 빌드하기
excerpt: 라즈베리 파이 커널 빌드
categories:
- linux
tags:
- linux
toc: true
comments: true
toc_sticky: true
---

라즈베리 파이에서 리눅스 커널을 수정하고 이를 적용하려면 커널을 빌드하고 설치할 수 있어야 합니다. '디버깅을 통해 배우는 리눅스 커널의 구조와 원리'를 읽으면서 앞으로 커널 빌드할 일이 꽤 생길 거 같아서 관련 내용을 정리하려 합니다.  <br>

불필요한 권한 설정을 피하기 위해 root 권한을 획득하였다.
```java
$ sudo su
```
커널 소스를 다운 받을 디렉토리를 생성한다.
```java
$ mkdir rpi_kernel_src
```

## Step 1:  커널 소스 다운로드

라즈비안 소스 코드를 다운 받기 전에 필요한 리눅스 유틸리티 프로그램을 설치한다.
```java
$ apt-get install git bc bison flex libss-dev
```
<br>rpi_kernel_src 폴더에 커널 소스를 다운 받는다.
```java
cd rpi_kernel_src
git clone --depth=1 --branch rpi-4.19.y https://github.com/raspberrypi/linux
```
소스 코드를 다운 받는 데 5분정도 소요된다. --branch 옵션을 넣지 않으면 최신 커널 소스를 다운받게 된다. 
## Step 2:  커널 빌드
커널 빌드를 하기 위해 빌드 쉘 스크립터 작성하자. 쉘 스크립터의 이름은 build_rpi_kernel.sh이다.
<script src="https://gist.github.com/banabina/6e0fba11ed7460b116655476e4f53fcd.js"></script>


```java
01 #!/bin/bash
```
'#!'는 스크립트를 실행할 쉘을 지정하는 선언문이다. 즉,  사용 하려는 명령어 해석기가 bash Shell임을 알려주는 것이다.  <br>
 
```java
05 KERNEL_TOP_PATH="$( cd "$(dirname "$0")" ; pwd -P )"
```
KERNEL_TOP_PATH에 현재 작업 디렉토리를 저장하는 라인이다. 즉, KERNEL_TOP_PATH에는 home/pi/rpi_kernel_src가 들어가게 된다. 한 줄에 두 개의 명령어가 합쳐져 있는데, cd "$(dirname "$0")" 와 pwd -P이다. <br>
pwd \- P 는 현재 쉘의 절대 경로를 받아오는 명령어이다.  pwd \-P는  쉘의 위치에 따라 결과가 달라지기 때문에 앞에 추가적인 명령이 붙는다.
\$0는 명령어의 첫번째 인자를 의미한다. 여기서는 home/pi/rpi_kernel_src/build_rpi_kernel.sh 이다. dirname 은 문자열에서 디렉토리만 출력하는 명령어이다. dirname "\$0"을 한 결과는 home/pi/rpi_kernel_src이 된다. <br>
따라서 현재 디렉토리를 이동 후 절대 경로를 받아왔으므로 KERNEL_TOP_PATH에는 현재 작업 디렉토리가 저장된다.

 **Tip**<br>
 큰따옴표 안에 넣은 값은 변수가 실제 값으로 치환된 후 출력되고<br> 작은 따옴표로 감싸진 문자열은 변화없이 그대로 출력된다.<br> 세미콜론은 하나의 명령이 끝날 때 뒤에 붙여서 한 명령이 끝났음을 나타낸다.<br>
 {: .notice--info}
 
```java
16 make O=$OUTPUT bcm2709_defconfig
```
/home/p/rpi_kernel_src/linux/arch/arm/configs/bcm2709_deconfig 경로에 있는 bcm2709_decofig 파일에 선언된 컨피그 파일을 참조해 .config 파일을 생성한다.<br>
```java
19 make O=$OUTPUT zImage modules dtbs -j4 2>&1 | tee $BUILD_LOG
```
리눅스 커널 소스를 빌드하는 명령어이다.<br>
파일을 저장한 후에는 chmod 명령어를 입력해 파일에 실행 권한을 부여해야 한다.
```java
root@raspberrypi:/home/pi/rpi_kernel_src# chmod +x build_rpi_kernel.sh
```

<br>이제 작성한 빌드 스크립트를 실행한다.
```java
root@raspberrypi:/home/pi/rpi_kernel_src# ./build_rpi_kernel.sh
```
빌드를 완료하는 데는 3~4시간 정도 소요 된다. (raspberry pi 3 b+ 모델로 진행했을 때 그러했다.)

## Step 3:  커널 설치
빌드한 커널 이미지를 설치해보자.
라즈비안 이미지를 설치하는 셸 스크립트를 작성
<script src="https://gist.github.com/banabina/602f819b8b0d5fda1cea6c3aff377027.js"></script>
<br>다음 명령어로 셸 스크립트를 실행한다.
```bash
root@raspberrypi:/home/pi/rpi_kernel_src# ./install_rpi_kernel.sh
```
커널 빌드 시 에러가 발생했다면 반드시 수정하고 설치해야 한다. 그렇지 않으면 제대로 설치되지 않는다.

### 전처리 코드 생성
리눅스 커널 소스를 분석하다 보면 수 많은 매크로를 만난다. 이 매크로가 소스 분석의 걸림돌로 작용한다. 전처리 코드는 이러한 매크로를 풀어서 표현한다. 따라서 전처리 코드를 생성해서 리눅스 커널 코드를 분석하면 분석에 도움을 받을 수 있다.

셸 스크립트 build_preprocess_rpi_kernel.sh을 작성한다.
<script src="https://gist.github.com/banabina/3a4000fd37e26aea54be9747ac3b1aae.js"></script>
<br>파일을 저장한 후에는 chmod 명령어를 입력해 파일에 실행 권한을 부여한다.
```bash
root@raspberrypi:/home/pi/rpi_kernel_src# chmod +x build_rpi_kernel.sh
```

linux/kernel/sched/core.c 파일을 전처리 코드로 추출하려면 다음 현식으로 셸 스크립트를실행하면 된다
```bash
./build_preprocess_rpi_kernel.sh kernel/sched/core.i
```
소스 코드의 디렉토리를 잘 못 입력하면 에러 메시지와 함께 빌드가 종료된다. linux 폴더를 기준으로 한 소스 코드의 위치를 입력하면 된다.


**Reference** <br>
https://www.raspberrpi.org/documentation/linux/kernel/building.md<br>
디버깅을 통해 배우는 리눅스 커널의 구조와 원리<br>
http://egloos.zum.com/rousalome/v/10011640<br>
 {: .notice--info}
