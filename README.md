#####################################################################
CS:APP Malloc Lab
Handout files for students

Copyright (c) 2002, R. Bryant and D. O'Hallaron, All rights reserved.
May not be used, modified, or copied without permission.

######################################################################

***********
Main Files:
***********

mm.{c,h}	
	Your solution malloc package. mm.c is the file that you
	will be handing in, and is the only file you should modify.

mdriver.c	
	The malloc driver that tests your mm.c file

short{1,2}-bal.rep
	Two tiny tracefiles to help you get started. 

Makefile	
	Builds the driver

**********************************
Other support files for the driver
**********************************

config.h	Configures the malloc lab driver
fsecs.{c,h}	Wrapper function for the different timer packages
clock.{c,h}	Routines for accessing the Pentium and Alpha cycle counters
fcyc.{c,h}	Timer functions based on cycle counters
ftimer.{c,h}	Timer functions based on interval timers and gettimeofday()
memlib.{c,h}	Models the heap and sbrk function

*******************************
Building and running the driver
*******************************
To build the driver, type "make" to the shell.

To run the driver on a tiny test trace:

	unix> mdriver -V -f short1-bal.rep

The -V option prints out helpful tracing and summary information.

To get a list of the driver flags:

	unix> mdriver -h


--------

# Malloc Lab

## 사전 필요지식
 - **Register와 Memory의 소통방식에 대한 간단한 이해**(Endianness까지는 몰라도 됌)
 - **메모리 정렬** (Data structure alignment라고도 함)
 - **Data Segment**
 - **무엇이든 trade off라는 관점!** (Heap의 할당은 메모리 할당에 대한 정책적인 부분이니까)


## Register와 Memory
레지스터는 `word addressable`한 반면 메모리는 `byte addressable` 하다. 

cf) 시대의 변화에 따라, 그리고 회사마다, 쓰이는 영역에 따라 word는 다르게 정의되는 경우가 많은데 이곳에서는 레지스터의 크기로 정의하고 이야기함

Computer Architecture의 ISA 설계에서 2번 째에 해당하는 이야기인 "Defines the storage elements that are accessible"에서는 2가지 Architecture가 있다.(Load-Store Architecture, Memory to Memory Architecture) Memory to Memory Architecture라 할 지라도 Memory hierarchy 때문에 Core는 레지스터와 레지스터는 메모리와 자주 소통하는 것이 좋다.

레지스터는 word addressable, 메모리는 byte addressalbe하다는 특성, 그리고 레지스터와 Core의 통신이 잦은 상황이 선호된다는 사실은 결과적으로 시스템에게 어떠한 요구사항을 만드는데 그것이 Memory(Data Structure) Alignment이다.

(실제로 Memory Alignment가 되어 있지 않아도 접근이 가능하게 설계한 경우도 있고 그렇지 않으면 접근하지 못하도록 만든 경우도 있다.)


## 메모리 정렬
먼저 메모리 정렬을 왜할까? 라는 질문이 생길 수 있는데 결론적으로 이야기하면 속도, 성능을 높이기 위해서이다.

**[-][0] | [0][1][1][-] | [-][-]**   *('|'<-- 주소값 4의 배수 경계점)*

위와 같이 데이터 저장되어 있다고 한다면 데이터를 읽기 위해 2번의 접근이 필요하다.

이러한 이유로 struct를 만들 때나, 개개의 데이터를 선언하고 저장할 때 메모리를 정렬하게 되면 접근 수를 줄일 수 있고, 잘 정렬하게 되면 padding 공간도 더 줄일 수 있다.

단순히 이러한 이유만으로 Memory alignment가 나온 것은 아니다. Load-Store Architecture구조라면 Core는 오직 레지스터와만 통신한다. 

그런데 실제로 데이터를 읽고 저장을 한다거나 할 때 ISA에서는 load와 store에 대한 기능을 제공할 것이다. Core는 레지스터로 부터 받은 byte를 통해 Instruction을 수행할 것이고, 이 레지스터에는 opcode와 operand가 함께 있기 때문에 load와 store에 대해서 명령에 전체 주소를 포함하고 있지 못하다.

즉, 1RXX XXXX XXXX XXXX .... (32bit)
이런 연산이 CPU에 들어왔다고 하고 
앞 1R이 1번 Register에 저장하라는 명령이라고 하고 뒷 부분이 주소가 될 것이고 이 주소는 모든 주소를 포함하지 못하게 된다.

2bit가 부족하고 곱하기 4를 해줘야 모든 주소를 포함할 수 있게 된다. (모두 표현이 아니라 4를 곱해야 포함할 수 있다.)

이러한 부분 때문에 4의 배수 주소만 접근 가능하도록 만드는 이유도 있다. 





