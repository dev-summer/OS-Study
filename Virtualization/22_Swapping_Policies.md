# 22 Swapping: Policies

OS의 페이지 교체 전략: 메모리에서 어떤 페이지를 쪼까낼것인가(swap out)?

## 22.1 Cache Management

메인 메모리는 시스템의 페이지 중 일부를 들고 있기 때문에 시스템의 가상 메모리 페이지들의 캐시라고 볼 수 있다. 즉, 페이지 교체 전략의 목표는 cache miss를 최소화하는 것(또는 cache hit를 최대화 하는 것)이다.
**AMAT** = TM + (PMISS + TD)
- AMAT(Average Memory Access Time)
- TM: cost of accessing memory
- TD: cost of accessing disk
- PMISS: probability of cache miss

현대의 시스템에서는 disk access의 비용이 크므로, cache miss를 최소화해야 한다.

## 22.2 The Optimal Replacement Policy

최적 페이지 교체 전략: 가장 먼 미래에 접근할 페이지를 교체한다(내쫓는다).
- cache miss가 가장 적게 발생한다.
- 구현하기가 어렵다.
- 실용적이지는 않지만, 어떤 페이지 교체 전략을 시뮬레이션 할 때, 효율성을 측정하기 위한 기준으로 사용할 수 있다.

1. 처음으로 페이지에 접근하는 경우, 무조건 cache miss가 뜬다 (최초에 캐시가 비어 있는 상태로 시작하기 때문).
    - 이 때 발생하는 cache miss를 cold-start miss 또는 compulsory miss라고도 한다.
    cf) capacity miss: 캐시가 꽉 차서 새로운 페이지를 가져오려면 기존 페이지 중 하나를 내쫓아야 할 때
    conflict miss: 하드웨어에서 발생한다. set-associativity 때문에 하드웨어 캐시에 페이지가 위치할 수 있는 곳에 제약이 있기 때문에 발생하게 된다. OS 페이지 캐시의 경우 캐시가 항상 fully-associative(페이지가 메모리 상의 어디에 위치해야 하는지에 대한 제약이 없음)하기 때문에, conflict miss가 발생하지 않는다.
2. 캐시가 꽉 차게 되면 캐시에 있는 페이지 중 가장 나중에 재접근하는 페이지를 내쫓고, 새로 접근해야 할 페이지를 그 자리에 넣는다.

## 22.3 FIFO (First-in First-out)

Belady's Anomaly (벨래디의 변이)
- FIFO 전략에서 일반적으로 캐시의 사이즈가 커지면 cache hit rate가 올라가지만,
캐시의 사이즈가 커졌는데 cache hit rate가 오히려 감소하는 경우.
- 예시
    - memory reference stream: 1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5
    - cache size = 3인 경우보다 cache size = 4일 때의 cache hit rate가 더 낮다.
- LRU 전략 등에서는 이 문제가 발생하지 않는다.
    - LRU는 스택 속성을 가지고 있기 때문이다.
    - 스택 속성을 가진 알고리즘들은 cache size = n + 1인 캐시는 자연스럽게 cache size = n인 캐시를 전부 포함하기 때문이다. 그렇기 때문에 cache size가 커지면 cache hit rate이 동일하거나 더 높아진다.

## 22.4 Random

랜덤임

## 22.5 LRU (Least Recently Used)

FIFO나 무작위 알고리즘은 바로 다음에 재참조할 페이지를 쫓아낼 수도 있다는 문제가 있다.
지역성의 원칙(principle of locality)에 기반한 정책이다.
- 지역성의 원칙: 프포그램은 특정한 코드 시퀀스나 데이터 구조에 꽤 자주 접근하는 경향이 있다.
- 이를 활용해서 어떤 페이지가 중요한 지 알아낼 수 있으며, LFU(Least Frequently Used)와 LRU(Least Recently Used)가 이 원칙에 기반해서 만들어졌다.

프로그램이 보여주는 지역성에는 2가지 종류가 있다.
1. 공간 지역성 (spatial locality)
    - 페이지 P에 접근하면, 페이지 P의 주변 페이지(P -1 또는 P + 1)에도 접근할 가능성이 높다.
2. 시간 지역성 (temporal locality)
    - 가까운 과거에 접근했던 페이지는 가까운 미래에 다시 접근할 가능성이 높다.

## 22.6 Workload Examples

페이지가 총 100개일 때, 캐시 사이즈가 1 ~ 100까지 변한다면 캐시 사이즈에 따라 각 정책의 효율성(hit rate)이 어떻게 달라질까?
- 지역성이 없는 경우
    - FIFO, Random, LRU의 성능이 모두 같다.
    - hit rate = cache size
    ![](https://i.imgur.com/bq2Qhjr.png)
- 80-20: 참조의 80%가 전체 페이지 중 20%에서 발생하는 경우
    - hot page: 많이 참조되는 20%의 페이지
    - cold page: 많이 참조되지 않는 80%의 페이지
    - 성능: 최적 > LRU > FIFO = Random
    ![](https://i.imgur.com/nEMV0QR.png)

- Looping Sequential: 페이지에 순서대로 접근 & 순환하는 경우 ( 1 2 3 1 2 ... )
    - FIFO나 LRU의 경우 가장 오래된 페이지를 내쫓게 된다.
    - 하지만 오래된 페이지일 수록 캐시에 있는 페이지보다 더 가까운 미래에 접근할 가능성이 높다.
    - cache size = workload - 1인 경우 hit rate = 0%가 된다. (cache size = 49, workload = 50)
    - 이 경우, Random의 성능이 더 좋다.
    ![](https://i.imgur.com/tCVK6i7.png)


## 22.7 Implementing Historical Algorithms

참조 history에 기반한 알고리즘을 구현하기 위해서는 모든 페이지 참조를 기록하고, 가장 많이/적게 참조된 페이지를 트래킹해야 한다. 이 과정이 성능을 저하시킬 수 있기 때문에, 이 과정을 개선하기 위한 다음과 같은 방법이 있다.
- 하드웨어 지원
    - 모든 페이지 접근 마다 기계가 메모리의 time field(프로세스별 페이지 테이블 또는 메모리 상의 별도의 배열)를 업데이트 한다. 어떤 페이지에 접근하면 하드웨어에 의해 time field가 현재 시각(접근한 시각)으로 설정된다. 페이지를 교체할 때 OS는 시스템의 모든 time field를 스캔해서 가장 오래 전에 접근한 페이지를 찾는다.
    - 시스템의 페이지 숫자가 증가할수록 가장 LRU한 페이지를 찾는 비용이 비싸진다.
    - 정말로 가장 LRU한 페이지를 찾아서 교체해야 할까? 근사한 페이지를 찾아서 교체해도 될까?

## 22.8 Approximating LRU

하드웨어의 **use bit**(또는 **reference bit**)을 사용해서 LRU의 근사치를 구할 수 있다.
- 시스템의 페이지별로 1비트짜리 use bit이 존재하고, use bit은 메모리 어딘가에 존재한다 (프로세스의 페이지 테이블이나 어딘가에 있는 배열 등).
- 어떤 페이지가 참조되면(읽기 또는 쓰기 발생), 하드웨어가 use bit를 1로 설정한다.
- 하드웨어는 use bit를 절대 0으로 설정하지 않는다. use bit를 0으로 설정하는 것은 OS의 역할이다.

OS가 use bit를 사용해서 LRU의 근사값을 구하는 방법
- **clock algorithm**
    - 시스템의 모든 페이지가 원형 리스트에 정렬되어 있다고 정의한다.
    - 시계 바늘은 원형 리스트 내의 특정 페이지를 가리키면서 시작한다 (어떤 페이지인지는 중요하지 않다).
    - 페이지 교체가 일어나야 하는 경우, OS는 시계 바늘이 현재 가리키고 있는 페이지 P의 use bit를 확인한다.
        - 만약 use bit가 1이라면 페이지 P가 최근에 사용되었다는 뜻이기 때문에, 교체하기에 좋은 후보가 아니다. 그러므로 페이지 P의 use bit를 0으로 설정하고, 시계 바늘이 다음 페이지(P + 1)로 이동한다.
        - use bit가 0인 페이지를 찾을 때까지 위의 과정을 반복한다.
        - 최악의 경우 전체 페이지를 돌면서 모든 페이지의 use bit를 0으로 설정하게 된다.
    - 80-20에 clock algorithm을 적용한 경우
    ![](https://i.imgur.com/j70hzLZ.png)

- clock algorithm 외에도 use bit를 사용해 LRU의 근사값을 구할 수 있는 방법들은 존재한다.


## 22.9 Considering Dirty Pages

페이지가 수정되었는지, 또는 현재 메모리에 존재하지 않는지를 추가로 고려해보자.


페이지가 수정되었을 경우(**dirty**), 해당 페이지를 내쫓기 위해서는 디스크에 해당 페이지를 쓰는 비싼 작업을 해야 한다. 만약 수정되지 않았을 경우(**clean**), 그냥 내쫓아도 된다.
→ clean 페이지를 내쫓는 것을 dirty 페이지를 내쫓는 것보다 선호하는 시스템들이 존재한다.
- 하드웨어의 **modified bit**(**dirty bit**)를 사용해서 페이지 수정 여부를 알 수 있다.
    - 페이지에 쓰기 작업이 일어나면 해당 비트가 설정된다.
    - clock algorithm을 적용할 수 있다
        - 처음에는 unused & clean 한 페이지를 찾는다.
        - 만약 없으면 unused & dirty 한 페이지를 찾는다.
        - ...

## 22.10 Other VM Policies

페이지 교체 전략 외에도 가상화를 위해 사용하는 전략들이 있다.
- 페이지 선택 전략(page selection policy): OS는 페이지를 언제 메모리에 가져올지를 결정해야 한다.
    - 대부분의 경우 OS는 demand paging을 사용한다.
        - demand paging: 페이지가 접근되었을 때(필요할 때) 페이지를 메모리에 가져온다.
        - prefetching: OS가 다음에 어떤 페이지가 필요할 지 예측해서 미리 가져오는 것. 성공 확률이 높을 때에만 수행되어야 한다.
- OS가 디스크에 페이지를 어떻게 쓸지를 결정해야 한다.
    - 한 번에 하나씩 쓸 수도 있지만, 많은 시스템들은 메모리에 여러 개의 쓰기 작업을 모아둔 후 디스크에 한 번에 쓴다.
    - 메모리에 여러 개의 쓰기 작업을 모으는 것을 clustering 또는 grouping이라고 한다.
    - 디스크는 작은 쓰기 작업을 여러 번 하는 것보다 큰 쓰기 작업을 한 번 수행하는 것이 더 효율적이기 때문이다.

## 22.11 Thrashing

메모리가 oversubscribed 되어서 가용 가능한 물리 메모리보다 실행 중인 프로세스의 메모리 요구가 더 커지면 OS는 어떻게 할까?
이런 경우 시스템은 끊임없이 페이징을 하게 되는데, 이 상태를 **thrashing**이라고 한다.
옛날 OS는 스래싱을 감지하고 해결하기 위해 복잡한 메커니즘을 사용했다.
- admission control: 스래싱이 발생하면 실행 중인 프로세스 중 일부를 실행 중지해서 working set(active하게 사용 중인 페이지들)이 메모리에 들어갈 수 있어 프로세스가 실행되도록 하는 접근법.

일부의 현재 OS는 스래싱에 더 엄격하게 접근한다.
- 리눅스의 일부 버전은 메모리가 oversubscribed 되면 out-of-memory killer를 실행한다.
    - out-of-memory killer는 메모리를 많이 사용하는 프로세스를 선택해서 종료시킨다.
    - X 서버를 종료시켜서 display를 요구하는 애플리케이션이 사용 불가능해 질 수 있는 문제점이 존재한다.


## Homework

### 1.
python paging-policy.py -s 0 -n 10

ARG addresses -1
ARG addressfile
ARG numaddrs 10
ARG policy FIFO
ARG clockbits 2
ARG cachesize 3
ARG maxpage 10
ARG seed 0
ARG notrace False

FIFO
Access: 8  Hit/Miss?  State of Memory? - miss [8]
Access: 7  Hit/Miss?  State of Memory? - miss [8, 7]
Access: 4  Hit/Miss?  State of Memory? - miss [8, 7, 4]
Access: 2  Hit/Miss?  State of Memory? - miss [7, 4, 2]
Access: 5  Hit/Miss?  State of Memory? - miss [4, 2, 5]
Access: 4  Hit/Miss?  State of Memory? - hit [4, 2, 5]
Access: 7  Hit/Miss?  State of Memory? - miss [2, 5, 7]
Access: 3  Hit/Miss?  State of Memory? - miss [5, 7, 3]
Access: 4  Hit/Miss?  State of Memory? - miss [7, 3, 4]
Access: 5  Hit/Miss?  State of Memory? - miss [3, 4, 5]

LRU
Access: 8  Hit/Miss?  State of Memory? - miss [8]
Access: 7  Hit/Miss?  State of Memory? - miss [8, 7]
Access: 4  Hit/Miss?  State of Memory? - miss [8, 7, 4]
Access: 2  Hit/Miss?  State of Memory? - miss [7, 4, 2]
Access: 5  Hit/Miss?  State of Memory? - miss [4, 2, 5]
Access: 4  Hit/Miss?  State of Memory? - hit [2, 5, 4]
Access: 7  Hit/Miss?  State of Memory? - miss [5, 4, 7]
Access: 3  Hit/Miss?  State of Memory? - miss [4, 7, 3]
Access: 4  Hit/Miss?  State of Memory? - hit [7, 3, 4]
Access: 5  Hit/Miss?  State of Memory? - miss [3, 4, 5]

OPT
Access: 8  Hit/Miss?  State of Memory? - miss [8]
Access: 7  Hit/Miss?  State of Memory? - miss [8, 7]
Access: 4  Hit/Miss?  State of Memory? - miss [8, 7, 4]
Access: 2  Hit/Miss?  State of Memory? - miss [7, 4, 2]
Access: 5  Hit/Miss?  State of Memory? - miss [7, 4, 5]
Access: 4  Hit/Miss?  State of Memory? - hit [7, 4, 5]
Access: 7  Hit/Miss?  State of Memory? - hit [7, 4, 5]
Access: 3  Hit/Miss?  State of Memory? - miss [4, 5, 3]
Access: 4  Hit/Miss?  State of Memory? - hit [4, 5, 3]
Access: 5  Hit/Miss?  State of Memory? - hit [4, 5, 3]

### 2.
cache size = 5, worst case streams
FIFO: 1 2 3 4 5 6 1 2 3 4 5 6 / cache size = pages accessed일 때 opt
LRU: 1 2 3 4 5 6 1 2 3 4 5 6 / cache size = pages accessed일 때 opt
MRU: 1 2 3 4 5 6 5 6 5 6 5 / 

1 2 3 4 5
1 2 3 4 6
1 2 3 4 5

### 4.
