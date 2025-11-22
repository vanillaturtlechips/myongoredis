# myongoredis
Go 언어로 구현한 In-Memory Key-Value Store입니다. Redis의 RESP(Redis Serialization Protocol) 규약을 준수합니다. 다양한 동시성 제어(Concurrency Control) 전략에 따른 성능 변화를 `go tool trace`로 시각화하고 분석하기 위해 개발되었습니다. 이 프로젝트는 단순한 기능 구현을 넘어, Production Level의 TCP 네트워크 프로그래밍 모범 사례(Best Practices)를 적용하는 데 중점을 두었습니다.

## 목표
1. Lock Contention Visualization: Mutex vs RWMutex vs Sharded Map 전략에 따른 성능 차이와 고루틴 스케줄링 병목 현상을 trace 도구로 직접 확인합니다.
2. TCP Network Programming: Timeouts, Graceful Shutdown, KeepAlive 등 견고한 네트워크 서버를 위한 핵심 패턴을 구현합니다.
3. Redis Internals: RESP 프로토콜 파싱과 인메모리 자료구조 설계(TTL, Eviction)를 통해 Redis의 동작 원리를 깊이 이해합니다.

## 아키텍처
실행 시 플래그(-mode)를 통해 동시성 제어 전략을 동적으로 교체할 수 있습니다.

1. Storage Engine (Pluggable)
실행 시 플래그(-mode)를 통해 동시성 제어 전략을 동적으로 교체할 수 있습니다.

- Global Mutex Mode: 단일 sync.Mutex로 모든 요청을 동기화 (Baseline).
- RWMutex Mode: sync.RWMutex를 사용하여 읽기(GET) 병렬 처리 최적화.
- Sharded Map Mode: Key 해싱(FNV-1a)을 통해 32개(기본값)의 샤드로 락을 분산시켜 Lock Contention 최소화.

2. TCP Server Best Practices

운영 환경을 고려한 안정성 확보 패턴이 적용되었습니다.

- [x] I/O Timeouts: 좀비 커넥션 및 Slowloris 공격 방지를 위한 SetDeadline 적용.
- [x] Graceful Shutdown: SIGTERM 수신 시 처리 중인 요청을 완료하고 Trace 데이터를 안전하게 저장 후 종료.
- [x] Backpressure: 세마포어(Semaphore) 패턴을 이용한 최대 동시 접속자 수 제한 (MaxConn).
- [x] Monitoring: expvar를 통한 실시간 메트릭 노출.

3. Protocol
- RESP (Redis Serialization Protocol): Redis 클라이언트(redis-cli)와 호환되는 통신 규약 구현.
- Supported Commands: SET, GET, PING, DEL

## 빌드하는 법

1. 빌드
```bash
git clone https://github.com/myong3/godis.git
cd godis
go mod tidy
```
2. Run Server
```bash
# Mode: mutex | rw | shard (default: mutex)
go run cmd/server/main.go -port 6379 -mode shard
```
3. Connect (via redis-cli or tcpkali)
```bash
redis-cli -p 6379
127.0.0.1:6379> SET mykey hello
OK
127.0.0.1:6379> GET mykey
"hello"
```
## 성능 분석

이 프로젝트의 핵심 실험입니다. trace.out을 통해 락 경합을 시각적으로 확인합니다.

1. Trace Start
```bash
서버 실행 시 자동으로 trace.out 파일이 생성됩니다. (서버 종료 시 저장됨)
```

2. Load Testing
tcpkali를 사용하여 동시 접속 부하를 발생시킵니다.
```bash
# 50개의 커넥션으로 10초간 SET/GET 반복 요청
tcpkali -c 50 -T 10s -m "SET key-{connection.index} value\r\nGET key-{connection.index}\r\n" 127.0.0.1:6379
```
3. Visualize
서버를 종료(Ctrl+C)한 후, 트레이스 도구를 실행합니다.
```bash
go tool trace trace.out
```
- Synchronization Blocking Profile: 고루틴이 락을 획득하기 위해 대기한 시간을 확인합니다.
- View Trace: 타임라인에서 고루틴들이 병렬로 실행되는지, ProcStop 상태로 대기하는지 비교합니다.


## 로드맵

- [ ] TTL (Time To Live): Lazy Expiration 패턴을 이용한 데이터 만료 구현.
- [ ] Eviction Policy: MaxMemory 도달 시 LRU/Random 키 축출 알고리즘.
- [ ] AOF Persistence: 데이터 영속성을 위한 파일 쓰기 구현.
- [ ] Pipeline Support: Request Pipelining 처리 최적화.


   
