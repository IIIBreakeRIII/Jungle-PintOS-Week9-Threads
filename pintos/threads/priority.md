## Priority Scheduling 구현
- thread 구조체에 있는 priority 가 큰 쓰레드부터 실행되도록 ready list 에 삽입
    - 나는 삽입 시 정렬되도록 insert_ordered 사용했고 pop 은 항상 pop_front()사용
- 선점(preempt) 적용
    - 선점이란 현재 CPU를 점유하고 있는 스레드보다 새로 생성된 스레드 혹은 우선순위가 변경된 스레드의 우선순위가 더 큰 경우
    - 바로 해당 스레드로 yield() 처리하는 것
- 세마포어
    - value 값으로 실행흐름을 제어, 이 때 thread_block() 과 thread_unblock() 함수를 기준으로 그 이후 코드의 실행이 끊어지는데 이걸 갖고 value 값을 제어하고 있음

- 실행흐름에 대해서..
어떤 쓰레드가 block 되거나 아무튼 제어권을 양보하면 다음 실행은 바로 그 다음줄부터 실행됨
이게 은근... 많이 헷갈림

## 테스트별 트러블 슈팅

### priority-sema
세마포어의 역할이 뭔지 이해를 못해서 코드 흐름이 이해가 안갔었음
```

  (priority-sema) begin
+ (priority-sema) Back in main thread.
  (priority-sema) Thread priority 30 woke up.
  (priority-sema) Back in main thread.
  (priority-sema) Thread priority 29 woke up.
  (priority-sema) Back in main thread.

```
    - 위의 테스트 결과에서 두번째 줄의 `Back in main thread.` 는 출력되면 안됨
    - 왜냐하면 메인은 바로 priority 30짜리 스레드한테 실행흐름을 넘겼기 때문임
    - 근데 자꾸 나왔었음
    - 문제는 이 부분이였음

    ```C
    void sema_up(struct semaphore *sema) {
    enum intr_level old_level;

    ASSERT(sema != NULL);

    old_level = intr_disable();
    sema->value++;
    if (!list_empty(&sema->waiters)) {
        thread_unblock(
            list_entry(list_pop_front(&sema->waiters), struct thread, elem));
    }
    //sema->value++;
    intr_set_level(old_level);
    }
    ```
    - 주석친 부분이 이전에 세마포어 자원을 늘리는 코드의 위치였음
    - 즉 블록처리 이후에 자원을 늘리고 있었는데 블록처리를 하면 해당 시점에 현재 쓰레드가 블락되면서 실행흐름이 끊어짐
    - 실제로 정확히 현재실행흐름이 끊어지는 함수는 `thread_launch()` 이다
    - 그렇기 때문에 세마포어의 자원을 늘려준 후 unblock 처리를 해야함
    - 왜냐하면 unblock 을 하면 대기리스트에 있는 가장 우선순위가 높은 스레드가 깨어나는데 그 때 세마포어값이 0이라면 다시 잠들 수밖에 없음
    - 아래코드가 sema_down 으로 자원획득을 할 때 호출하는 함수인데 sema->value == 0이면 바로 다시 현재 스레드를 block 처리해버림
    - 그렇기 때문에 sema_up(자원반환)에서 자원을 먼저 증가시키고 block 해제를 해야한다!

    ```C
    void sema_down(struct semaphore *sema) {
    enum intr_level old_level;

    ASSERT(sema != NULL);
    ASSERT(!intr_context());

    old_level = intr_disable();
    while (sema->value == 0) {
        //우선순위 기준으로 세마포어 대기리스트에 삽입
        list_insert_ordered(&sema->waiters, &thread_current()->elem,
                            compare_t_priority, NULL);
        //현재 스레드 block -> 대기 리스트의 다음 스레드 run
        thread_block();
    }
    sema->value--;
    intr_set_level(old_level);
    }
    ```

### priority-condvar
- 조건변수가 대체 뭔지 이해가 안가서 힘듬
- 정확히는 세마포어랑 락이랑 조건변수 같이 나오니까 뭐가 뭔지 헷갈려서 실행흐름을 쫒아갈 수 가 없었음
- 아무튼 이제 개략적으로는 알고 있다고 생각함
- 이따 donate 까지 하고 정리필요

### priority-danate-multiple



