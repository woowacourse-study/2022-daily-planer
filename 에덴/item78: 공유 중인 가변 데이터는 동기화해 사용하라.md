### 우리가 아는 동기화

- 배타적 실행

```java
public class Shared {

    public static void main(String[] args) {
        ShareThread shareTread = new ShareThread();
        Thread thread1 = new Thread(() -> {
            shareTread.setValue(100, 1500);
        });
        
        Thread thread2 = new Thread(() -> {
            shareTread.setValue(10, 1000);
        });
        thread1.setName("스레드 1");
        thread2.setName("스레드 2");
        thread1.start();
        thread2.start();
    }
}

class ShareThread {

    private int sharedValue = 0;

    public void setValue(int value, int millis) {
        this.sharedValue = value;
        try {
            Thread.sleep(millis);
        } catch (Exception e) {
        }
        System.out.println(Thread.currentThread().getName() + "의 Value 값은 " + this.sharedValue + "입니다.");
    }
}

// 스레드 2의 Value 값은 10입니다.
// 스레드 1의 Value 값은 10입니다.
```

> public `synchronized` void setValue(int value, int millis) {}
>
> 출력 결과
>
> 스레드 1의 Value 값은 100입니다.
>
> 스레드 2의 Value 값은 10입니다.

많은 프로그래머가 동기화를 배타적 실행, 즉 한 스레드가 변경하는 중이라서 상태가 일관되지 않은 순간의 객체를 다른 스레드가 보지 못하게 막는 용도로만 생각한다.

한 객체가 일관된 상태를 가지고 생성되고, 이 객체에 접근하는 메서드는 그 객체에 락(lock)을 건다.

락을 건 메서드는 객체의 상태를 확인하고 필요하면 수정한다.

즉, 객체를 하나의 일관된 상태에서 다른 일관된 상태로 변화시킨다.

동기화를 제대로 사용하면 어떤 메서드도 이 객체의 상태가 일관되지 않은 순간을 볼 수 없을 것이다.

### 동기화의 또 다른 기능

- 스레드 사이 안정적 통신

여기에 중요한 기능이 한가지 더 있다.

동기화 없이는 한 스레드가 만든 변화를 다른 스레드에서 확인하지 못할 수도 있다.

동기화는 일관성이 깨진 상태를 볼 수 없게 하는 것은 물론, 등기화된 메서드나 블록에 들어간 스레드가 같은 락의 보호하에 수행된 모든 이전 수정의 최종 결과를 보게 해준다.

동기화는 배타적 실행뿐 아니라 스레드 사이의 안정적인 통신에 꼭 필요하다.

```java
public class StopThread {

    private static boolean stopRequested;

    public static void main(String[] args) throws InterruptedException {
        Thread backgroundThread = new Thread(() -> {
            int i = 0;
            while (!stopRequested) {
                i++;
            }
        });
        
        backgroundThread.start();
        TimeUnit.SECONDS.sleep(1);
        stopRequested = true;
    }
}

```

-> 종료되지 않는다.

동기화가 없다면 VM 에서는 다음과 같이 최적화할 수 있다(hoisting).

```
while (!stopRequested) {
    i++
    }
    
if (!stopRequested) {
    while (true) {
        i++
        }
    }
```

```java
public class StopThread {

    private static boolean stopRequested;
    
    private static synchronized void requestStop() {
        stopRequested = true;
    }
    
    private static synchronized boolean stopRequested() {
        return stopRequested;
    }

    public static void main(String[] args) throws InterruptedException {
        Thread backgroundThread = new Thread(() -> {
            int i = 0;
            while (!stopRequested) {
                i++;
            }
        });
        
        backgroundThread.start();
        TimeUnit.SECONDS.sleep(1);
        stopRequested = true;
    }
}

```

쓰기 메서드와 읽기 메서드 모두 동기화 해야 한다.

> 가장 최근의 값을 읽는 volatile

동기화하는 또다른 방안으로, stopRequested 필드를 volatile 로 선언하면 동기화 생략이 가능하다.

volatile 한정자는 배타적 수행과는 상관 없지만 항상 가장 최근에 기록된 값을 읽게됨을 보장한다.

volatile 필드를 사용하여 스레드가 정상 종료된다.

```java

public class MainVolatile {
    
    private static volatile boolean stopRequested;
    
    public static void main(String[] args) throws InterruptedException {
        Thread backgroundThread = new Thread(() -> {
            int i = 0;

            while (!stopRequested) {
                i++;
            }
        });
        
        backgroundThread.start();
        stopRequested = true;
    }
}
```

이 키워드를 적용한 변수는 L1, L2등에 캐시를 참고하지 않고 직접 메모리를 참조하도록 한다.

![volatile](./images/volatile.png)

하지만, volatile 필드 사용에도 주의해야할 사항이 있다.

```java

public class MainVolatile {
    
    private static volatile int nextSerialNumber = 0;
    
    public static int generateSerialNumber() {
        return nextSerialNumber++;
    }
}
// Non-atomic operation on volatile field 'nextSerialNumber' 
```

이 메서드는 매번 고유한 값을 반환할 의도로 만들어졌다.

이 메서드의 상태는 nextSerialNumber 라는 단 하나의 필드로 결정되는데, 원자적으로 접근할 수 있고 어떤 값이든 허용한다.

따라서 굳이 동기화되지 않더라도 불변식을 보호할 수 있어보인다.

하지만 동기화 없이는 올바로 동작하지 않는 코드다.

> nextSerialNumber++;

증가 연산자(++)을 사용하여 코드상으로는 하나지만 실제로는 nextSerialNumber 필드에 두번 접근한다.

먼저 값을 읽고, 그런 다음 1이 증가한 새로운 값을 저장한다.

만약 두번째 스레드가 이 두 접근 사이를 비집고 들어와 값을 읽어가면 첫번째 스레드와 똑같은 값을 돌려받게 된다.

> 해결방안1

```java

public class MainVolatile {
    
    private static int nextSerialNumber = 0;    

    public synchronized static int generateSerialNumber() {
        return nextSerialNumber++;
    }
}
```

synchronized를 메서드에 붙였다면 nextSerialNumber 필드에 volatile을 제거해야한다.

이 메서드를 더 견고하게 하려면 int 대신 long을 사용하거나 nextSeraialNumber 가 최댓값에 도달하면 예외를 던지게해야 한다.

> 해결방안2

AtomicLong 타입으로 필드를 선언

volatile이 동기화의 두 효과 중 통신쪽에만 지원하지만 이 패키지는 원자성(배타적 실행)까지 지원

우리가 generateSerialNumber에 원하는 바로 그 기능이다. 성능도 동기화 버전보다 더 우수

```java

public class MainVolatile {
    
    private static final AtomLong nextSerialNumber = new AtomicLong();    

    public static long generateSerialNumber() {
        return nextSerialNumber.getAndIncrement();
    }
}
```

정리
이번 아이템에서 언급한 문제들을 피하기 가장 좋은 방법은 애초에 가변 데이터를 공유하지 않는것이다.

불변 데이터만 공유하거나 아무것도 공유하지 말자.

가변 데이터는 단일 스레드에서만 쓰도록 하자.
