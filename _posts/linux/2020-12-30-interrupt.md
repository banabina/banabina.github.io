---
title: 인터럽트
categories:
- linux
tags:
- linux
toc: true
toc_sticky: true
---

## 1\. 인터럽트란?

-   일반적인 상황에서 갑자기 발생하는 비동기적인 통지나 이벤트
-   하드웨어 관점에서 인터럽트는 하드웨어의 변화를 감지해서 외부 입력으로 전달되는 전기 신호
-   인터럽트가 발생하면 프로세스는 하던 일을 멈추고 인터럽트 벡터와 인터럽트 핸들러를 실행하여 하드웨어의 변화를 처리한다
-   운영체제에서 인터럽트 핸들러는 빨리 실행해야 하는 것
    -   인터럽트가 발생하면 실행 중인 코드가 멈추기 때문

### ARMv7 아키텍처에서의 인터럽트 처리

-   ARMv7 아키텍처에서 인터럽트는 익센션의 한 종류로 처리
-   외부 하드웨어 입력이나 오류 이벤트가 발생하면 익센션 모드로 진입
-   익셉션이 발생했다고 감지하면 익셉션 종류별로 이미 정해 놓은 주소로 브랜치

### 인터럽트 핸들러란?

-   인터럽트가 발생하면 이를 핸들링하기 위한 함수가 호출되는데 이를 인터럽트 핸들러라 함
-   함수 형태로 존재, 커널 내부의 IRQ(Interrupt ReQuest) 서브 시스템을 통해 호출
-   request\_irq()를 통해 인터럽트 핸들러를 등록

## 2\. 인터럽트 컨텍스트

-   현재 실행 중인 프로세스가 현재 인터럽트를 처리 중이라는 것을 의마한다
-   리눅스 커널에서 컨텍스트
    -   프로세스 실행 그 자체를 의미, 현재 실행 중인 프로세스 정보를 담고 있는 레지스터 세트로 표현
    -   레지스터 세트는 cpu\_context\_save 구조체
    -   프로세스 스택의 최상단 주소에 위치한 thread\_info 구조체의 cpu\_context 필드에 저장

### in\_interrupt() 함수

-   현재 실행 중인 코드가 인터럽트 컨텍스트 구간인지 알려주는 함수
-   true를 반환하면 인터럽트 컨텍스트이고 false를 반환하면 프로세스 컨텍스트

사용예시

```
idata = kmalloc(sizeof(*idata), in_interrupt() ? GFP_ATOMIC : GFP_KERNEL);
```

-   인터럽트 컨텍스트인 경우 GFP\_ATOMIC 옵션으로 메모리를 할당하여 휴면하지 않고 메모리를 할당받는다

## 3\. 인터럽트가 발생하면 어떤 코드가 실행될까

인터럽트의 처리방식은 HW 아키텍처에 따라 달라질 수 있다. ARMv7에서 인터럽트는 익셉션의 한 종류로 처리한다. 인터럽트가 발생하면 가장 먼저 인터럽트 벡터인 vector\_irq 레이블이 실행된다.

ARMv7 아키텍처의 익셉션 벡터 테이블

```
오프셋     익셉션 종류
--------------------------------------- 
0x00     Not used
0x04     Undefined Instruction
0x08    Supervisor Call
0x0C    Prefetch Abort
0x10     Data Abort
0x14     Not used
0x18     IRQ interrupt
0x1C     FIQ interrupt
```

-   ARM 프로세서는 익센셥 벡터 베이스 주소에서 0x18을 더한 주소로 HW적으로 PC를 브랜치함

vector\_irq 레이블

-   인터럽트가 발생한 모드(유저 / 커널 모드)에 따라 다음 레이블로 브랜치함
    -   \_\_irq\_svc : 커널 모드 (ARM 프로세서 기준 : Supervisor Mode)
    -   \_\_irq\_usr : 유저 모드 (ARM 프로세서 기준 : User Mode)

\_\_irq\_svc 인터럽트 벡터 코드

-   스택 공간에 실행 중인 프로세스의 레지스터 세트를 푸쉬
-   bcm\_2386\_arm\_irqchip\_handle\_irq() 함수를 호출
    -   handle\_arch\_irq 전역변수에 지정한 함수

### 인터럽트 핸들러의 호출 흐름

프로세스가 실행되는 중 인터럽트가 발생하면 인터럽트 벡터인 \_\_irq\_svc 레이블로 브랜치하고 handle\_irq\_event\_percpu() 함수까지 실행

```
__irq_svc
    → bcm2836_arm_irqchip_handle_irq
        → __handle_domain_irq
            → generic_handle_irq
                → bcm2836_chained_handle_irq
                    → generic_handle_irq
                        → handle_level_irq
                            → handle_irq_event
                                → __handle_irq_event_percpu
```

-   인터럽트에 대한 여러 가지 예외 처리를 수행
-   1.  커널이 인터럽트를 처리하고 이는 도중 인터럽트가 발생했을 때 리턴 처리
-   2.  인터럽트 정보를 인터럽트 디스크립터에 저장
-   3.  가끔 전달되는 쓰레기 인터럽트 값에 대한 예외 처리

## 4\. 인터럽트 핸들러 등록

외부 장치에서 인터럽트가 발생한 후 해당 인터럽트 핸들러가 호출되려면 부팅 과정에서 인터럽트를 인터럽트 핸들러와 함께 등록해야한다.

## request\_irq() 함수

인터럽트 초기화하는 과정에서 호출된다.  
request\_irq() 함수의 선언부  
[https://github.com/raspberrypi/linux/blob/rpi-4.19.y/include/linux/interrupt.h](https://github.com/raspberrypi/linux/blob/rpi-4.19.y/include/linux/interrupt.h)

```
static inline int __must_check
request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
        const char *name, void *dev)
```

-   irq : 인터럽트 번호
-   handler : 인터럽트가 발생하면 호출될 인터럽트 핸들러의 주소
-   flags : 인터럽트의 속성 플래그
-   name : 인터럽트 이름
-   dev : 인터럽트 핸들러에 전달하는 매개변수

## 5\. 인터럽트 디스크립터

커널은 인터럽트 디스크립터인 irq\_desc 구조체로 인터럽트 별 속성 정보를 관리한다.디스크립터는 커널이 특정 드라이버나 메모리 같은 중요 객체를 관리하려고 쓰는 자료구조이다.  
[https://github.com/raspberrypi/linux/blob/rpi-4.19.y/include/linux/irqdesc.h](https://github.com/raspberrypi/linux/blob/rpi-4.19.y/include/linux/irqdesc.h)

```
struct irq_desc {
    struct irq_common_data    irq_common_data;
    struct irq_data        irq_data;
    unsigned int __percpu    *kstat_irqs;
    irq_flow_handler_t    handle_irq;
#ifdef CONFIG_IRQ_PREFLOW_FASTEOI
    irq_preflow_handler_t    preflow_handler;
#endif
    struct irqaction    *action;    /* IRQ action list */
    unsigned int        status_use_accessors;
```

-   irq\_common\_data : 커널에서 처리하는 irq\_chip 관련 함수에 대한 정보를 담고 있다
-   irq\_data : 인터럽트 번호와 해당 하드웨어 핀 번호를 나타낸다
-   kstat\_irqs : 인터럽트가 발생한 횟수가 저장된다
-   action : 인터럽트 속성 중 핵심 정보를 가진 구조체

irqaction 구조체  
[https://github.com/raspberrypi/linux/blob/rpi-4.19.y/include/linux/interrupt.h](https://github.com/raspberrypi/linux/blob/rpi-4.19.y/include/linux/interrupt.h)

```
struct irqaction {
    irq_handler_t        handler;
    void            *dev_id;
    void __percpu        *percpu_dev_id;
    struct irqaction    *next;
    irq_handler_t        thread_fn;
    struct task_struct    *thread;
    struct irqaction    *secondary;
    unsigned int        irq;
    unsigned int        flags;
    unsigned long        thread_flags;
    unsigned long        thread_mask;
    const char        *name;
    struct proc_dir_entry    *dir;
} ____cacheline_internodealigned_in_smp;
```

-   handler : 인터럽트 핸들러의 함수 주소
-   dev\_id : 인터럽트 핸들러에 전달되는 매개변수
-   thread\_fn : 인터럽트를 IRQ 스레드 방식으로 처리할 때 IRQ 스레드 처리 함수의 주소를 저장하는 필드
-   irq : 인터럽트 번호
-   flags : 인터럽트 플래그 설정 필드

**Reference**  
디버깅을 통해 배우는 리눅스 커널의 구조와 원리  
[http://egloos.zum.com/rousalome/v/10018210](http://egloos.zum.com/rousalome/v/10018210)
