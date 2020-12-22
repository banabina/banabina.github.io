---
title: 라즈베리 파이 커널 빌드하기
excerpt: 라즈베리 파이 커널 빌드
categories:
- Linux
tags:
- Linux
toc: true
comments: true
toc_sticky: true

---

**'디버깅을 통해 배우는 리눅스 커널의 구조와 원리' 책의 실습을 진행하였다.<br><br>

실습 내용은 아래와 같다
1. 라즈비안 리눅스 소스 코드를 내려 받기
2. 커널 소스 빌드
3. 커널 설치

커널을 빌드, 설치하는 과정은 앞으로 실습에서도 많이 사용될 것 같아 내용을 정리하려 한다.<br><br>

불필요한 권한 설정을 피하기 위해 root 권한을 획득 후 진행
```java
$ sudo su
```
커널 소스를 다운 받을 디렉토리를 생성
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
git clone --depth=1 --branch rpi-4.19y https://github.com/raspberrypi/linux
```
소스 코드를 다운 받는 데 10 분 정도 소요된다. 브랜치를 rpi-4.19.y로 지정해주자.

## Step 2:  커널 빌드
커널 빌드를 하기 위해 빌드 쉘 스크립터 작성하자. 쉘 스크립터의 이름은 build_rpi_kernel.sh이다.
```java
01 #!/bin/bash
02 
03 echo "configure build output path"
04 
05 KERNEL_TOP_PATH="$( cd "$(dirname "$0")" ; pwd -P )"
06 OUTPUT="$KERNEL_TOP_PATH/out"
07 echo "$OUTPUT"
08 
09 KERNEL=kernel7
10 BUILD_LOG="$KERNEL_TOP_PATH/rpi_build_log.txt"
11 
12 echo "move kernel source"
13 cd linux
14 
15 echo "make defconfig"
16 make O=$OUTPUT bcm2709_defconfig
17 
18 echo "kernel build"
19 make O=$OUTPUT zImage modules dtbs -j4 2>&1 | tee $BUILD_LOG
```

<br>빌드 스크립트의 내용을 살펴보기 위해 앞에 번호를 붙였다. <br>

```java
01 #!/bin/bash
```
'#!'는 스크립트를 실행할 쉘을 지정하는 선언문이다. 즉,  사용 하려는 명령어 해석기가 bash Shell임을 알려주는 것이다.  <br>
 
```java
05 KERNEL_TOP_PATH="$( cd "$(dirname "$0")" ; pwd -P )"
```
KERNEL_TOP_PATH에 현재 작업 디렉토리를 저장하는 라인이다. 한 줄에 여러 개의 명령어가 합쳐져 있는데, cd "$(dirname "$0")" 와 pwd -P로 나누어 볼 수 있다. <br>
먼저, pwd \- P 는 현재 쉘의 절대 경로를 받아오는 명령어이다.  다만, 쉘의 위치에 따라 결과가 달라지기 때문에 앞에 추가적인 명령이 붙는다.
\$0는 명령어의 첫번째 인자를 의미한다. 여기서는 home/pi/rpi_kernel_src/build_rpi_kernel.sh 이다. dirname 은 문자열에서 디렉토리만 출력하는 명령어이다. dirname "\$0"을 한 결과는 home/pi/rpi_kernel_src이 된다. <br>
따라서 현재 디렉토리를 이동 후 절대 경로를 받아왔으므로 KERNEL_TOP_PATH에는 현재 작업 디렉토리가 저장된다.<br><br>
 **Tip**:큰따옴표 안에 넣은 값은 변수가 실제 값으로 치환된 후 출력되고 작은 따옴표로 감싸진 문자열은 변화없이 그대로 출력된다. 세미콜론은 하나의 명령이 끝날 때 뒤에 붙여서 한 명령이 끝났음을 나타낸다.<br>
 
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
```java
#!/bin/bash

KERNEL_TOP_PATH="$( cd "$(dirname "$0")" ; pwd -P )"
OUTPUT="$KERNEL_TOP_PATH/out"
echo "$OUTPUT"

cd linux

make O=$OUTPUT modules_install
cp $OUTPUT/arch/arm/boot/dts/*.dtb /boot/
cp $OUTPUT/arch/arm/boot/dts/overlays/*.dtb* /boot/overlays/
cp $OUTPUT/arch/arm/boot/dts/overlays/README /boot/overlays/
cp $OUTPUT/arch/arm/boot/zImage /boot/kernel7.img
```
<br>다음 명령어로 셸 스크립트를 실행한다.
```java
root@raspberrypi:/home/pi/rpi_kernel_src# ./install_rpi_kernel.sh
```
커널 빌드 시 에러가 발생했다면 반드시 수정하고 설치해야 한다. 그렇지 않으면 제대로 설치되지 않는다.

### 전처리 코드 생성
리눅스 커널 소스를 분석하다 보면 수 많은 매크로를 만난다. 이 매크로가 소스 분석의 걸림돌로 작용한다. 전처리 코드는 이러한 매크로를 풀어서 표현한다. 따라서 전처리 코드를 생성해서 리눅스 커널 코드를 분석하면 분석에 도움을 받을 수 있다.

셸 스크립트 build_preprocess_rpi_kernel.sh을 작성한다.
```java
#!/bin/bash

echo "configure build output path"

KERNEL_TOP_PATH="$( cd "$(dirname "$0")" ; pwd -P )"
OUTPUT="$KERNEL_TOP_PATH/out"
echo "$OUTPUT"

KERNEL=kernel7
BUILD_LOG="$KERNEL_TOP_PATH/rpi_preproccess_build_log.txt"

PREPROCESS_FILE=$1
echo "build preprocessed file: $PREPROCESS_FILE"

echo "move kernel source"
cd linux

echo "make deconfig"
make O=$OUTPUT bcm2709_defconfig

echo "kernel build"
make $PREPROCESS_FILE O=$OUTPUT zImage modules dtbs -j4 2>$1 | tee $BUILD_LOG
``` 
<br>파일을 저장한 후에는 chmod 명령어를 입력해 파일에 실행 권한을 부여
```java
root@raspberrypi:/home/pi/rpi_kernel_src# chmod +x build_rpi_kernel.sh
```

linux/kernel/sched/core.c 파일을 전처리 코드로 추출하려면 다음 현식으로 셸 스크립트를실행하면 된다
```java
./build_preprocess_rpi_kernel.sh /kernel/shced/core.i
```
소스 코드의 디렉토리를 잘 못 입력하면 에러 메시지와 함께 빌드가 종료된다. linux 폴더를 기준으로 한 소스 코드의 위치를 입력하면 된다

<br>*reference
https://www.raspberrpi.org/documentation/linux/kernel/building.md
디버깅을 통해 배우는 리눅스 커널의 구조와 원리
http://egloos.zum.com/rousalome/v/10011640
