## 비동기 프로그래밍
### 동기 서버
- 반복 서버(interactive server): 클라이언트로부터 요청받은 순서대로 처리하는 서버
- 동시 서버(concurrent server): 요청을 동시에 처리하는 서버
- 반복 서버는 순차적으로 실행되기 때문에 앞서 들어간 요청이 시간이 오래걸리면 throughput이 낮아지게 됩니다.
