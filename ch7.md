### 별 건 아니고

- inputstream을 입력 받아 업로드 하는 API에 대해 부하 테스트 하다가 서버가 메모리 부족으로 그냥 죽었음
- 다시 정리하고 켜보니, Heap Memory 사용률이 80% 가까이 되더라,, 강제로 작업 중단이 됐을테니, 자원이 제대로 정리되지 않아서 발생하는 것 같았음
그
- 물론 inputstream 으로 업로드 하는 방식은 heap memory를 쓰지 않고 buffer를 사용해서 읽기 때문에 heap memory에 영향을 주는 건 아닐 거라 생각함, 아마 부하 테스트 하면서 너무 많은 객체가 동시에 생성되어서 영향을 줬을 거라 생각함
- 그래서,, 가능한 객체를 재사용하자,, 라는 셀프 결론을 내림

# 내용 정리

- 자바는 실행 중인 스레드 강제 종료를 지원하지 않음
    - 강제 종료하면 공유 상태가 비정상적인 상태에 놓일 수 있음
        - ex) 공유 파일을 open 하고 작업하던 스레드를 강제 종료하면 close를 못할 수 있는 문제 발생
    - 인터럽트(interrupt)로 스레드를 멈춰달라고 요청해야함
- 설계할 때, 인터럽트를 받으면 작업을 모두 정리하고 종료하도록 해야함
    - 작업, 스레드가 스스로 멈출 수 있도록 구성해야함

## 작업 중단

- 취소 가능한 작업 → 외부에서 해당 작업을 임의 시점에 종료 상태에 이르도록 할 수 있는 작업
- 언제 필요한가?
    - 사용자 취소 요청: 취소 버튼 클릭 등
    - 시간 제한 작업: 제한 시간 이후에는 취소
    - 애플리케이션 이벤트: 원하는 결과를 위해 여러 작업을 실행 시키고 답을 구했다면 나머지 작업 취소
    - 오류
    - 종료
- 가장 기본적인 형태 취소 방식
    - 취소 요청 플래그를 설정하고 실행 중인 작업에서 그 플래그를 주기적으로 확인
- 취소 정책(cancellation policy)을 명확히 정의할 것
    - 어떻게 취소 요청을 보낼 수 있는가?
    - 내부에서 취소 요청을 언제 확인하는가?
    - 취소 요청이 들어오면 실행 중이던 작업은 어떻게 동작하는가?

### 인터럽트

- 스레드에 인터럽트를 거는 행위는 현재 실행 중인 작업을 멈추고 다른 일을 해야 한다고 신호를 보내는 것
    - 작업을 중단하고자 하는 부분에서 인터럽트를 사용하자.
- 스레드에 interrupt 메서드를 호출하는 건 아무것도 수행하지 않는다. 단지 해당 스레드에 인터럽트 요청이 있었다는 메시지를 전달할 뿐이다.
    - interrupt를 사용해 작업을 취소할 수 있도록 준비해야한다. 자바 표준에 맞는 인터럽트 관련 기능을 충분히 지원하자.
- 작업 취소 기능을 구현할 때 인터럽트가 가장 적절한 방법이라고 볼 수 있다.

### 인터럽트 정책

- 작업 별 작업 취소 정책이 있는 것 처럼 스레드 별 인터럽트 처리 정책이 있어야 한다.
- 인터럽트 처리 정책 → 인터럽트 요청 시, 스레드가 인터럽트 처리하는 지침
    - 스레드 수준이나 서비스 수준 작업 중단 기능 제공
        - 최대한 빠르게 중단
        - 사용한 자원 적절한 정리
        - 작업 중단 요청 스레드에게 작업 중단하고 있다는 사실을 알려줄 방법 제공

- 작업(task)과 스레드(thread)가 인터럽트 상황에 서로 어떻게 동작해야 하는지 명확히 구분할 것
- 작업은 스레드 풀과 같이 실행만 전담하는 스레드를 빌려서 사용하게 된다.
    - 따라서 작업은 스레드의 인터럽트 상태 그대로 유지
    - 스레드를 소유하는 프로그램이 인터럽트 상태에 직접 대응하도록 해야한다.

- 대부분의 블로킹 메소드가 InterruptedException을 던지도록 설계된 이유는
    - 실행 중 최대한 빨리 작업 중단
    - 호출한 스레드에 인터럽트 요청 넘기기
    - 스레드에서 인터럽트 대응해 추가적인 작업 할 수 있도록

- 작업은 실행하는 스레드에서 적용되고 있는 인터럽트 정책에 어떤 가정도 해선 안된다.
    - 실제로는 작업 중단으로 해석할 수 있고, 인터럽트 대응해 뭔가 작업 처리가 필요할 수 도 있다.
    - 따라서 작업에서 선택할 수 있는 방법은 인터럽트 상태를 유지하는 것이다.
        - InterruptedException을 던지거나
        - Thread.currentThread().interrupt() 호출해 스레드의 인터럽트 상태를 유지한다.
- 비판이 있긴 하지만 인터럽트에 대한 실제적 중단 시점을 개발자가 임의로 늦출 수 있도록 제공한다. 응답성과 안정성을 능동적으로 관리해야한다.

### 인터럽트에 대한 대응

- 블로킹 메소드를 호출한 후, InterruptedException이 발생할 때 처리할 수 있는 실질적 방법?
    - 예외를 상위 메소드로 전달 → 이 메소드 자체도 블로킹 메소드가 된다.
    - 호출 스택 상단에 위치한 메소드가 직접 처리할 수 있도록 언터럽트 상태 유지
- 중단 기능 지원 X + 블로킹 메소드 호출하는 작업은, 블로킹 메소드 기능을 재시도하도록 하는 것이 좋다.

```java
boolean interrupted = false;
try {
  while(true) {
    try {
      return queue.take(); // 블로킹 메소드
    } catch (InterruptedException e) {
      interrupted = true; // 그냥 넘어가고 재시도
    }
  }
} finally {
  if (interrupted) Thread.currentThread().interrupt(); // 인터럽트 상태 복구
}
```

- 진행 과정 곳곳에 현재 스레드 인터럽트 상태를 확인한다면 인터럽트에 대한 응답 속도를 높일 수 있다.

### Future를 사용해 작업 중단

- Future
    - 작업이 실행되는 추상화 모델
    - 예외 처리는 어떻게 하는지
    - 작업을 중단할 때는 어떻게 하는지 (cancel)
- 작업 구현할 때 인터럽트가 걸리면 작업 중단 요청으로 해석하고 행동하면 Future를 통해서 쉽게 작업 중단이 가능하다.

### 인터럽트에 응답하지 않는 블로킹 작업 다루기

- 표준을 따르지 않는 중단 방법
- 모든 블로킹 메소드가 인터럽트에 대응하도록 되어 있지는 않다.
    - 동기적 소켓 IO, 암묵적 락 확보 대기의 경우 인터럽트 상태 변수의 값을 설정하는 것 말고는 아무런 효과가 없다.
    - [java.io](http://java.io/) 동기적 소켓 IO
    - java.nio 동기적 IO
    - Selector 비동기적 IO
    - 락 확보
- 표준적이지 않은 방법으로 작업을 중단하는 기능을 클래스 속으로 감출 수 있다.

## 스레드 기반 서비스 중단

- 스레드 기반 서비스 → 스레드 풀과 같이 내부적으로 스레드 생성하는 서비스
- 애플리케이션을 깔끔하게 종료하기 위해서는
    - 스레드 기반 서비스 내부 스레드를 안전하게 종료시켜야함
    - 강제 종료는 안되기 대문에 스레드에게 종료 요청 해야함
- 스레드 소유한 객체만 해당 스레드에 인터럽트 걸거나 우선 순위 조정하는 작업을 하게 하자.
- 스레드 하나가 외부 특정 객체에 소유된다는 개념을 활용하자.
- 즉, 인터럽트를 걸어야 하는 상황이라면, 스레드를 소유한 스레드 풀에서 책임 져야 한다.
- 스레드 기반 서비스가 시작, 종료까지 모든 기능에 메소드를 직접 제공할 것

### 예시

- 종료 방법
    - 강제 종료 → 응답 빠르다. 작업이 중단되는 과정에서 문제 발생 여지가 있다.
    - 안전 종료 → 응답 느리다. 작업을 잃을 가능성이 없어 안전하다.
- 스레드 기반 서비스 구현 시 종료 방법을 선택할 수 있도록 준비하면 좋다.
- 스레드 소유한 클래스는 자신이 소유한 서비스, 스레드의 시작과 종료 기능을 관리한다.

### 독약(poison pill)

- 특정 객체를 큐에 쌓도록 하고, 이 객체를 받으면 종료하는 규칙을 설정
- 프로듀서 컨슈머 패턴에서 각 요소의 수를 정확히 알고 있을 때만 사용 가능

### 단번에 실행하는 서비스

- 일련의 작업을 순서대로 처리 + 작업이 모두 끝나기 전까지 리턴 되지 않는 메소드
- 메소드 내부에서만 사용한 Executor 인스턴스가 하나 있다면 쉽게 관리할 수 있다.

### shutdownNow 메소드의 약점

- 개별 작업 스스로 작업 진행 정보를 외부에 알려주기 전에는 서비스 종료시 실행 중이던 작업의 상태를 알 수 없다.
- 알 수 있으려면 개별 작업이 리턴될 때 자신을 실행했던 스레드의 인터럽트 상태를 유지해야 한다.

## 비정상적인 스레드 종료 상황 처리

- 오류 때문에 스레드가 멈춘 경우에도 전체 애플리케이션은 마치 오류 없이 계속해 동작하는 것처럼 보일 수도 있다.
    - 스레드에서 오류가 발생해 멈추지 않도록 예방
    - 멈춘 스레드를 찾아내는 방법
- 스레드를 예상치 못하게 종료시키는 가장 큰 원인 → RuntimeException
- 스레드 풀 의 경우에 등록되는 작업이 어떤 예외처리를 하는지 알 방법이 없기 때문에 방어적으로 구성되어야한다. 스레드가 비정상 종료하는 경우에 스레드 풀 객체가 이를 인지하고 적절한 처리를 해야한다.
    - 비정상 종료된 스레드가 있을 때, 스레드 풀이 종료된 스레드를 삭제하고 새로운 스레드를 생성해 작업을 이어나갈 수도 있고, 스레드 풀 자체가 종료되는 중이거나 이미 작업을 처리할 스레드가 만들어저 있는 상황에서는 스레드를 생성하지 않고 놔둘 수 있다.

### 정의되지 않은 예외 처리

- UncaughtExceptionHandler → 처리하지 못한 예외로 스레드가 종료되는 시점을 잡아서 처리
- 처리되지 못한 예외 때문에 스레드가 종료되면 JVM이 애플리케이션에서 정의한 UncaughtExceptionHandler 호출
- 기본 동작으로 스택 트레이스를 콘솔 System.err 스트림에 출력한다.
- 개별 Thread.setUncaughtEceptionHanlder를 사용할 수도 있고, setDefaultUncaughtExceptionHandler를 사용해 기본 동작 지정 가능
- 예외 시 로그 파일에 오류 출력하는 간단한 기능이라도 확보해야 하므로, 모든 스레드를 대상으로 UncaughtExceptionHandler를 활용해야 한다.
- 스레드 소유자만 UncaughtExceptionHandler 를 설정하는 게 바람직하므로, 스레드 풀에 ThreadFacotry를 함께 건낼 수 있다.
- 작업이 중단되는 경우 오류 발생 사실을 즉시 알고자 한다면
    - Runnable, Callable의 run 메소드에서 try-catch 해서 처리
    - ThreadPoolExecutor 클래스에 afterExecute 메소드 오버라이드
- Future를 받아오는 작업의 경우 예외는 Future.get을 통해서 넘어온다.

## JVM 종료

- 절차대로 종료되는 경우
    - 일반 스레드가 모두 종료되는 시점 (데몬 스레드 X)
    - System.exit 호출하는 경우
    - 기타 여러가지 상황(SIGINT 시그널 전달 받는 경우 등)
    - Runtime.halt 메소드 호출로 운영체제 수준에서 강제 종료하는 방법

### 종료 훅

- Runtime.addShutdownHook을 통해 등록
- 예정된 절차대로 종료되는 경우 JVM이 종료 훅을 실행시킨다.
    - 실행 순서 보장 X →  의존성 있는 종료 훅을 여러 개 등록하지 말 것. 차라리 하나의 훅으로 묶어서 등록하자.
- 훅을 완수한 다음 finalize 메소드 호출한다.
- 만약 종료 훅이나 finalize가 계속 실행된다면 종료 절차가 멈춘다. JVM은 강제 대기 상태가 된다.
- 종료 훅은 서비스, 애플리케이션 자체 정리 목적으로 사용하기 좋다.
- 서비스별로 각자 종료 훅을 만들어 등록하기 보다는 모든 서비스를 정리할 하나의 훅을 사용해 서비스간 의존성에 맞춰 순서대로 정리하는 방법도 있다.
    - 이렇게 하면 올바른 순서대로 서비스를 종료하고 마무리할 수 있다.

### 데몬 스레드

- 부수적인 기능 처리 원할 때 + 해당 스레드가 JVM 종료에 영향을 주지 않고 싶을 때 → 데몬 스레드 (daemon thread)
- 스레드 하나가 종료되면 JVM은 남아있는 모든 스레드 가운데 일반 스레드가 있는지 확인한다. 일반 스레드가 모두 종료 되었고 남아있는 스레드가 모두 데몬 스레드라면 즉시 JVM 종료 절차를 밟는다. → 즉, JVM은 데몬 스레드 종료를 기다리지 않음
- JVM이 종료될 때 모든 데몬 스레드는 버려진다.
- 데몬 스레드는 보통 부수적인 용도로 사용한다. 데몬 스레드는 종료할 때 동작 수행을 보장하지 않는다.
    - 가비지 컬렉터
    - 메모리 내부에 관리하고 있는 캐시 기한 만료된 항목을 주기적으로 제거하는 등
- 데몬 스레드는 예고 없이 종료될 수 있다. 따라서 애플리케이션 내부에서 시작시키고 종료시키며 사용하기에는 그다지 좋은 방법이 아니다.

### finalize 메소드

- 실행 여부, 언제 실행될지에 대한 보장 X
- 자원 정리를 위해서라면 try-finally 와 같은 구문을 활용할 것
- finalize 메소드는 사용하지 마라.

## 요약

- 자바에서는 선점적으로 작업 중단, 스레드 종료 기능 제공 X
- 인터럽트를 통해서 스레드간 협력을 통해 작업 중단 기능 구현
- 작업 중단 기능 구현 및 프로그램에 일관적으로 적용하는 일은 개발자의 몫
- FutureTask, Executor 등 프레임웍을 사용해 작업, 서비스 중단 기능 쉽게 구현할 수 있다.