# 2장 스레드 안전성

스레드와 락은 목적을 위한 도구 일뿐 스레드에 안전한 코드를 작성하는 것은 근본적으로는 상태, 특히 공유되고 변경할 수 있는 상태에 대한 접근을 관리하는 것이다. 공유됐다는 것은 여러 스레드가 특정 변수에 접근할 수 있다는 뜻이고, 변경할 수 있다mutable는 것은 해당 변수 값이 변경 될 수 있다는 뜻이다. 스레드 안전성은 코드를 보호하는것이 아닌, 데이터에 제어 없이 동시 접근하는 것을 막으려는 것이다.

객체가 스레드에 안전한지의 여부는 해당 객체에 여러 스레드가 접근할지의 여부에 달렸다. 즉, 객체가 뭘 하는지가 아닌, 프로그램에서 객체가 어떻게 사용되는가의 문제이다. 스레드가 하나 이상 상태 변수에 접근하고 그 중 하나라도 변수에 값을 쓰면, 해당 변수에 접근할 때 관련된 모든 스레드가 동기화를 통해 조율해야 한다. (synchronized, volatile, 명시적 락, 단일 연산 변수atomic variable emd)

<br>

⚠️ 잘못된 프로그램을 고치는 세 가지 방법
+ 해당 상태 변수를 스레드 간에 공유하지 않는다
+ 해당 상태 변수를 변경할 수 없도록 만든다
+ 해당 상태 변수에 접근할 땐 언제나 동기화를 사용한다

<br>스레드 안전한 클래스를 설계할 땐, 바람직한 객체 지향 기법이 가장 좋다. 캡슐화와 불변 객체를 잘 활용하고, 불변 조건을 명확하게 기술해야 한다.

때론 바람직한 객체 지향 설계 기법이 요구사항과 배치되는 경우도 있는데, 이런 경우 성능이나 기존 코드와의 호완성을 위해 설계 원칙을 양보해야 한다. 또는 추상화와 캡슐화 기법이 성능과 배치되는 경우도 있는데, 이런 경우는 코드를 올바르게 작성하는 일이 항상 먼저이고, 그 다음 필요한 만큼 성능을 개선해야 한다. 최적화는 실제와 동일한 환경에서 성능 측정을 해본 이후에 요구 사항에 미달될 때만 하는 편이 좋다.

<br>

## 2.1 스레드 안전성이란?

  스레드에 대한 정의의 핵심은 모두 **정확성**correctness 개념과 관계 있다. 정확성이란 클래스가 해당 클래스의 명세에 부합한다는 뜻이다. 잘 작성된 클래스 명세는 객체 상태를 제약하는 불변조건invariants과 연산 수행 후 효과를 기술하는 후조건postcondition을 정의한다. 여러 스레드가 클래스에 접근할 때 계속 정확하게 동작하면 해당 클래스는 스레드 안전하다.

### 2.1.1 예제: 상태 없는 서블릿

서블릿 기반 인수 분해 서비스 만들기

```java
public class StatelessFactorizer implements Servlet {
	public void service(ServletRequest req, ServletResponse resp) {
		BigInteger i = extractFromRequest(req);
		BigInteger[] factors = factor(i);
		encodeIntoResponse(resp, factors);
	}
}
```

   예제의 경우 선언한 변수가 없고, 다른 클래스의 변수를 참조하지도 않는다. 지역변수에만 저장하고, 해당 스레드 에서만 접근할 수 있다. 객체에 접근하는 스레드가 어떤 일을 하든 다른 스레드가 수행하는 동작의 정확성에 영향을 끼칠 수 없기 떄문에 **상태 없는 객체**는 항상 **스레드 안전**하다
   <br>
   <br>

## 2.2 단일 연산

  상태 없는 개체에 상태를 하나 추가해보자. 처리한 요청의 수를 기록하는 ‘접속 카운터’를 long 필드에 추가하고 각 요청마다 값을 증가시켜보자.

```java
@NotThreadSafe
public class UnsafeCountingFactorizer implements Servlet {
	private long count = 0;
	public long getCount() { return count; }
	
	public void service(ServletRequest req, ServletResponse resp) {
		BigInteger i = extractFromRequest(req);
		BigInteger[] factors = factor(i);
		**++count;**
		encodeIntoResponse(resp, factors);
	}
}
```

   단일 스레드 환경에서는 잘 동작하겠지만 스레드에 안전하지 않다. 실제로 단일 연산이 아닌(값 읽고, 1을 더하고, 값을 저장) 증가 연산자 때문에 원하는 결과가 도출되지 않는다. 병렬 프로그램의 입장에서 타이밍이 안 좋을 때 결과가 잘못될 가능성은 굉장히 중요한 개념이기 때문에 **경쟁 조건**race condition이라는 별도 용어로 정의한다.
   <br>
   <br>

### 2.2.1 경쟁 조건

   경쟁 조건은 상대적인 시점이나 JVM이 여러 스레드를 교차해서 실행하는 상황에 따라 계산의 정확성이 달라질 때 나타나며, 타이밍이 딱 맞았을 때만 정답을 얻는 경우를 말한다. 일반적인 경쟁 조건 형태는 잠재적으로 유효하지 않은 값을 참조해서 다음에 뭘 할지를 결정하는 **점검 후 행동** check-then-act 형태의 구문이다.

   대부분의 경쟁 조건은 관찰 결과의 무효화로 특징 지어진다. 즉, 잠재적으로 유효하지 않은 관찰 결과로 결정을 내리거나 계산을 하는 것이다. 이런 류의 경쟁 조건을 점검 후 행동이라고 한다. 어떤 사실을 확인하고, 그 관찰에 기반해 행동을 하지만 해당 관찰은 관찰한 시각과 행동한 시각 사이에 더 이상 유효하지 않게 됐을 수도 있다.

### 2.2.2 예제: 늦은 초기화 시 경쟁 조건

  점검 후 행동하는 흔한 프로그래밍 패턴으로 **늦은 초기화**lazy initialization이 있다. 늦은 초기화는 특정 객체가 실제 필요할 때까지 초기화를 미루고 동시에 단 한 번만 초기화 되도록 하기 위한 것이다. 다음은 늦은 초기화 패턴을 구현한 예제이다.

```java
@NotThreadSafe
public class LazyInitRace {
	private ExpensiveObject instance = null;
	public ExpensiveObject getInstance() {
		if (instance == null) 
			instance = new ExpensiveObject();
		return instance;
	}
}
```

   이러한 늦은 초기화는 경쟁 조건 때문에 제대로 동작하지 않을 가능성이 있다. 두 개의 스레드가 예측하기 어려운 타이밍에 동시 혹은 읽고 판단하는 사이에 동작하여 서로 다른 인스턴스를 가져가게 되는 결과가 나올수도 있다. 

  예제 2.2의 접속 횟수를 세는 부분은 또 다른 종류의 경쟁 조건을 가지고 있다. 카운트를 증가시키느 작업은 **읽고 수정하고 쓰기**read-modify-write 동작이며 이는 이전 상태를 기준으로 객체의 상태를 변경하는 것이다. 이전 값을 알아내고 카운터를 갱신하는 동안 다른 스레드에서 그 값을 변경하거나 사용하지 않도록 해야 한다. 

  대부분 병렬 처리 오류가 그렇듯, 경쟁 조건 떄문에 프로그램에 오류가 항상 발생하는것은 아니며, 운 나쁘게 타이밍 이슈로 인해 발생한다. 하지만 경쟁 조건은 그 자체로 심각한 문제를 일으킬 수 있는 경우도 있으니 조심해야 한다 (ex: 데이터 저장 공간을 생성할 때 LazyInitRace를 사용하여 다른 저장 공간이 생성되어 일관성이 사라짐, 객체 식별자 생성시 완전히 별개의 두개의 객체가 같은 식별자를 가지게 되는 객체 유일성 제약 깨짐)

### 2.2.3 복합 동작

  경쟁 조건을 피하려면 변수가 수정되는 동안 다른 스레드가 해당 변수를 사용하지 못하도록 막을 방법이 있어야 하며, 이런 방법으로 보호해두면 특정 스레드에서 변수를 수정할 때 다른 스레드는 수정 도중이 아닌 수정 이전이나 이후에만 상태를 읽거나 변경을 가할 수 있다.


    💡 단일 연산 작업은 자신을 포함해 같은 상태를 다루는 모든 작업이 **단일 연산**인 작업을 지칭한다. 작업 A를 실행 중인 스레드 관점에서 다른 스레드가 B를 실행할 때 작업 B가 모두 수행됐거나 또는 전혀 수행되지 않은 두가지 상태로만 파악된다면 작업 A의 눈으로 볼 때 작업B는 단일 연산이다.

  점검 후 행동과 읽고 수정하고 쓰기 같은 일련의 동작을 **복합 동작**compound action이라고 하며, 스레드에 안전하기 위해서 전체가 단일 연산으로 실행돼야 하는 일련의 동작을 말한다. 

다음 예제는 예제 2.2의 문제를 스레드 안전한 기존 클래스를 이용해 고쳐본 예제이다.

```java
@ThreadSafe
public class CountingFactorizer implements Servlet {
	private final AtomicLing count = new AtomicLong(0); // 단일 연산 변수 적용
	public long getCount() { return count.get(); }
	
	public void service(ServletRequest req, ServletResponse resp) {
		BigInteger i = extractFromRequest(req);
		BigInteger[] factors = factor(i);
		**count.incrementAndGet();**
		encodeIntoResponse(resp, factors);
	}
}
```

  숫자나 객체 참조 값에 대해 상태를 단일 연산으로 변경할 수 있도록 단일 연산 변수를 사용하였다. 서블릿 상태가 카운터의 상태이고 카운터가 스레드에 안전하기 때문에 서블릿도 스레드에 안전하다. <br><br>

## 2.3 락

  서블릿 상태 전부를 스레드 안전한 객체를 써서 관리함으로써 스레드 안전성을 유지하면서도 예제 서블릿에 상태 변수를 하나 추가할 수 있었다. 하지만 더 많은 상태를 추가할 때도 스레드 안전한 상태 변수를 추가하기만 하면 충분할까?

  서로 다른 클라이언트가 연이어 같은 숫자를 인수분해하길 원하는 경우를 고려해서, 가장 최근 계산 결과를 캐시에 보관해 인수분해 예제 서블릿의 성능을 향상시켜보자. 이 전략에선 서블릿은 가장 마지막으로 인수분해하기 위해 입력한 수와, 그 입력값을 인수분해한 결과 값을 기억하고 있어야 한다. 예제 2.4에서 처럼 AtomicLong과 비슷한 AtomicReference 클래스를 사용한다면?

```java
@NotThreadSafe
public class UnsafeCachingFactorizer implements Servlet {
	private final AtomicReference<BigInteger> lastNumber = new AtomicReference<BigInteger>();
	private final AtomicReference<BigInteger> lastFactors = new AtomicReference<BigInteger>();

	public void service(ServletRequest req, ServletResponse resp) {
		BigInteger i = extractFromRequest(req);
		if (i.equals(lastNumber.get()))
			encodeIntoResponse(resp, lastFactors.get());
		else {
			BigInteger[] factors = factor(i);
			lastNumber.set(i);
			lastFactors.set(factors);
			encodeInfoResponse(resp, factors);
		}
	}
}
```

  예제에서의 단일 연산 참조 변수 각각은 스레드에 안전하지만 틀린 결과를 낼 수 있는 경쟁 조건을 갖고 있다. 여러 개의 변수가 하나의 불변 조건을 구성하고 있다면, 이 변수들은 서로 독립적이지 않다. 즉, 한 변수의 값이 다른 변수에 들어갈 수 있는 값을 제한 할 수 있다. 따라서 상태를 일관성있게 유지하려면 변수 하나를 갱신할 땐, 다른 변수도 동일한 단일 연산 작업 내에서 함께 변경해야 한다.

### 2.3.1 암묵적인 락

  자바에서는 단일 연산 특성을 보장하기 위해 synchronized라는 구문으로 사용할 수 있는 락을 제공한다. synchronized 구문은 락으로 사용될 객체의 참조 값과, 해당 락으로 보호하려면 코드 블록으로 구성된다. 

  모든 자바 객체는 락으로 사용할 수 있으며, 자바에 내장된 락을 암묵적인 락intrinsic lock 혹은 모니터 락monitor lock이라고 한다. 락은 스레드가 synchronized 블록에 들어가기 전에 자동으로 확보되며 블록을 벗어날 때 자동으로 해제된다. 

  자바에서 암묵적인 락은 뮤텍스mutexs로 동작하며, 즉 한 번에 한 스레드만 특정 락을 소유할 수 있다. 그렇기 때문에 같은 락으로 보호되는 synchronized 서로 다른 블록 역시 서로 단일 연산으로 실행된다. 그러므로 한 스레드가 synchronized 블록을 실행 중이라면 같은 락으로 보호되는 synchronized 블록에 다른 스레드가 들어와 있을 수 없다. 

```java
@ThreadSafe
public class UnsafeCachingFactorizer implements Servlet {
	@GuardedBy("this") private BigInteger lastNumber;
	@GuardedBy("this") private BigInteger[] lastFactors;

	public synchronized void service(ServletRequest req, ServletResponse resp) {
		... 예제 2.5와 동일 ...
	}
}
```

  service 메소드에 synchronized 키워드를 추가해 한 번에 한 스레드만 실행할 수 있게 했으나, 극단적인 해결법이라 인수분해 서블릿을 여러 클라이언트가 동시에 사용할 수 없고, 응답성이 엄청 떨어지게 된다.

### 2.3.2 재진입성

  스레드가 다른 스레드가 가진 락을 요청하면 해당 스레드는 대기 상태에 들어간다. 하지만 암묵적인 락은 재진입reentrant 가능하기 때문에 특정 스레드가 자기가 이미 획득한 락을 다시 확보할 수 있다. 재진입성은 확보 요청 단위가 아닌 스레드 단위로 락을 얻는다는것을 의미한다. 

   재진입 가능한 락이 없으면 하위 클래스에서 synchronized 메소드를 재정의하고 상위 클래스의 메소드를 호출하는 다음 예제와 같은 코드도 데드락에 빠질것이다. 재진입성은 이런 문제를 해결해준다.

```java
public class Widget {
	public synchronized void doSomething() {....}

	public class LoggingWidget extends Widget {
		public synchronized void doSomething() {
			super.doSomething();
		}
	}
}
```

  

   synchronized는 synchronized(this) 블록과 동일한 의미를 가지므로, 두개의 메소드는 같은 모니터 객체(’this’)에 대한 동기화를 수행한다. 한 스레드가 모니터 객체의 락을 획득하면, 같은 모니터 객체에 대해 선언된 모든 synchronized 블록으로의 진입권을 갖는다. <br><br>

## 2.4 락으로 상태 보호하기

  락은 공유된 상태에 배타적으로 접근할 수 있도록 보장하는 규칙을 만들때 유용하며 항상 **일관적인 상태**를 유지할 수 있도록 한다. 락을 확보한 상태로 복합 동작을 실행하면 해당 복합 동작을 단일 연산으로 보이게 할 수 있으나, 특정 변수에 대한 접근을 조율하기 위해 동기화 할때는 해당 변수에 접근하는 모든 부분을 동기화해야한다.

    특정 객체의 락을 얻는다고 해도 다른 스레드가 해당 객체에 접근하는 걸 막을 순 없으며, 다른 스레드가 동일한 락을 얻지 못하게 할 수 있을 뿐이다. 락 규칙이나 동기화 정책을 만들고 프로그램 내에 규칙과 정책을 일관성 있게 따르는건 순전히 개발자에게 달렸다.

  락의 일반적인 사용 예는 모든 변경 가능한 변수를 객체 안에 캡슐화 하고, 해당 객체의 암묵적인 락을 사용해 캡슐화한 변수에 접근하는 모든 코드 경로를 동기화함으로써 여러 스레드가 동시 접근하는 상태에서 내부 변수를 보호하는 방법이다. 

  모든 데이터를 락으로 보호해야 하는 건 아니고, 변경 가능한 데이터를 여러 스레드에서 접근해 사용하는 경우에만 해당한다. 특정 변수가 락으로 보호되면, 한 번에 한 스레드만 해당 변수에 접근할 수 있다는 점을 보장할 수 있다. 또한 클래스에 여러 상태 변수에 대한 불변 조건이 있으면 각 변수는 같은 락으로 보호돼야 한다. 
  <br><br>

## 2.5 활동성과 성능

  예제 2.6에서 인수 분해 서블릿에 캐시 기능을 추가하면서 service 메소드 전체를 동기화 했다. service 메소드는 락을 확보하고 service 실행 후 락 해제를 하는 과정을 한 번에 한 스레드만 실행 할 수 있게 되었고 여러 요청을 동시에 처리 할 수 없게 되었다. 

  synchronized 블록의 범위를 줄이면 동시성을 향상시킬 수 있으나 단일 연산으로 처리해야 하는 작업을 너무 여러 개의 synchronized 블록으로 나누는것은 지양해야 한다. 

```java
@ThreadSafe
public class CachedFactorizer implements Servlet {
	@GuaredBy("this") private BigInteger lastNumber;
	@GuaredBy("this") private BigInteger[] lastFactors;
	@GuaredBy("this") private long hits;
	@GuaredBy("this") private long cacheHits;

	public synchronized long getHits() { return hits; }
	public synchronized double getCacheHitRatio() {
		return (double) cacheHits / (double) hits;
	}

	public void service(ServletRequest req, ServletResponse resp) {
		BigInteger i = extractFromRequest(req);
		BigInteger[] factors = null;
		synchronized(this) {
			++hits;
			if(i.equals(lastNumber)) {
				++cacheHits;
				factors = lastFactors.clone();
			}
		}
		if(factors == null) {
			factors = factor(i);
			synchronized(this) {
				lastNumber = i;
				lastFactors = factors.clone();
			}
		}
		encodeInfoResponse(resp, factors);
	}
}
```

   위의 예제에서 전체 메소드를 동기화하는 대신 두개의 짧은 코드 블록을 synchronized 키워드로 보호했다. 하나는 캐시된 결과를 갖고 있는지 검사하는 일종의 확인 후 동작check-and-act 이고, 또 하나는 캐시된 입력 값과 결과를 새로운 값으로 변경하는 부분이다. 동기화 블록 밖에 있는 코드는 다른 스레드와 공유되지 않는 지역 변수만 사용하기 때문에 동기화가 필요없다.

   단일 연산 변수 대신 long 필드를 썼는데, 이미 동기화 블록을 사용하고 있기 때문에 서로 다른 두가지 동기화 수단을 사용해서 혼동을 줄 필요가 없기 때문이다. 

  synchronized 블록의 크기를 적정하게 유지하려면 안전성, 단순성, 성능 간의 적절한 타협이 필요하다. 예제 2.8와 같이 적절한 타협점을 찾아가도록 하면 된다. 종종 단순성과 성능이 심하게 상충하는 경우 성능을 위해 단순성(잠재적으로 안전성을 훼손하면서)을 희생하지 않도록 한다. 또한 블록 안의 코드가 무엇을 하는지, 수행 시간을 잘 파악하여 계산량이 많거나 잠재적 대기 상태에 들어갈 수 있는 작업이 있다면 락을 짧거나 잡지 않도록 해야 한다.