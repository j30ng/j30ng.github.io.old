# `free` 명령이 숨기고 있는 것들

## `free` 명령어
free를 쳐보자
```bash
$ free -m
              total        used        free      shared     buffers       cache   available
Mem:           7919        1134        4897          72         223        1663        6376
Swap:         95561           0       95561
```
책에서같이 `-/+ buffers/cache` 라인은 안 나옴. 버전이 달라 그럴 것으로 추정

1. total: 시스템에서 인식하는 총 메모리
2. used: 시스템에서 사용하고 있는 메모리
3. free: 시스템이 사용하지 않고 있는 메모리
4. shared: 프로세스 사이에 공유되는 메모리
5. buffers: 커널 버퍼로 사용하고 있는 메모리
6. cache: 페이지 캐시와 slab을 의미
7. available: (책에 없음) 새 애플리케이션을 띄우는 데 얼마만큼의 가용 메모리가 있는지에 대한 예측

두 번째 라인 swap
```bash
              total        used        free      shared     buffers       cache   available
...
Swap:         95561           0       95561
```

1. total: 총 swap 영역 크기
2. used: 사용 중인 swap 영역의 크기
3. free: 사용 가능한 swap 크기

### buffers와 cache 영역
buffers와 cache는 커널이 I/O 작업을 빠르게 하기 위해 사용하는 캐시임

시스템을 띄우고 오래 운영하면 이 영역이 늘어나는 것을 볼 수 있지만, 많이 늘어난 후엔 또 다시 줄어들음
#### buffers (Buffer cache)
super block, inode block을 읽고 캐싱해두는 영역

#### cache (Page cache)
data block 즉 파일의 내용을 캐싱해두는 영역

## /proc/meminfo
```bash
$ cat /proc/meminfo
MemTotal:        8109612 kB
MemFree:         5007772 kB
MemAvailable:    6521628 kB
Buffers:          229032 kB
Cached:          1466844 kB
SwapCached:            0 kB # 스왑으로 빠졌다가 다시 메모리에 올라온 크기
Active:          2050508 kB
Inactive:         592640 kB
Active(anon):     947188 kB # 자주 읽히고 있는 유저 영역 메모리 크기
Inactive(anon):    74380 kB # 자주 안 읽히고 있는 유저 영역 메모리 크기
Active(file):    1103320 kB # 자주 읽히고 있는 파일 캐시 크기
Inactive(file):   518260 kB # 자주 안 읽히고 있는 파일 캐시 크기
...
SwapTotal:      97855484 kB
SwapFree:       97855484 kB
Dirty:                16 kB # Active(file), Inactive(file) 영역 중 
...
Slab:             236808 kB
SReclaimable:     196068 kB
SUnreclaim:        40740 kB
...
```