# 7 CPU Scheduling

**스케줄링 정책**(**discipline**이라고도 함)


## 7.1 Workload Assumptions

시스템에서 실행되고 있는 프로세스들을 합쳐서 **workload**, 프로세스를 **job**이라고도 부른다. 시스템에서 실행 중인 프로세스에 대해 다음과 같이 가정할 것이다.
1. 모든 job은 걸리는 시간이 동일하다.
2. 모든 job은 동시에 도착한다.
3. job은 일단 시작되면 완료될 때까지 실행된다.
4. 모든 job은 CPU만을 사용한다 (예를 들어 I/O 작업은 수행하지 않음)
5. 각 job의 실행 시간을 알고 있다.

## 7.2 Scheduling Metrics

**Scheduling Metric**: 스케줄링 정책을 비교하기 위한 것. 평가 기준은 크게 아래 두 가지가 있다. 
- 성능(performance)
    - **turnaround time(반환시간)**: job이 완료된 시간 - job이 도착한 시간
- 공정성(fairness)
- 이 둘은 스케줄링 정책에서 종종 상충한다.

## 7.3 First In, First Out (FIFO)

= First Come First Served (FCFS)
선입선출.
먼저 도착한 작업의 수행시간이 길 경우, 평균 반환 시간이 커진다. 이것을 **Convoy Effect**라고 한다.
- Convoy Effect: 상대적으로 자원이 적게 드는 프로세스가 자원이 많이 드는 프로세스보다 큐의 뒤에 위치하게 되는 것.

## 7.4 Shortest Job First (SJF)

모든 job이 동시에 도착할 경우에는 SJF가 최적의 알고리즘이다. 그렇지 않을 경우, 현재 도착해 있는 작업 중 가장 수행 시간이 짧은 작업을 우선적으로 수행-완료한 후 다음 작업을 수행하기 때문에 수행 시간이 긴 작업이 가장 먼저 도착하면 또 convoy 문제가 발생한다.

## 7.5 Shortest Time-to-Completion First (STCF)

= SJF + preemptiveness
= Shortest Remaining Time (SRT)
= Preemptive Shortest Job First(PSJF)

SJF의 경우 비선점적인(non-preemptive) 스케줄러이다. STCF는 선점이 가능하기 때문에, 새로운 job이 도착하면 남은 모든 작업 중에서 가장 실행(완료)시간이 적게 걸리는 job을 골라서 그 job을 시작한다 (기존에 실행하던거 중단함).

## 7.6 A New Metric: Response Time

**Response Time**: job이 처음 실행된 시간 - job이 도착한 시간
SJF/STCF의 경우 실행시간이 긴 작업은 응답 시간이 엄청나게 길어지게 된다.

## 7.7 Round Robin (RR)

= time-slicing (시분할)
응답 시간에 민감한 스케줄링 알고리즘.
RR은 job을 **time slice(scheduling quantum)** 동안 실행한다. 한 time slice동안 실행하고 나면, 다음 job을 또 time slice동안 실행.. 이걸 계쏙 반복
- time slice의 길이는 timer-interrupt period의 배수이다.
    - 예: timer가 10 milisec마다 인터럽트 한다면, time slice는 10, 20과 같은 10 milisec의 배수이다.
- RR에서는 time slice의 길이가 매우 중요하다.
    - time slice가 짧을 수록 response 지표에서 RR의 성능이 좋아진다.
    - time slice가 너무 짧으면 context switching의 비용이 너무 커져서 성능이 저하된다.
    - turnaround time 지표에서의 성능은 나쁘다.. (다같이 늦게 끝나니까)
    → fair한 스케줄링 정책은 turnaround time은 구리다

## 7.8 Incorporating I/O

I/O 요청을 시작하는 job이 있을 경우, I/O가 진행되는 동안 그 job은 CPU를 사용하지 않는다. 이걸 I/O 작업이 완료될 때까지 **blocked** 되었다고 한다. blocked 된 동안에 스케줄러가 CPU에 다른 job을 스케줄링 해야 한다.
I/O가 완료되어 인터럽트가 발생하면, OS는 I/O를 발행한 프로세스를 blocked 상태에서 ready 상태로 이동시킨다.
![](https://i.imgur.com/XGgkOcA.png)

## Homework

1. 3 jobs, length = 200
    - SJF: response time = (0 + 200 + 400) / 3 = 200, turnaround time = (200 + 400 + 600) / 3 = 400
    - FIFO: 동일함

2. 3 jobs, length = 100, 200, 300
    - SJF: response time = (0 + 100 + 300) / 3, turnaround time = (100 + 300 + 600) / 3
    - FIFO: 동일함

3. 3 jobs, length = 100, 200, 300, RR time slice = 1
    - response time = (0 + 1 + 2) / 3 = 1
    - turnaround time = (598 + 599 + 600) / 3

4. when all workloads have same length or when workloads arrive in increasing manner

5. when the length of time slice is greater than or equal to the length of most time-consuming job

6. response time increases
```
python ./scheduler.py -p SJF -j 5 -l 200,100,200,300,100 -c
```
response time = 260, turnaround time = 440, wait time = 260

```
python ./scheduler.py -p SJF -j 5 -l 400,200,400,600,200 -c
```
response time = 520, turnaround time = 520, wait time = 520

```
python ./scheduler.py -p SJF -j 5 -l 400,300,500,300,200 -c
```
response time = 540, turnaround time = 880, wait time = 540

→ given that all jobs arrive at the same time (at the beinning), response time = (new workload)/(previous workload)

7. (0 + N + 2N + ... N * N) / N = (1 + 2 + ... + N)
