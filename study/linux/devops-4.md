# `free` 명령이 숨기고 있는 것들

## `free` 명령어
free를 쳐보자
```bash
$ free -m # -m: MB 반위로 나타냄
              total        used        free      shared     buffers       cache   available
Mem:           7919        1134        4897          72         223        1663        6376
Swap:         95561           0       95561
```
책에서는 `-/+ buffers/cache` 라인이 나오는데 여기서는 안 나옴. 버전이 달라 그럴 것으로 추정

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

used가 0인 것을 보면 스왑 영역을 아직 사용하고 있지 않다는 것을 알 수 있다.

### buffers와 cache 영역
buffers와 cache는 커널이 I/O 작업을 빠르게 하기 위해 사용하는 캐시임

시스템을 띄우고 오래 운영하면 이 영역이 늘어나는 것을 볼 수 있지만, 많이 늘어난 후엔 또 다시 줄어들음
#### buffers (Buffer cache)
super block, inode block을 읽고 캐싱해두는 영역

#### cache (Page cache)
파일 데이터를 캐싱해두는 영역

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
Dirty:                16 kB # Active(file), Inactive(file) 영역 중 디스크로 다시 써져야 하는 부분
...
Slab:             236808 kB
SReclaimable:     196068 kB
SUnreclaim:        40740 kB
...
```

anon/file 영역에 잡혀있는 메모리는 LRU 기반 리스트인 Active와 Inactive 두 가지로 관리

참조 시기가 오래될수록 Active 리스트에서 Inactive 리스트로 이동하게 되고 이후 free 영역으로 이동

### Active -> Inactive
프로세스가 메모리 할당을 요청할 때 Active 리스트에 연결

그 후에 메모리가 부족하게 되면 `kswapd` 또는 커널 내부 `try_to_free_pages()`를 통해 LRU 리스트를 확인

그렇게 되면 Active 리스트에서 Inactive 리스트로 이동이 이루어지거나, Inactive 리스트에 있던 페이지가 해제됨

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <stdbool.h>

#define GiB (long) 1024 * 1024

int main() {
    void * my_block = NULL;
    int count = 0;

    while (true) {
        my_block = (void *) malloc(GiB); // malloc()으로 메모리 할당
        if (!my_block) {
            printf("ERROR!");
            break;
        }
        printf("Currently allocating %ld MB\n", (++count) * GiB);
        memset(my_block, 1, GiB);
        sleep(1);

        if (count == 10) // 10번만 하고 break함
            break;
    }
    sleep(600); // Active -> Inactive 이동이 일어나는지 보기위해 sleep()
    return 0;
}
```
간단한 테스트용 코드


```bash
$ cat /proc/meminfo | grep -i '(anon)'
Active(anon):     100392 kB
Inactive(anon):    45604 kB
$ cat /proc/meminfo | grep -i '(anon)'
Active(anon):     102388 kB
Inactive(anon):    45604 kB
$ cat /proc/meminfo | grep -i '(anon)'
Active(anon):     103544 kB
Inactive(anon):    45604 kB
$ cat /proc/meminfo | grep -i '(anon)'
Active(anon):     104512 kB
Inactive(anon):    45604 kB
$ cat /proc/meminfo | grep -i '(anon)'
Active(anon):     104512 kB
Inactive(anon):    45604 kB
$ cat /proc/meminfo | grep -i '(anon)'
Active(anon):     104512 kB
Inactive(anon):    45604 kB
```

10번 메모리 할당이 이루어지는 동안 Active 영역이 증가하지만 그 뒤에 자동으로 Inactive로 이동하진 않음

커널 파라미터 중 `vm.min_free_kbytes`라는 게 있는데, 시스템이 유지해야하는 free 메모리의 크기를 의미하며 이 값을 높게 설정하면 `kswapd`가 돌게 할 수 있음

크레인 장비에 붙어서 `sudo sysctl -w vm.min_free_kbytes=6553500`을 입력해봤더니 그냥 뻗어버림;

### slab 영역

커널이 사용하는 메모리

```bash
$ cat /proc/meminfo
...
Slab:             205964 kB
SReclaimable:     185864 kB
SUnreclaim:        20100 kB
...
```

1. Slab: dentry cache, inode cache 등 커널이 사용하는 메모리
2. SReclaimable: 재사용되어 프로세스에게 할당될 수 있는 메모리 (주로 캐시 메모리)
3. SUnreclaim: 커널이 현재 사용 중이며, 메모리가 부족해도 해제하여 할당되지 않음

`slabtop` 명령을 통해 slab에 대한 정보를 알 수 있다.

```bash
$ slabtop -o
 Active / Total Objects (% used)    : 647719 / 658539 (98.4%)
 Active / Total Slabs (% used)      : 48801 / 48801 (100.0%)
 Active / Total Caches (% used)     : 88 / 181 (48.6%)
 Active / Total Size (% used)       : 191876.23K / 193291.97K (99.3%)
 Minimum / Average / Maximum Object : 0.02K / 0.29K / 4096.00K

  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
369186 362583  98%    0.10K   9978       37     39912K buffer_head
111236 111221  99%    0.98K  27809        4    111236K ext4_inode_cache
 94200  94146  99%    0.19K   4710       20     18840K dentry
 22036  21906  99%    0.55K   3148        7     12592K radix_tree_node
...
```

### slab 영역의 존재 이유

커널이나 디바이스 드라이버가 I/O 작업을 빠르게 하기 위해 inode cache, dentry cache등을 사용하거나, 네트워크 소켓을 위한 메모리를 확보할 때 slab 영역 사용

메모리가 필요할 때마다 buddy system을 통해 4kb 페이지 단위로 할당 받기엔 너무 자잘한 것들이라 internal fragmentation이 일어나기 쉽기 때문에 slab 할당자가 메모리를 잘게 나누어 사용하는 것

slab은 참고로 `free`를 쳤을 때 used 컬럼에 포함되서 표시된다. slab에서 메모리 릭이 발생하는 경우 `ps` 쳐서 나오는 모든 프로세스 메모리를 더해도 used 컬럼 표시 사이즈와 안 맞게 된다고 한다.

### dentry 캐시 메모리 테스트
책에서는 cd랑 ls를 많이 치면 dentry 캐시 메모리가 올라가는 걸 확인할 수 있다고 했는데 재현이 잘 안되었음.




