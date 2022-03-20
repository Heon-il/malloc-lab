# Malloc Lab

## 사전 필요지식
 - **Register와 Memory의 소통방식에 대한 간단한 이해**(Endianness까지는 몰라도 됌)
 - **메모리 정렬** (Data structure alignment라고도 함)
 - **Data Segment**
 - **무엇이든 trade off라는 관점!** (Heap의 할당은 메모리 할당에 대한 정책적인 부분이니까)


## 과제 목적
**힙 할당 `정책`을 이해**
(힙 할당 정책을 이해하는 것이기 때문에 Heap영역을 만들 때 init에서 MAX_HEAP으로 Heap의 전체 크기를 받은 후 힙이 기본적으로 어떤 정책으로 관리되고 운영되는지에 대한 기본을 이해)

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

그런데 실제로 데이터를 읽고 저장을 한다거나 할 때 ISA에서는 load와 store에 대한 기능을 제공할 것이다. Core는 레지스터로 부터 받은 byte를 통해 Instruction을 수행할 것이고, 이 레지스터에는 opcode와 operand가 함께 있기 때문에 명령에 전체 주소를 포함하고 있지 못하다.

즉, 1RXX XXXX XXXX XXXX .... (32bit)
이런 연산이 CPU에 들어왔다고 하고 
앞 1R이 1번 Register에 저장하라는 명령이라고 하고 뒷 부분이 주소가 될 것이고 이 주소는 모든 주소를 포함하지 못하게 된다.

2bit가 부족하고 곱하기 4를 해줘야 모든 주소를 포함할 수 있게 된다. (모두 표현이 아니라 4를 곱해야 포함할 수 있다.)

이러한 부분 때문에 4의 배수 주소만 접근 가능하도록 만드는 이유도 있다. 


## Heap
C에서 Data Segment는 Data, Code, Stack, Heap으로 나뉘어 진다. 여기서 Heap은 Runtime시에 크기를 정하고 사용할 수 있도록 구분된 영역이다.
Heap이라고 해서 데이터 영역이 특별한 것이 아니고, 0,1 flag를 통해 allocation과 free상태를 저장하고, list를 만들어 힙을 `관리`


### Block
 * Heap을 Block이라는 개념으로 나눠서 관리 
 * Block의 크기는 최소 2word 이상이 되어야 함 (2word 이상의 정렬제한 조건이 되어야 함)
 	- memory alignment의 특성 때문에 기본적으로는 block의 주소는 word의 배수에 맞게 align 되어 있어야 한다.
	- 할당 상태에 대한 정보를 담는 flag, block의 크기 등 다른 오버헤드를 담아야 하기 때문에 1 word를 1 block으로 한다면 공간이 부족해진다. ==> 최소 2 block
		- 우리가 User입장에서 Data를 읽어 들일 때는 Header의 정보는 전혀 필요가 없다. 그렇기 때문에 최소 8byte align이 되어야 실제 데이터가 담긴 부분을 시작으로 Data를 읽어 들일 수가 있다. 아래 빨간색 부터 데이터를 read하는 것
		![](https://user-images.githubusercontent.com/42313359/145668427-876a7b17-2ab8-407f-b42e-119db819ee40.png)


	- malloc은 2 word를 기준으로 align한다. 그렇기 때문에 32bit의 경우 8의 배수 주소를 return하고 64bit는 16의 배수를 return한다. (GUN에서)
	- 2 word 이상의 정렬 제한을 원한다면 `aligned_alloc` 이나 `posix_memalign`을 사용할 수 있다.(GNU에서)

> **결국, 이 block에 어떤 정보를 담고 block들을 어떻게 관리하는지, 어떤 정책으로 관리하는지가 Heap 메모리 이용도, 할당 속도를 높이는지에 대한 것이 된다.**

## implicit
#### Block
 * Header
 	- 크기는 1word 
	- 이 1word에 할당상태를 나타내는 flag와 block의 크기를 담으면서 8byte align을 맞추기 위해 하위 3bit를 나누는 인코딩, 디코딩 방식을 이용한다.(32bit에서)


가장 기본적으로 Header하나만 있는 block에서 연결리스트를 만들어서 Heap할당을 관리한다고 하면 **단편화 문제**가 발생하고, **반환 시간을 상수 시간**으로 줄이지 못한다.

#### 기본적인 해결 방법
* 단편화 문제는 `coalecing(연결)`로 해결
	-  언제 연결하는지가 이슈가 될 수 있는데 여기에서는 `즉시 연결`을 가정. (실제 빠른 할당기는 `지연 연결`을 선택)
* 반환 시간을 상수시간으로 줄이기 : `경계태그`를 이용해 해결
	- 경계태그는 Header와 같은 정보가 담긴 1word를 block의 끝에 Footer로서 배치하면서 이뤄질 수 있음
	- 1word의 공간이 추가적으로 사용된다는 단점이 있지만, RAM의 크기가 충분히 커진만큼 큰 문제가 되지 않고, 반환시간을 상수시간으로 줄일 수 있다는 점이 trade off에서 더 우위를 가진다.




#### 가용 블록 검색 및 배치
가용 블록을 검색 방식은 여러가지가 있을 수 있지만, 가장 기본적인 방법으로는 first fit, next fit, best fit이 있다.

* **First fit**
	- 리스트를 처음 부터 검색해서 크기가 맞는 첫 번째 block을 선택
* **Next fit**
	- 검색이 종료된 지점을 기억한다음 그 종료 지점으로 부터 First Fit처럼 하나씩 검색
* **Best fit**
	- 모든 block을 검색하고 가능한 block중 크기가 가장 작은 block을 배치


Best fit은 메모리 이용률을 높일 수는 있지만, 검색 시간이 오래 걸린다. First fit은 평균 검색시간이 오래 걸린다는 단점이 존재하지만, 큰 block을 뒤로 보낸다는 장점이 있다. Next fit은 빠른 검색시간을 가질 수 있다. 하지만, 연구결과에 의하면 최악의 메모리 이용도를 갖는다고 한다.

> **만약 block을 찾지 못한다면?**
여러가지 방법으로 할 수 있지만, 여기서는 Heap영역을 늘리고 마지막 block과 연결하는 형식으로 구현됌

#### 블록 할당
블록 할당을 할 때에는
* 정렬조건을 만족
* block이 요청된 크기보다 컸다면 크기 조절 해주기
* Header와 Footer 정보 담기

#### implicit 구현을 통해 느껴야 한다고 생각된 부분
* Heap 영역에서 가장 쉬운 방법론으로 메모리를 어떻게 할당하고 관리할 수 있는지
* 어떤 정책을 하는지에 따라 각 부분에 trade off가 생긴다는 관점
* 현대는 메모리가 커진 만큼 block에 추가적인 정보를 더 담으면서 더 좋은 정책을 펼칠 수 있겠다

## explicit
- explicit은 block에 `predecessor`와 `successor`를 추가하여 Free block들을 list로 관리한다.
- implicit에서 free block들만 탐색하면 되기 때문에 검색시간이 전체 block의 수에서 free block의 수로 줄어든다.
- free block list를 관리하는 방법에는 여러가지가 있을 수 있는데 가장 간단한 방법으로는 LIFO를 이용하는 것이고, 주소 순 또는 block의 크기 순으로 정렬하여 list를 관리해줄 수 있다. 


## segregated list
 - size class를 만들어 여러 개의 free block을 관리한다. 쉽게 말해서 explicit을 여러 size class로 구분하여 free block을 관리하는 것이다. 이때, size들은 2의 거듭 제곱의 범위로 나눠주는 것이 일반적인 방법이다.

## 최적화 이슈
explicit과 segregated list 등의 개념까지 이해하는 것은 간단한 일이다. 하지만, 이를 최적화 하고 메모리 이용도와 속도를 높이는 것은 많은 디자인 스페이스를 고려해야하기 때문에 연구 주제이자 도전적인 과제로 느껴진다. malloc lab 프로젝트에서 98점이라는 높은 점수를 받은 코드를 보았는데 이 코드를 기반으로 내가 이해한 최적화에 대해 적어보고자 한다.([링크](https://github.com/mightydeveloper/Malloc-Lab/blob/master/mm.c), 가장 기본적으로는 segregated list를 이용하였고 최적화 하여 점수를 높임)


- free block list를 크기 순으로 정렬 + first fit + segregated list
	- first fit의 장점은 큰 블록들을 뒤에 남겨두는 경향성이 있다는 것이다.
	- 이러한 경향성과 block을 크기 순으로 정렬하는 것은 first fit의 장점을 최대한 살릴 수 있고, 분리된 size 때문에 검색 속도를 또한 더 높일 수 있다.(정렬이 되어 있기 때문에 best fit의 특성까지 가지게 된다.)
- place 함수에서는 test case에 맞는 적절한 size를 100으로 보고 큰 블록들을 뒤에 남겨두기 위한 경향성을 유지하려고 한다.
- realloc
	 - heap extension을 최소화
	 	- next block이 free인지 확인하고 공간이 충분하다면 expanding
		- 공간이 충분하지 않다면 heap extension을 해서 확장을 하고 옮긴다.(나의 추측 : `한 번 realloc한 block은 다시 realloc할 가능성이 높다고 가정하고 buffer를 주는 듯 하다.` C++의 vector의 capacity가 어떻게 변하는지의 양상을 보면 일정 범위 이상에서 부터 1.5 정도의 배수 형태를 유지하기 때문에 이렇게 추측 할 수 있을 것 같다.)
	- 큰 block을 뒤로 남겨두는 경향성 때문에 realloc 에서도 그 성질을 이용하면 단편화를 줄일 수 있는 것으로 보인다.

	- realloc bit를 둔다.
		-	buffer 사이즈를 기준으로 비교해서 next block에 realloc 비트를 준다. realloc된 block을 위한 임시 공간을 확보해 heap extension 과정을 최소화하려고 한다.


## Garbage Collector
### 기초
- Directed Graph로 메모리를 바라본다.
- 노드는 루트 노드와 힙 노드로 구분된다.
- 루트 노드는 힙에 속하지 않고 힙노드로 방향을 가지는 메모리에 대응된다 
ex) stack의 지역 변수, 매개 변수, 전역 변수 등
- 루트 노드에서 도달할 수 없는 노드가 존재하면 그 노드는 도달할 수 없다고 이야기 한다. 이렇게 도달 불가능한 노드는 쓰레기(garbage)가 된다.
![Untitled](https://user-images.githubusercontent.com/42313359/146148864-a404c21b-c3e6-4201-9727-6c13964401da.png)
- 도달 불가능한 노드들을 free 시켜서 반환 받고 이를 가용 리스트에 추가한다.
- 자바와 같은 가비지 컬렉터는 포인터를 어떻게 생성하고 사용하는지에 대해 깊게 제어하고 그래프의 정확한 표현을 유지하기 때문에 모든 Garbage를 free시킬 수 있다.
- C와 C++를 위한 일반적인 Garbage Collector 같은 경우는 그래프의 정확한 표현을 유지할 수 없기 때문에 Garbage Collector의 기능을 완벽하게 제공하기는 힘들고 이와 같은 Collector를 보수적인 가비지 컬렉터(conservative garbage collector)라고 한다.
- 컬렉터들은 새로운 요청에 의해 자신의 기능을 제공(garbage를 찾아서 회수 하는 등)하거나 별도의 스레드를 생성해 병렬로 그래프를 관리하며 garbage를 회수한다.
- 만약 C와 같은 프로그램을 위한 보수적인 가비지 컬렉터를 malloc과 연동할 수 있는지 모델링 해본다면 아래와 같은 그림으로 표현될 수 있다.
![Untitled (1)](https://user-images.githubusercontent.com/42313359/146149066-302727df-eb8c-4faf-83a6-79f16e3687fb.png)

### Mark & Sweep 가비지 컬렉터

- Garbage Collection을 수행하기 위한 하나의 알고리즘
: Mark 단계와 Sweep 단계로 나누어진다.
- **Mark** : 도달 가능한 Heap Node들을 찾고 표시(mark)한다.
- **Sweep** : Mark단계에서 표시되지 않은 Garbage들을 회수 한다.

> C언어로 의사코드를 통해 Mark & Sweep 가비지 컬렉터 알고리즘이 어떻게 동작하는지 이해를 해보도록 하자!

```C
typedef void* ptr; // void 포인터 타입을 ptr로 정의

/* 함수 설명 */
ptr isPtr(ptr p); // p가 할당된 블록 내의 어떤 워드를 가리키면, 해당 블록의 시작을 가리키는 포인터를 리턴, 그 외는 NULL을 리턴
int blockMarked(ptr b); // 블록 b가 이미 mark되어 있다면 true 리턴
int blockAllocated(ptr b); // 블록 b가 할당되어 있다면 true 리턴
void markBlock(ptr b); // 블록 b를 mark
int length(ptr b); // 블록 b의 길이를 워드로 리턴(헤더 제외)
void unmarkBlock(ptr b); // 블록 b의 상태를 marked에서 unmarked로 변경
ptr nextBlock(ptr b); // 힙 내 블록 b의 다음 블록을 리턴
```

#### mark function
```C
void mark(ptr p){
	if ((b== isPtr(p)) == NULL)
		return;
	if (blockMarked(b))
		return;
	markBlock(b);
	len = length(b);
	for (i =0; i<len; i++)
		mark(b[i]);
	return;	
}
```
#### sweep function
``` C
void sweep(ptr b, ptr end){
	while(b<end){
		if(blockMarked(b))
			unmarkBlock(b);
		else if(blockAllocated(b))
			free(b);
		b = nextBlock(b);
	}
	return;
}
```

### 추가
[링크](https://medium.com/@joongwon/jvm-garbage-collection-algorithms-3869b7b0aa6f)에서 Garbage Collection Algorithm을 쉽게 잘 설명해주신다. 아직 개발 언어를 정하지는 못했지만 나중에 내가 java 개발자가 된다면 [이 글](https://d2.naver.com/helloworld/1329)도 읽고 이해할 수 있으면 좋겠다. 



## 마무리
- malloc이 공간을 어떻게 할당해줄 수 있는지 낮은 수준에서 이해해볼 수 있어서 좋았다.
- segregated list까지 이해를 하면서 trade off, 최적화를 통해 메모리 이용률, 속도를 높일 수 있는 방법들을 알아 볼 수 있었다.
- 할당기를 구현을 할 때에는 캐스팅, 메모리 alignment 등 신경써줘야 할 것이 많기 때문에 c언어와 더 가까워 질 수 있는 시간이었다.(처음부터 스스로 구현을 해야 했더라면 상당히 까다로운 작업이었을 것)
- Garbage Collector에 대한 기본적인 이해를 해볼 수 있었다.
- 낮은 수준에서 Garbage Collector를 구현해보고 싶었지만(다른 사람이 만든 코드를 보고 이해해보고 싶었지만,) 실패.. 








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
