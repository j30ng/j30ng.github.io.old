# Load Average와 시스템 부하

## Load Average

### 정의
[proc(5) - Linux manual page](http://man7.org/linux/man-pages/man5/proc.5.html)

> /proc/loadavg
>
> The first three fields in this file are load average figures
> giving the number of jobs in the run queue (state R) or wait‐
> ing for disk I/O (state D) averaged over 1, 5, and 15 minutes.

즉 load average는 커널이 1, 5, 15분 간격으로 R, D 상태에 있는 프로세스 개수의 평균을 구한 값

### 의미
Load average 값이 **높으면** -> 많은 수의 프로세스가 실행 중 or IO 작업을 대기 중  
Load average 값이 **낮으면** -> 적은 수의 프로세스가 실행 중 or 대기 중

의미를 상황에 따라 다르게 다르게 해석해야 할 수 있음. 예를 들면 같은 load average 값이더라도 CPU core의 개수에 따라 부하가 높다는 의미일 수도 있고 낮다는 의미일 수 있다

### load average 확인
`uptime` 명령어로 확인
```bash
$ uptime
 01:10:23 up 34 min,  4 users,  load average: 0.04, 0.14, 0.42
```

## 커널이 Load Average를 계산하는 과정을 알아보자

1. `uptime`을 `strace`
```bash
$ strace -s 65535 -f -t -o uptime_dump uptime
 01:10:36 up 34 min,  4 users,  load average: 0.03, 0.13, 0.42
$ cat uptime_dump
25496 01:10:36 fcntl(4, F_SETLKW, {l_type=F_UNLCK, l_whence=SEEK_SET, l_start=0, l_len=0}) = 0
25496 01:10:36 alarm(0)                 = 10
25496 01:10:36 rt_sigaction(SIGALRM, {SIG_DFL, [], SA_RESTORER, 0x7f8916b974b0}, NULL, 8) = 0
25496 01:10:36 close(4)                 = 0
25496 01:10:36 open("/proc/loadavg", O_RDONLY) = 4
25496 01:10:36 lseek(4, 0, SEEK_SET)    = 0
25496 01:10:36 read(4, "0.03 0.13 0.42 1/586 25496\n", 8191) = 27
25496 01:10:36 fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 7), ...}) = 0
25496 01:10:36 write(1, " 01:10:36 up 34 min,  4 users,  load average: 0.03, 0.13, 0.42\n", 63) = 63
25496 01:10:36 close(1)                 = 0
```
`uptime`은 `/proc/loadavg` file을 읽어서 출력하는 것임을 확인
```bash
$ cat /proc/loadavg
0.03 0.08 0.34 1/584 25543
```
다른 값이 나오는 건 함정

2. `loadavg_proc_show()` - linux-2.6.34.7/fs/proc/loadavg.c
```c
static int loadavg_proc_show(struct seq_file *m, void *v)
{
    unsigned long avnrun[3]; // 1분, 5분, 15분 평균 load average값이 담기는 배열

    get_avenrun(avnrun, FIXED_1/200, 0); // get_avenrun()을 호출하여 avnrun에 값 입력

    seq_printf(m, "%lu.%02lu %lu.%02lu %lu.%02lu %ld/%d %d\n", // Load average값 출력
        LOAD_INT(avnrun[0]), LOAD_FRAC(avnrun[0]),
        LOAD_INT(avnrun[1]), LOAD_FRAC(avnrun[1]),
        LOAD_INT(avnrun[2]), LOAD_FRAC(avnrun[2]),
        nr_running(), nr_threads,
        task_active_pid_ns(current)->last_pid);
    return 0;
}
```
이제 get_avenrun()를 찾아보자

3. `get_avenrun()` - linux-2.6.34.7/kernel/sched.c
```c
/**
 * get_avenrun - get the load average array
 * @loads:	pointer to dest load array
 * @offset:	offset to add
 * @shift:	shift count to shift the result left
 *
 * These values are estimates at best, so no need for locking.
 */
void get_avenrun(unsigned long *loads, unsigned long offset, int shift)
{
    loads[0] = (avenrun[0] + offset) << shift;
    loads[1] = (avenrun[1] + offset) << shift;
    loads[2] = (avenrun[2] + offset) << shift;
}
```
`get_avenrun()`함수는 그냥 load average 값을 옮겨만 주는 함수고, load average값 (avenrun 배열)을 셋팅해주는 함수는  따로 있는 것을 확인 할 수 있음.

4. `calc_global_load()` - linux-2.6.34.7/kernel/sched.c
```c
/*
 * calc_load - update the avenrun load estimates 10 ticks after the
 * CPUs have updated calc_load_tasks.
 */
void calc_global_load(void)
{
    unsigned long upd = calc_load_update + 10;
    long active;

    if (time_before(jiffies, upd))
        return;

    active = atomic_long_read(&calc_load_tasks); // calc_load_tasks 값을 읽어 active 변수에 넣음
    active = active > 0 ? active * FIXED_1 : 0;  // 이 active 값을 바탕으로 load를 계산하게 됨

    avenrun[0] = calc_load(avenrun[0], EXP_1, active);  //  1분 평균 load 계산
    avenrun[1] = calc_load(avenrun[1], EXP_5, active);  //  5분 평균 load 계산
    avenrun[2] = calc_load(avenrun[2], EXP_15, active); // 15분 평균 load 계산

    calc_load_update += LOAD_FREQ;
}
```
`calc_load_tasks`의 값이 어떻게 셋팅되는지 알기 위해 또 다른 코드를 보자

5. `calc_load_account_active()` - linux-2.6.34.7/kernel/sched.c
```c
/*
 * Either called from update_cpu_load() or from a cpu going idle
 */
static void calc_load_account_active(struct rq *this_rq)
{
    long nr_active, delta;

    nr_active = this_rq->nr_running; // Run queue의 nr_running 상태의 프로세스 개수로 초기화
    nr_active += (long) this_rq->nr_uninterruptible; // 여기에 nr_uninterruptible 프로세스 개수 추가

    if (nr_active != this_rq->calc_load_active) { // 이전에 계산된 nr_active와 다르다면 업데이트
        delta = nr_active - this_rq->calc_load_active;
        this_rq->calc_load_active = nr_active;
        atomic_long_add(delta, &calc_load_tasks);
    }
}
```

1~5번 과정을 통해 알아본 것을 정리하면, kernel은 주기적으로 `calc_load_account_active()`를 호출하여 run queue의 active 프로세스 수를 업데이트해 놓는다. 동시에 주기적으로 (5초에 한 번) `calc_global_load()`를 호출하여 active 프로세스 수를 토대로 1/5/15분 평균 load average를 계산한다.

## CPU-bound vs. I/O-bound
load average 값만 가지고는 CPU에 부하가 높은 건지, I/O 병목이 심한 건지 알기 어렵다. `nr_running` 프로세스들과 `nr_uninterruptible` 프로세스들의 합을 토대로 계산해서 그렇다

### CPU-bound
아래와 같은 CPU-bound 앱을 실행시켜 보면,
```python
#!/usr/bin/python3

test = 0
while True:
    test = test + 1
```
```bash
$ ./cpu_bound.py
```
```bash
$ uptime
 02:03:56 up  1:28,  5 users,  load average: 0.05, 0.03, 0.00
$ uptime
 02:04:11 up  1:28,  5 users,  load average: 0.19, 0.06, 0.01
$ uptime
 02:04:26 up  1:28,  5 users,  load average: 0.37, 0.11, 0.03
```
load average가 쭉쭉 올라간다

### I/O-bound
반면 I/O-bound 앱의 경우,
```python
#!/usr/bin/python3

while True:
    f = open("./test.txt", "w")
    f.write("test")
    f.close()
```
```bash
$ ./io_bound.py
```
```bash
$ uptime
 02:09:24 up  1:33,  5 users,  load average: 0.11, 0.07, 0.02
$ uptime
 02:09:34 up  1:33,  5 users,  load average: 0.25, 0.10, 0.03
$ uptime
 02:09:48 up  1:34,  5 users,  load average: 0.41, 0.14, 0.04
```
마찬가지로 쭉쭉 올라간다

따라서 load average가 높다고 해서 단순히 CPU가 더 많이 꼽힌 장비를 쓴다고 나아지는 것이 아니다

### `vmstat`
load average에 더해서 어떤 종류의 부하가 일어나는 것인지 알고 싶으면 `vmstat`을 사용

cpu_bound.py를 실행시켜보자
```bash
$ ./cpu_bound.py
```
그리고 `vmstat` 실행
```bash
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 4203600 227224 2848580    0    0     0     0  107  824  0  0 100  0  0
 1  0      0 4200500 227224 2848580    0    0     0     0  301  730  9  0 91  0  0
 3  0      0 4200500 227224 2848580    0    0     0     0  336  646 13  0 87  0  0
 1  0      0 4200500 227224 2848580    0    0     0     0  365  679 13  0 87  0  0
 1  0      0 4200500 227228 2848580    0    0     0    24  355  690 13  0 87  0  0
 ...
```

이번엔 io_bound.py를 실행시켜보자
```bash
$ ./io_bound.py
```
```bash
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 4202176 227252 2848536    0    0     0     0  103  654  0  0 100  0  0
 0  1      0 4199200 227252 2848536    0    0     0  9180 2584 7673  1  2 94  3  0
 0  1      0 4198952 227252 2848536    0    0     0 31304 8304 24389  2  4 87  7  0
 0  1      0 4198952 227252 2848540    0    0     0 31628 8407 24652  2  4 87  7  0
 0  1      0 4198952 227252 2848540    0    0     0 28620 7799 22390  2  3 87  7  0
...
```
`vmstat`이 찍어내는 라인 앞쪽 컬럼 중에 보면 `r`과 `b`가 있는데, 각각 running, blocked를 의미한다. 따라서 load average값과 더불어서 `vmstat`에서 지속되는 `r`, `b` 값을 보고 어떤 종류의 부하인지 유추 가능


