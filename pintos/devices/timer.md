## Alarm-Clock의 구현

> Busy Waiting?

- 프로그램이 다른 작업을 수행하며 기다림
    - 특정 조건이 충족할 때까지 반복적으로 체크하며 기다림

> Alarm-Clock?

- 일정 시간이 지난 후에 다음 작업을 수행하도록 "예약"하는 것
    - 프로그램은 대기 상태에 들어간 동안 사이클 사용하지 않음
        - CPU 자원 절약 가능

#### Busy Waiting 방식의 기존 코드

```C
// ticks만큼 일시 중지하는 함수
void timer_sleep(int64_t ticks)
{
	// 함수가 호출된 현재 시각을 'start' 변수에 저장
  int64_t start = timer_ticks();
	// 인터럽트가 활성화 상태인지 확인
  ASSERT(intr_get_level() == INTR_ON);
  // 'start' 시간부터 'ticks'만큼의 시간이 경과할 때까지 루프 반복
  while(timer_elapsed(start) < ticks)
	  // CPU 양보 코드
    thread_yield();
}
```

> ticks?
> - 일정 시간 간격으로 발생하는 시스템의 기본적인 단위
> - 100번 틱 : 초당 100번의 틱이 발생

#### Alarm-Clock 방식의 수정 코드

```C
// 대략 ticks 동안 실행을 일시 중단
void timer_sleep (int64_t ticks) {
  int64_t start = timer_ticks();
  ASSERT(intr_get_level() == INTR_ON);

	if(timer_elapsed(start) < ticks) {
		// 깨어날 목표 시각을 계산 = start + ticks
		thread_sleep(start + ticks);
	}
}
```

#### My QnA

- 왜 반복문이 아닌 조건문인가?
    - 반복문은
        - 스레드가 깨어있는 상태로 시간을 계속 확인하는 "대기 매커니즘"
    - 조건문은
        - 스레드를 재우기 전에 "사전 조건 검사 매커니즘"
        - 여기에서 사전 조건은, `timer_elapsed(start)` 즉, "현재시각 - `start`"
