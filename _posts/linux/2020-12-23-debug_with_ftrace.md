---
title: ftrace로 디버깅하기
toc: true
toc_sticky: true
categories:
- linux
tags:
- linux
excerpt: |-
  ftrace란?
  리눅스 커널에서 제공하는 강력한 트레이서이다.  ftrace를 사용하면 커널 내부에서 어떤 일이 발생하는 지 파악하는데 도움을 준다. 커널에서 호출되는 이벤트나 함수의 흐름을 쫒아갈 때 사용할 수 있는 디버깅툴이라 생각하면 된다.
---

**ftrace란?**<br>
리눅스 커널에서 제공하는 강력한 트레이서이다.  ftrace를 사용하면 커널 내부에서 어떤 일이 발생하는 지 파악하는데 도움을 준다. 커널에서 호출되는 이벤트나 함수의 흐름을 쫒아갈 때 사용할 수 있는 디버깅툴이라 생각하면 된다.



##  1. ftrace 일단 사용해보기
툴을 익히는 제일 좋은 방법은 일단 사용해보는 것이라 생각한다. 사용 환경은 라즈베리파이 3b+ 모델 라즈비안 4.19y에서 진행하였다. <br><br>
불필요한 권한 설정을 피하기 위해 root 권한을 획득하였다.
```bash
$ sudo su
```
work 디렉토리에 rpi_debug 폴더를 생성하였다. 해당 폴더에 ftrace 설정을 위한 셸 스크립트와 로그 파일을 생성할 것이다.
```bash
root@raspberrypi:/home/pi/work# mkdir rpi_debug
```

### ftrace 설정

먼저, ftrace 설정을 위한 셸 스크립트를 생성한다. 
```bash
root@raspberrypi:/home/pi/work/rpi_debug# vi irq_setup_ftrace.sh
```
<br>ftrace 설정에 따라 생성되는 로그 파일의 내용이 달라진다. 설정을 통해서 필요한 내용을 출력하고 이를 활용해 디버깅하도록 하자.<br><br>
irq_setup_ftrace.sh 파일에 아래의 내용을 입력한다.
<script src="https://gist.github.com/banabina/da65246bc2c290846f882d94d6f2f284.js"></script>
'echo 0 > [파일명]' 명령어를 입력하면 해당 파일에 0이 저장되고 'echo 1 > [파일명]'을 입력하면 해당 파일에 1이 저장된다. 
```
 3 echo 0 > /sys/kernel/debug/tracing/tracing_on
...
21 echo 1 > /sys/kernel/debug/tracing/tracing_on
```
/sys/kernel/debug/tracing/tracing_on 파일에 0을 저장하면 ftrace를 비활성화하고  /sys/kernel/debug/tracing/tracing_on 파일에 1을 저장하면 ftrace를 활성화한다. 시작 시에 ftrace를 비활성화하고 설정을 마친 후 ftrace를 활성화한 것이다.  ftrace를 활성화된 상태에서 설정을 바꾸면오동작하는 경우가 발생할 수  있기 때문에 설정 전 ftrace를 비활성화 시켰다. 코드 중간에 sleep 1이 추가된 이유도 안정적인 실행을 위함이다.<br>
```
14 echo 1 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable
15 echo 1 > /sys/kernel/debug/tracing/events/sched/sched_wakeup/enable

18 echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/enable
19 echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_exit/enable
```
events 폴더의 sched_switch, sched_wakeup, irq_handler_entry, irq_handler_exit의 enable 파일에 1을 저장하였다. 각각의 이벤트를 활성화한 것이다. 활성화 시킨 이벤트는 ftrace 이벤트 포맷에서 다시 만날 수 있다.

셸 스크립트를 실행 시키기 위해 권한을 부여한다.
```bash
chmod +x irq_setup_ftrace.sh
```
셸 스크립트를 실행시킨다. 셸 스크립트를 실행시키면 라즈베리파이에서 백그라운드로 ftrace 로그를 로깅하고 있는 상태가 된다.  
```
./irq_setup_ftrace.sh
```

### ftrace 로그 추출

ftrace의 log가 들어있는 위치는 /sys/kernel/debug/tracing/trace 이다. 로그 파일을 복사해오는 셸 스크립트를 작성해서 ftrace 로그를 추출하자

get_ftrace.sh
<script src="https://gist.github.com/banabina/3683f0d1f44f5e7d323cd53d14558dd6.js"></script>
해당 스크립트도 동일하게 권한을 부여하고 실행하자. ftrace_log.txt 파일이 생성된 걸 확인할 수 있다.  

### ftrace 로그 분석하기

ftrace_log.txt에 생성된 로그 한줄을 가져왔다. 여기서 ftrace의 공통 항목을 살펴보자.
```
 |프로세스 이름| |PID|  |CPU번호|  |컨텍스트 정보|  |타임 스탬프|        |이벤트|
 kworker/u8:2 -  125     [000]         d.h.        1284.875630:   irq_handler_entry: irq=86 name=mmc1
```
위 로그를 해석하면 다음과 같다.
- 프로세스 이름은 kworker/u8:2
- PID는 125
- CPU 번호는 [000]
- 컨텍스트 정보는 d.h.
	-  d : 현재 해당 cpu의 인터럽트를 비활성화
	-  h : 현재 프로세스는 인터럽트 컨텍스트
 - 타임 스탬프는 커널 로그에 출력하는 시간 정보이다
 - 마지막으로 irq_handler_entry는 이벤트 이름이다. 
	 - 설정에서 활성화 했던 이벤트 중 하나이다.
	 -  이벤트에 따라 뒤에 추가되는 로그의 내용이 달리지며 해석하는 방법도 이벤트마다 다르다.


## 2.  ftrace 설정
ftrace를 설정하고 로그를 추출하여 이를 분석해 보았다. ftrace에는 많은 설정 내용이 있다. 설정에 대한 자세한 내용을 살펴보자.<br>
ftrace의 설정 파일은 /sys/kernel/debug/tracing 에서 찾아 볼 수 있다. 
```
ls /sys/kernel/debug/tracing 
```
명령어를 다음과 같이 입력하면 해당 디렉토리에 있는 파일 목록을 볼 수 있다.
![tracing 디렉토리]({{ site.url }}/assets/images/post/linux/ftraceList_sys_kernel_debug_tracing.png){: .align-center}

해당 디렉토리에는 많은 파일이 있지만 자주 사용되는 파일은 옆에 V 체크를 해두었다. O 표시가 되어있는 trace 파일에는 ftrace의 로그가 기록된다.

### tracing_on : 트레이서 활성화 / 비활성화
```
$ echo 0 > /sys/kernel/debug/tracing/tracing_on
$ echo 1 > /sys/kernel/debug/tracing/tracing_on
```
tracing_on은 ftrace를 활성화 하기 위해 설정해야 하는 파일이다. 파일에 0을 저장하면 ftrace를 비활성화, 1을 저장하면 ftrace를 활성화한다. 부팅 시 기본 값은 0이다.

### ftrace 이벤트 설정
가장 중요한 설정이다. ftrace 이벤트는 커널을 구성하는 서브 시스템과 기능별로 세부 동작을 출력하는 기능이다. ftrace 이벤트를 설정해서 원하는 이벤트만 출력하도록 할 수 있다. <br>
아래와 같이 설정하면 ftrace의 이벤트를 모두 비활성화 할 수 있다.
```
$ echo 0 > /sys/kernel/debug/tracing/events/enable
```
 모든 이벤트를 설정하면 버벅거리고 너무 많은 정보가 출력되게 된다. 따라서 ftrace의 이벤트를 비활성화한 후 원하는 이벤트를 활성화 시키자. 활성화 하고자 하는 이벤트는 다음과 같이 해당 파일에 1을 저장하면 된다.
```
$ echo 1 > /sys/kernel/debug/tracing/events/sched/sched_wakeup/enable
$ echo 1 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable
$ echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/enable
$ echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_exit/enable
```
위의 두 줄은 스케줄링의 동작을 기록하는 sched_wakeup과 sched_switch 이벤트를 활성화했다. 아래 두 줄은 인터럽트 핸들링의 시작과 종료 시점을 기록하는 irq_handler_entry와 irq_handler_exit 이벤트를 활성화였다. <br><br>
이벤트의 종류는 많다. 그 중 대표적인 이벤트인 sched와 irq를 살펴보자
- sched : 프로세스가 스케줄링되는 동작과 스케줄링 프로파일링을 트레이싱하는 이벤트
  - sched_switch : 컨텍스트 스위칭 동작
  - sched_wakeup : 프로세스를 깨우는 동작
- irq : 인터럽트와 Soft IRQ를 트레이싱하는 이벤트
  - irq_handler_entry : 인터럽트가 발생한 시각과 인터럽트 번호 및 이름 출력
  - irq_handler_exit : 인터럽트 핸들링 완료
  -  softirq_raise : Soft IRQ 서비스 실행 요청
  -  softirq_entry: Soft IRQ 서비스 실행 시작
  -  softirq_exit : Soft IRQ 서비스 실행 완료

### tracer 설정
ftrace는 nop, function, function_graph와 같은 트레이서를 제공한다.
- nop : 기본 트레이서. ftrace 이벤트만 출력한다
- function : 함수 트레이서. set_ftrace_filter로 지정한 함수를 누가 호출했는 지 출력한다.
- function_graph : 함수 실행 시간과 세부 호출 정보를 그래프 포맷으로 출력

```
$ echo nop > /sys/kernel/debug/tracing/current_tracer
$ echo function > /sys/kernel/debug/tracing/current_tracer
$ echo function_graph > /sys/kernel/debug/tracing/current_tracer
```
다음과 같은 current_tracer 설정에 트레이서의 이름을 저장해주면 된다.

### set_ftrace_filter 설정
위의 tracer 설정의 function 혹은 function_graph으로 설정한 경우 작동하는 파일이다. set_ftrace_filter 파일에 트레이싱하고 싶은 함수를 지정하면 된다.
```
$ echo secondary_start_kernel > /sys/kernel/debug/tracing/set_ftrace_filter
$ echo schedule_ttwu_do_wakeup > /sys/kernel/debug/tracing/set_ftrace_filter
```
리눅스 커널에 존재하는 모든 함수를 필터로 지정할 수는 없다. /sys/kernel/debug/tracing/available_filter_functions 파일에 포함된 함수만 지정할 수 있다. 함수를 지정하지 않은 경우 모든 함수를 트레이싱하게 되어 락업이 상태에 빠지게 된다. available_filter_functions 파일에 없는 함수를 지정하려도 락업 상태가 될 수 있으니 주의하자.

## 3.  ftrace 메시지 분석

ftrace 로그 분석에 대해서도 자세한 내용을 살펴보자. 위에서 사용했던 로그를 다시 가져왔다.
```
 |프로세스 이름| |PID|  |CPU번호|  |컨텍스트 정보|  |타임 스탬프|        |이벤트|
 kworker/u8:2 -  125     [000]         d.h.        1284.875630:   irq_handler_entry: irq=86 name=mmc1
```

### 컨텍스트 정보
 컨텍스트 정보는 4개의 알파벳으로 출력된다. 세부 항목이 활성화 되지 않는 경우 "."을 출력한다.각각의 내용은 다음과 같다.
- 인터럽트 활성화/비활성화 여부 
	- d : 해당 cpu 라인의 인터럽트를 비활성화한 상태
- 선점 스케줄링 설정 여부
	- n : 현재 프로세스가 선점 스케줄링 될 수 있는 상태
- 인터럽트 컨텍스트나 Soft IRQ 컨텍스트 여부
	- h / s : h이면 인터럽트 컨텍스트, s면 Soft IRQ 컨텍스트
- 프로세스의 thread_info 구조체의 preempt_cout 값

### ftrace 이벤트 분석하기

ftrace에서 가장 많이 사용하는 이벤트인 sched_switch와 irq_handler_entry/irq_handler_eixt에 대한 분석 방법이다.

- sched switch

![sched_switch]({{ site.url }}/assets/images/post/linux/sched_switch.png){: .align-center .open-new}

위 로그를 해석하면 다음과 같다.<br> *"swapper/0" 프로세스 에서 "irq/86-mmc1" 프로세스로 스케줄링 하는 동작을 출력한다.*

- irq_handler_entry/irq_handler_eixt

irq_handler_entry는 인터럽트 핸들러를 실행하기 직전에 출력하고 irq_handler_exit은 인터럽트 핸들러 실행을 마무리한 직후에 출력된다.

```
    kworker/u8:2-125   [000] d.h.  1284.878737: irq_handler_entry: irq=86 name=mmc1
    kworker/u8:2-125   [000] d.h.  1284.878747: irq_handler_exit: irq=86 ret=handled
```
첫번째 로그는 인터럽트 발생에 대한 정보이다. 이를 해석하면 다음과 같다.<br>*pid가 125인 kworker/u8:2 프로세스가 실행하는 중에 86번 인터럽트가 발생했다.* <br>
두 번째 로그는 86번 인터럽트 핸들러의 실행을 마무리했다는 정보이다. 타임 스탬프를 토대로 해당 인터럽트 핸들러가  1284.878737초에 시작해서  1284.878747초에 마무리된 걸 알 수 있다.


### ftrace는 커널 내부의 어떤 소스 코드에서 이벤트를 출력했나
ftrace 이벤트가 출력 될 수 있는 건 커낼 내부에 이에 해당하는 소스코드가 있기 때문이다. 예를 들어, ftrace 이벤트 중 하나인 sche_switch를 출력하는 함수는 trace_sched_switch이다. trace_sched_switch 함수가 실행될 때 sched_switch는 프로세스가 스케줄링하는 정보를 출력한다. <br>
리눅스 소스 코드에서 trace_sched_switch를 검색하는 방법은 다음과 같다.
```
root@raspberrypi:/home/pi/rpi_kernel_src/linux# egrep -nr trace_sched_switch *
```
검색 결과로 kernel/sched/core.c의 3512 라인이 출력됐다.
```
kernel/sched/core.c:3512:	trace_sched_switch(preempt, prev, next);
```
해당 파일에 들어가면 sched_switch 이벤트가 __schedule() 함수에서 출력 한다는 사실을 알 수 있다.
```
vi kernel/sched/core.c
```
- vi 편집기에서 라인을 보고 싶으면 :set number를 입력하면 된다
- 원하는 라인을 입력하고 shift + g를 누르면 해당 라인으로 이동한다.

**reference** <br>
디버깅을 통해 배우는 리눅스 커널의 구조와 원리<br>
lwn.net/Articles/365835<br>
lwn.net/Articles/366796<br>
{: .notice--info}
