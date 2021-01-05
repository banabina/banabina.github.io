---
title: "[linux] IRQ 스레드란"
tags:
- linux
categories:
- linux
toc: true
toc_sticky: true
---

IRQ 스레드는 인터럽트 후반부 처리 기법 중 하나 입니다. IRQ 스레드를 알기 전 인터럽트 후반부 처리가 무엇인지 알아봅니다.

## 1\. 인터럽트 후반부 기법이란?

인터럽트 후반부 기법은 인터럽트가 발생했을 때 바로 처리해야할 코드를 실행하고 그 후에 인터럽트를 처리하는 기법을 의미합니다. 인터럽트 후반부 기법을 사용하는 이유는 인터럽트는 **빠르게** 처리되야 하기 때문입니다. 실시간으로 빠르게 처리되야 할 코드는 기존의 인터럽트 핸들러에서 처리하고 급하게 처리하지 않아도 되는 코드는 인터럽트 후반부에서 처리하게 합니다. 인터럽트 후반부 기법을 사용하지 않아 인터럽트 핸들러의 실행 시간이 길어진다면 시스템의 반응 속도가 매우 느려질 수 있습니다. 더 나아가 커널에서는 시간이 오래 걸리는 함수를 호출할 경우 커널 패닉을 유발하자는 제약을 둡니다.

### Top Half / Bottom Half

-   Top Half : 인터럽트가 발생한 후 빨리 처리해야 하는 일(인터럽트 핸들러가 하는 일)
-   Bottom Half : 조금 있다 처리해도 되는 일(인터럽트 처리를 프로세스 레벨에서 수행하는 방식)

### 인터럽트 후반부 처리 기법의 종류

인터럽트 후반부 처리 기법은 IRQ 스레드 외에도 존재합니다. 각 기법은 인터럽트 후반부를 처리하는 방식이 조금씩 다릅니다.

-   IRQ 스레드
-   Soft IRQ
-   태스크릿
-   워크큐

## 2\. IRQ 스레드란?

IRQ는 Interrupt Request의 약자로 HW에 대한 인터럽트를 처리한다는 의미입니다. 스레드는 커널 공간에서만 수행되는 커널 스레드를 의미합니다. IRQ와 스레드가 합쳐진 IRQ 스레드는 인터럽트 후반부 처리를 위한 인터럽트 처리 전용 프로세스입니다.

### IRQ 스레드 확인

터미널에 ps -ely 명령어를 입력하면 다음과 같은 결과를 확인할 수 있습니다.

```
S   UID   PID  PPID  C PRI  NI   RSS    SZ WCHAN  TTY          TIME CMD
...
S     0    81     2  0   9   -     0     0 -      ?        00:00:00 irq/86-mmc1
```

가장 오른쪽에 보이는 irq/86-mmc1이 IRQ 스레드입니다. IRQ 스레드는 "irq/인터럽트 번호-인터럽트 이름"과 같은 규칙에 따라 프로세스의 이름이 정해집니다. 다음과 같은 형식을 가진 프로세스가 보이면 IRQ 스레드라 생각하면 됩니다.

## 3\. IRQ 스레드 생성

![interrupt_thread]({{ site.url }}/assets/images/post/linux/interrupt_thread.png){: .align-center .open-new}

### request\_threaded\_irq() 함수

-   1단계 : 인터럽트 디스크립터 설정
    -   request\_threaded\_irq() 함수에 전달된 인자를 인터럽트 디스크립터 필드에 할당
-   2단계 : IRQ 스레드 생성
    -   irq\_handler\_t의 thread\_fn 인자에 IRQ 스레드 처리 함수 주소를 지정하면 IRQ 스레드를 생성

### \_\_setup\_irq() 함수

-   IRQ 스레드 처리 함수가 등록됐는지 점검
-   IRQ 스레드가 등록됐으면 setup\_irq\_thread() 함수를 호출해 IRQ 스레드를 생성

### setup\_irq\_thread() 함수

[https://github.com/raspberrypi/linux/blob/rpi-4.19.y/kernel/irq/manage.c](https://github.com/raspberrypi/linux/blob/rpi-4.19.y/kernel/irq/manage.c)

```
static int
setup_irq_thread(struct irqaction *new, unsigned int irq, bool secondary)
{
    struct task_struct *t;
    struct sched_param param = {
        .sched_priority = MAX_USER_RT_PRIO/2,
    };

    if (!secondary) {
        t = kthread_create(irq_thread, new, "irq/%d-%s", irq,
                   new->name);
    } else {
        t = kthread_create(irq_thread, new, "irq/%d-s-%s", irq,
                   new->name);
        param.sched_priority -= 1;
    }
```

setup\_irq\_thread() 함수에서는 kthread\_create() 함수를 호출해서 커널 스레드를 생성합니다. 스레드 핸들러 함수로 irq\_thread 함수를 전달해줍니다.

### IRQ 스레드 처리 함수 vs IRQ 스레드 핸들러 함수

-   IRQ 스레드 처리 함수 : 인터럽트별로 지정한 IRQ 스레드별로 후반부를 처리하는 함수를 의미
-   IRQ 스레드 핸들러 함수 : irq\_thread() 함수를 뜻하며 인터럽트별로 지정된 IRQ 스레드 처리 함수를 호출하는 역할을 수행

### irq\_thraed() 함수

![irq_thread]({{ site.url }}/assets/images/post/linux/irq_thread.png){: .align-center .open-new}


IRQ 스레드 핸들러 함수에 irqaction 구조체 타입의 매개변수로 전달합니다. 이는 IRQ 스레드 처리 함수에서 인터럽트 후반부 처리를 위해 인터럽트의 세부 속성 정보가 담긴 자료구조가 필요하기 때문입니다.

## 4\. IRQ 스레드는 언제 실행할까

IRQ 스레드는 생성된 후 다음과 같은 동작을 계속해서 반복 수행합니다.

1.  인터럽트 핸들러에서 IRQ\_WAKE\_THREAD 반환
2.  IRQ 스레드를 깨움
3.  IRQ 스레드 핸들러인 irq\_thread() 함수를 실행
4.  irq\_thread() 함수에서 IRQ 스레드 처리 함수 호출

![interrupt_thread]({{ site.url }}/assets/images/post/linux/irq_thread2.png){: .align-center .open-new}

-   IRQ 스레드 실행의 출발점은 인터럽트 핸들러에서 IRQ\_WAKE\_THREAD를 반환하는 시점

**Reference**  
디버깅을 통해 배우는 리눅스 커널의 구조와 원리
