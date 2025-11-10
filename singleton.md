# singleton

## singleton이란?
- 단 하나의 유일한 객체를 만들기 위한 패턴
- 메모리 절약을 위해 새로운 인스턴스가 아닌 기존 인스턴스를 가져와서 활용하는 기법
- ex) 데이베이스 연결 모듈, 디스크 IO, DBCP 등

## 구현 원리
- 생성 시 private을 붙여줌
- 내부에서는 instance가 null 일 때 새로운 객체를, null이 아니면 기존 객체(getInstance())를 가져와서 return

## 패턴 구현 기법
- Eager Initialization
- Static block initialization
- Lazy initialization
- Thread safe initialization
- Double-Checked Locking
- Bill Pugh Solution
- Enum 이용

### Eager Initialization
- 장점
    - 한번만 미리 만들어둠(직관적, 심플)
    - static final 사용(멀티쓰레드 안전)
- 단점
    - 객체 미사용 시에도 메모리에 떠있음(객체가 크면 자원 낭비율 더 커짐)
    - 예외 처리 불가

```java
class Singleton {
    // 싱글톤 클래스 객체를 담을 인스턴스 변수
    private static final Singleton INSTANCE = new Singleton();

    // 생성자를 private로 선언 (외부에서 new 사용 X)
    private Singleton() {}

    public static Singleton getInstance() {
        return INSTANCE;
    }
}
```

### Static block initialization
- 장점
    - 예외 처리 가능
- 단점
    - 여전히 static 성질을 가지고 있음

```java
class Singleton {
    // 싱글톤 클래스 객체를 담을 인스턴스 변수
    private static Singleton instance;

    // 생성자를 private로 선언 (외부에서 new 사용 X)
    private Singleton() {}
    
    // static 블록을 이용해 예외 처리
    static {
        try {
            instance = new Singleton();
        } catch (Exception e) {
            throw new RuntimeException("싱글톤 객체 생성 오류");
        }
    }

    public static Singleton getInstance() {
        return instance;
    }
}
```

### Lazy initialization
- 장점
    - null 유무로 초기화 결정
    - 미사용 고정 메모리 해결(getInstance 호출 시 그때서 초기화 진행)
- 단점
    - thread safe 하지 않음(스레드 1,2가 있다 할 때 스레드1이 if문을 타서 아직 singleton 인스턴스 생성 전이고 스레드 2가 if문에 진입하면 이 역시 아직은 생성이 안되어있길래 새로운 인스턴스를 생성할 수도 있음. 즉 스레드1이 인스턴스를 생성하지 않은 상태라면 스레드2 진입 시 싱글톤이 안 됨)

```java
class Singleton {
    // 싱글톤 클래스 객체를 담을 인스턴스 변수
    private static Singleton instance;

    // 생성자를 private로 선언 (외부에서 new 사용 X)
    private Singleton() {}
	
    // 외부에서 정적 메서드를 호출하면 그제서야 초기화 진행 (lazy)
    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton(); // 오직 1개의 객체만 생성
        }
        return instance;
    }
}
```

아래와 같은 경우에 싱글톤이지만 2개의 객체가 되어버릴 수도 있음
```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

// 싱글톤 객체
class Singleton {
    private static Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton(); // 오직 1개의 객체만 생성
        }
        return instance;
    }
}

public class Main {
    public static void main(String[] args) {
        // 1. 싱글톤 객체를 담을 배열
        Singleton[] singleton = new Singleton[10];

        // 2. 스레드 풀 생성
        ExecutorService service = Executors.newCachedThreadPool();

        // 3. 반복문을 통해 10개의 스레드가 동시에 인스턴스 생성
        for (int i = 0; i < 10; i++) {
            final int num = i;
            service.submit(() -> {
                singleton[num] = Singleton.getInstance();
            });
        }

        // 4. 종료
        service.shutdown();
		
        // 5. 싱글톤 객체 주소 출력
        for(Singleton s : singleton) {
            System.out.println(s.toString());
        }
    }
}
```

### Thread safe initialization
- 장점
    - synchronized를 통한 동기화 처리(thread safe)
- 단점
    - 장점으로 인한 overhead 발생으로 성능 하락 발생

```java
class Singleton {
    private static Singleton instance;

    private Singleton() {}

    // synchronized 메서드
    public static synchronized Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

### Double-Checked Locking
- 장점
    - 매번 synchronized으로 잠기는게 문제니까 최초 초기화시에만 적용, 이후엔 비적용으로 해결
- 단점
    - 인스턴스에 volatile 키워드를 붙여야 I/O 불일치 문제 해결
    - volatile을 사용하기 위해선 JVM 1.5 이상, 높은 JVM 이해도, JVM에 따라서도 스레드 safe 하지 않는 경우가 있기에 **사용을 지양**

#### volatile 란? 
- java는 멀티스레드인 경우 변수를 메인 메모리가 아닌 캐시 메모리에서 가져와 사용
- 비동기로 캐시 값을 저장하면서 쓰다보니 스레드들간 캐시 불일치 문제가 발생
- 이 키워드를 통해서 캐시가 아닌 메인 메모리에서 읽도록 지정

```java
class Singleton {
    private static volatile Singleton instance; // volatile 키워드 적용

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
        	// 메서드에 동기화 거는게 아닌, Singleton 클래스 자체를 동기화 걸어버림
            synchronized (Singleton.class) { 
                if(instance == null) { 
                    instance = new Singleton(); // 최초 초기화만 동기화 작업이 일어나서 리소스 낭비를 최소화
                }
            }
        }
        return instance; // 최초 초기화가 되면 앞으로 생성된 인스턴스만 반환
    }
}
```

### Bill Pugh Solution (LazyHolder)
- 장점
    - 멀티스레드 안전
    - Lazy Loading
    - 클래스 안에 내부 클래스(holder)를 두어 JVM의 클래스 로더 매커니즘과 클래스가 로드되는 시점을 이용한 방법 (스레드 세이프)
    - static 메소드에서는 static 멤버만을 호출할 수 있기 때문에 내부 클래스를 static으로 설정이밖에도 내부 클래스의 치명적인 문제점인 메모리 누수 문제를 해결하기 위하여 내부 클래스를 static으로 설정
- 단점
    - 다만 클라이언트가 임의로 싱글톤을 파괴할 수 있다는 단점을 지님 (Reflection API, 직렬화/역직렬화를 통해)

```java
class Singleton {

    private Singleton() {}

    // static 내부 클래스를 이용
    // Holder로 만들어, 클래스가 메모리에 로드되지 않고 getInstance 메서드가 호출되어야 로드됨
    private static class SingleInstanceHolder {
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return SingleInstanceHolder.INSTANCE;
    }
}
```