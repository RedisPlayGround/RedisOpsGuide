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

---

## Redis 상용 이슈들

### Thundering Herd

- 특정 이벤트로 인해서 많은 프로세스가 동작하는 데, 그 중에 하나의 프로세스만 이벤트를 처리할 수 있어서,

    많은 프로세스가 특정 리소스를 가지고 경쟁하면서 많은 리소스를 낭비하게 되는 경우
- 웹 서비스에서는 Cache Miss로 인해서, 많은 프로세스가 같은 Key를 DB에서 읽으려고 시도하면서, 특정 서버에

    부하를 극도로 증가시키는 경우를 의미한다

![image](https://github.com/TTTAttributedLabel/TTTAttributedLabel/assets/40031858/b7a0aebb-2523-45bb-a2e3-dca905112ac4)

### Thundering Herd의 원인

- 캐시가 없을 때 발생함
  - 캐시가 없는 현상의 이유
    - 캐시 서버의 추가/삭제
    - 해당 키의 TTL에 의한 데이터 삭제
    - 캐시 서버 메모리 부족으로 해당 키의 Eviction

### Cache Stampede

- Cache의 Expire Time 설정으로 인해서 대규모의 중복된 DB쿼리와 중복된 Cache 쓰기가 발생하는 현상(Thundering Herd)

### Probabilistic Early Recomputation

- Cache Stampede를 해결하기 위한 방법 중의 하나
- 키의 TTL이 완료하기 전에 Random 한 가상의 Expire Time을 설정해서 미리 키의 내용을 갱신하는 방법
![image](https://github.com/TTTAttributedLabel/TTTAttributedLabel/assets/40031858/3e68fc45-b45f-48ef-9930-61486e12bd7d)
- DELTA, BETA 값을 지정하게 된다
- DELTA는 실제로 캐시 재계산을 위한 시간 범위
  - 예) 대략 500ms 근처에서 재계산이 일어나면 좋겠다
- BETA는 여기에 다시 가중치를 준다
  - 기본으로는 1.0을 사용
  - BETA < 1.0 은 좀 더 소극적으로 재계싼을 하게 된다
  - BETA > 1.0 은 좀 더 적극적으로 재계산을 하게 된다
- Expiry는 캐시가 Expire될 시간을 말한다
  - Expiry = now() + 남은 ttl 시간
- 즉 계산식은 다음과 같다
  - Now() + abs(DELTA * BETA * log(random())) > expiry
  - Now() + abs(DELTA * BETA * log(random*())) > Now() + ttl_ms
  - ttl_ms - abs(DELTA* BETA * log(random())) > 0


## 예시

```java
@Service
public class RedisService {

    private RedissonClient redissonClient;

    @Value("${redis.host}")
    private String redisHost;

    @Value("${redis.port}")
    private int redisPort;

    @PostConstruct
    public void init() {
        Config config = new Config();
        config.useSingleServer().setAddress("redis://" + redisHost + ":" + redisPort);
        redissonClient = Redisson.create(config);
    }

    public String getValue(String key) {
        RBucket<String> bucket = redissonClient.getBucket(key);
        String value = bucket.get();
        if (value == null) {
            if (shouldRecompute(key)) {
                value = doExpensiveComputation();
                bucket.set(value, Duration.ofMinutes(5));
            } else {
                waitRandomTime();
                return getValue(key);
            }
        }
        return value;
    }

    private boolean shouldRecompute(String key) {
        double probability = hash(key) % 0.1;
        double threshold = 0.05;
        return probability < threshold;
    }

    private void waitRandomTime() {
        try {
            Thread.sleep(new Random().nextInt(10) + 1);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    private String doExpensiveComputation() {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        return "value";
    }

    private int hash(String key) {
        return key.hashCode();
    }
}
```

---

### Hot Key

- 과도하게 읽기/쓰기 요청이 집중되게 되는 key
- 해당 Key의 접근으로 인해서 Storage( DB, Cache) 성능 저하가 발생하는 Key를 Hot Key라고 부른다

![image](https://github.com/TTTAttributedLabel/TTTAttributedLabel/assets/40031858/bb023931-86b7-4c60-8038-ca02849837f1)

### Hot Key에 대한 일반적인 해결책
- Query Off(Read From Secondary)
- Local Cache
- Multiple Write And Read From One

### Query Off
- Redis의 경우 Replication 기능을 제공해 준다
- Write는 Primary, Read는 Secondary를 이용하여, Read 부하를 줄일 수 있다
- AWS ElasticCache Redis의 경우 최대 5개의 읽기 전용 복제본을 추가할 수 있다

#### 읽기 분배 - QUERY OFF

![image](https://github.com/TTTAttributedLabel/TTTAttributedLabel/assets/40031858/cc201b30-d5c6-42d3-8ff2-9f9e72fb683b)

### API 서버에서의 Local Cache

API 서버에서 직접 특정 Key들을 Cache해서 Cache 서버에 가지 않고 API 서버에서 바로 처리한다 (API 서버 수만큼 처리량이 나눠짐)

![image](https://github.com/TTTAttributedLabel/TTTAttributedLabel/assets/40031858/1f7652be-db62-4ff3-ac60-f0dc26aea89e)

#### Local Cache 단점
- Cache가 공유되지 않고 API 서버에만 존재
- 데이터가 변경되었을 때, 이를 통지 받는 메커니즘이 필요
  - 변경 통지가 없다면, TTL이 끝날 때까지 데이터의 불일치가 발생
- 이런 문제의 해결점으로 Client-Side Cache을 제공하는 Cache 솔루션이 있다

### Multi Write Read One
- Cache를 써야 할때 하나의 Key를 남기는 것이 아니라, 여러 개의 키로 남기고 읽을 때는 하나의 Key만 읽어서 부하를 분산
- APi 서버에서 여러 Cache 서버에 Write를 한다
- Write(A) => Write(A1), Write(A2), Write(A3)
- Read(A)  => Read(A1) or Read(A2) or Read(A3)

![image](https://github.com/TTTAttributedLabel/TTTAttributedLabel/assets/40031858/30b355d1-9c73-4357-9113-0d2fdccec073)

![image](https://github.com/TTTAttributedLabel/TTTAttributedLabel/assets/40031858/006da94d-c252-4c83-b67c-2030cef7ea88)

#### Multi Write Read One의 단점
- 좀더 많은 Cache 장비를 사용하여야 한다 ㅠ


### Timeout

![image](https://github.com/TTTAttributedLabel/TTTAttributedLabel/assets/40031858/1f0630d1-72a5-4df8-be76-8aaf95b56b0d)

위와 같은 경우 정상적인 경우 Add가 문제 없다

다만 Callee가 먼저 TimeOut이 발생하는 경우
- Callee가 먼저 Timeout이 발생하면 Caller에서 에러가 리턴된다


![image](https://github.com/TTTAttributedLabel/TTTAttributedLabel/assets/40031858/2270e412-46e9-4063-a522-a22e32cd87c7)

Caller가 먼저 Timeout이 발생하는 경우
- Caller가 먼저 Timeout이 발생하면 Callee는 계속 실행이 된다
- Callee가 딱 Processing Timeout직전에 수행이 완료되면, 실제 데이터는 이미 들어가 있는 상태가 된다
- Retry를 한다면 데이터 중복이 발생하게 된다
![image](https://github.com/TTTAttributedLabel/TTTAttributedLabel/assets/40031858/6a6d9772-72b5-4404-93bb-69adb4942f4a)

### TimeOut설정 결론

- 항상 caller의 Timeout 설정이 Callee보다 커야한다

![image](https://github.com/TTTAttributedLabel/TTTAttributedLabel/assets/40031858/e7e17416-ddbb-4025-8dcb-2b7e60e19014)