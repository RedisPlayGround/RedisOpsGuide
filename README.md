# 상용환경에서 레디스를 운영하며 겪거나 겪을 수 있는 장애처리 및 이슈들 해결 정리


## Redis 장애 처리

### Redis 장애의 원인

- Redis는 Main Thread가 대부분의 처리를 하는 Single Thread 형태이기 때문에 하나의 명령에서 시간이

    많이 걸리면 전체 성능 저하가 일어난다
- 실제로 더 빠른 CPU, 더 많은 메모리를 달수록 성능이 좋아지지만, Scale Up을 하더라도, 잘못된 사용 패턴은 장애를 일으킬 수 있다

### Redis 장애 종류

||||
|:--:|:--:|:--:|
|장애 타입| 소분류| 내용|
|메모리| 메모리 과다 사용<br/> (maxmemory 설정)|Maxmemory가 설정되었을 때, maxmemory policy에 따라서, 더이상 eviction 할 수 <br/> 있는 메모리가 없다면 OOM 에러를 Redis가 전달한다|
|메모리|RSS 관리|Redis에서 실제로 물리메모리보다 더 많은 메모리를 사용하면, 해당 페이지에 Swap이 발생하고, 이에 접근할 때마다<br/> 디스크 페이지에 접근할 수 있어서 성능이 떨어진다. 실제 used memory와 RSS 사용은 다르다|
|설정|기본 설정 사용|기본 설정으로 사용 시에 SAVE 설정이 1시간에 1개, 5분에 1000개 1분에 10000개가 변경이 된다면 디스크에 메모리를 <br/> 덤프하게 되므로, IO를 과다하게 사용해서 장애가 발생한다. (실서비스에서 1분에 10000개 변경은 항상일어나는 상황)|
|싱글 스레드| 과도한 Value 크기|Redis는 싱글스레드이기 때문에 하나의 명령이 긴 시간을 차지하면 결국 Redis 성능저하로 이어진다. <br/> Hgetall, hwals등의 collection의 데이터를 과도하게 많이 가져온다거나, 몇 MB 이상의 Key나 <br/> Value를 사용할 경우 문제가 발생한다|
|싱글 스레드| O(N) 명렁의 사용|Keys나 flushdb/flushall, 큰 크기의 collection을 지우는 등의 문제 역시, Redis의 성능을 떨어트린다|

---

### Redis 메모리 관련 장애
- Redis 메모리 부족으로 인한 이슈
  - Redis의 처리 속도가 떨어진다 (Swap 등의 이슈)
  - OOM 이슈로 처리가 되지 않고 명령이 실패한다

해결방법으로는 다음과 같은 두가지가 있다

1. Scale Up
2. Key를 지워서 메모리 확보 (이부분은 키를 지워도 메모리를 바로 회수 안할 수도 있음)

#### Redis 메모리 관련 - Copy on Write 이슈
- Redis가 fork할 때, 메모리 사용량이 최대 두 배까지 늘어날 수 있다
- Redis가 fork를 하게되는 경우
  - Replica가 연결이 되는 순간 데이터 이전을 위한 RDB 만들면서 fork
  - AOF Rewrite를 하기 위한 경우 (bgrewriteaof 명령) fork
  - RDB 생성을 하기 위한 경우 (bgsave 명령) fork
- COW(Copy on Write)
  - Fork를 할 경우 부모와 자식 프로세스가 읽기용 메모리는 공유해서 메모리를 절약함
  - 공유 메모리에 쓰기가 발생하는 프로세스가 해당 메모리를 복사해서 사용하게 됨
  - Redis는 Child Process는 RDB 생성을 담당하기 때문에 읽기만 주로 발생 

    ![image](https://github.com/YassinAJDI/PopularMovies/assets/40031858/28a3e774-9e03-43ff-bea7-64403e1dd79f)
  - 해당 메모리 Page에 변경이 발생한 해당 프로세스가 해당 메모리 Page를 복사하고 여기에 Write를 실행한다
  - 이로 인해, 전체 공유된 Page에 Write가 발생하면 최대 메모리 사용량이 두 배까지 증가 

    ![image](https://github.com/YassinAJDI/PopularMovies/assets/40031858/dc4a4873-2089-4751-8401-afbe93c06da9)

- `현재 이를 해결 하는 좋은 방법은 Redis측에서 없다`
  - `따라서, 한 대의 Redis 서버의 메모리 사용량을 물리 메모리의 절반 이하로 유지하자`


### Redis 메모리 관련 장애 확인 방법

#### Application 레벨에서 확인
- API별 Latency를 측정한다
- Redis 관련 호출에서 시간이 얼마나 걸리는지 확인한다
- Redis 응답이 무엇이 오는지 확인한다

#### Redis 레벨에서의 확인
- 메모리 사용량이 얼마인지 확인한다
  - Used memory와 RSS가 현재 설정된 메모리 크기와 유사할 경우 문제가 있다
  - OS와 다른 프로세스에서 사용하는 메모리 사이즈가 있으므로, 4G머신에서 만약 3G 이상 Redis가 쓰고 있다면 문제의 소지가 있을 수 있다
- Slow Log가 남는지 확인한다

## `Redis 메모리 관련 장애 해결방법 정리`

### 첫 번째 방법
- 메모리가 더 큰 장비로 업그레이드 한다
  - 비용이 들어가지만 가장 좋은 방법
- Redis는 항상 Fork의 위험성이 있으므로, 메모리가 충분할 수록 좋다

### 두 번째 방법
- 효율성이 떨어지는 Key를 지운다
  - Redis는 개별 Key의 Hit, Miss를 보여주지 않으므로, 개별로 따로 관리해야 한다
  - 실제 캐시되는 키가 줄어드므로, DB부하가 늘어날 수 있다

---

### Redis 기본 설정 관련 장애
- Redis 기본 설정 사용으로 인한 과도한 IO로 인한 성능 저하
- Redis가 bgsave 동작이 Fork로 인해서 메모리 사용량도 늘어날 수 있다

### Redis 기본 설정 관련 장애 확인 방법

- 서버 레벨에서의 확인
  - Disk 사용량을 확인한다. Bgsave 옵션으로 인해서 Write가 많아지므로 Write가 많아지는지 확인한다
- Redis 레벨에서의 확인
  - SAVE 관련 설정을 확인한다. (서비스를 위해 해당 설정은 끄는 것이 좋음)
    - config get save

        ![image](https://github.com/TTTAttributedLabel/TTTAttributedLabel/assets/40031858/a99f9584-bb69-45ac-bcd5-67a765608074)

---

### Redis Single Thread 관련 장애

- 과도한 Value로 인해 발생하는 장애
  - Redis의 Sorted Set, Hash, Set등의 자료구조는 내부적으로 다시 Hash Table등을 구성해서 관리한다

### Redis Single Threaded 관련 장애 확인

- 사용하면 안되는 명령을 사용중인지 확인한다
  - KEYS 명령의 사용 횟수가 계속 늘어나면 해당 명령이 문제를 일으킬 수 있다
- O(N) 계열 커맨드의 사용이 늘어나는지 확인한다
  - Hgetall, hwals, smembers, zrange 계열 함수 (usec_per_call 이 두자리나 세자리면 체크해야함)
- Monitor 명령을 통해서 들어오는 KEY들의 빈도를 체크한다
  - Monitor명령은 해당 서버에 부할르 추가로 주게되므로 사용하면서 서버 부하가 더 커지는지 확인해야 한다
- Scan 명령을 통해서 각 Key의 사이즈를 확인해서 특정 크기 이상의 KEY를 확인한다

- Info all 에서 commands stats를 확인한다
  - usec_per_call이 micro seconds이므로 주요 호출 명령에서 해당 값이 크면 확인이 필요하다
    ![image](https://github.com/TTTAttributedLabel/TTTAttributedLabel/assets/40031858/128e211e-fab7-4a54-b8e7-35bc380a3f7e)

- 특정 Key의 사이즈를 확인하는 방법은 다 돌리는 수밖에 없다. (keys대신 scan을 사용하자)

---

### O(N) 명령으로 인한 장애

- Redis는 싱글 스레드이므로 O(N)명령은 Redis성능에 영향을 미치는 대표적인 부분인다
  - 대표적으로 Redis에 어떤 Key들이 있는지를 확인하기 위해서 KEYS명령을 사용하는 경우이다
  - KEYS명령 이외에도, Collection을 삭제하거나, 모두 가져오는 케이스들이 있다
    - Ex) hgetall, smembers 

### KEYS 명령으로 인한 장애 확인

- info all 명령에서 KEYS의 calls 수가 계속 증가하는지 확인한다.

### KEYS 명령으로 인한 장애 해결

- KEYS 명령 대신에 scan 명령으로 바꾼다

#### Spring Security OAuth 성능 개선 사례

- 2018년 6월까지 버전에는 RedisTokenStore를 통해서 oauth access token을 관리할 경우, token개수가 많아지면

    전체 인증 로직이 느려지는 버그가 존재
- 버그의 이유
  - O(N)자료구조인 list 자료구조를 사용
  - 백만개의 token 중에서 하나의 token을 찾아야 하는 경우가 반복적으로 발생
- 실제로 국내외 많은 서비스에서 해당 이슈로 인한 장애가 발생

---

### Redis 보안 관련

- 절대로 Redis의 포트를 Public에 공개하면 안된다
  - Redis 6부터는 ACL 설정을 제공
- 해당 이슈로 전체 시스템의 보안 윟벼이 높아지는 경우가 많음

