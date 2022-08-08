# 3장 객체 공유

---

  3장에서는 여러 개의 스레드에서 특정 객체를 동시에 사용하려 할 때 섞이지 않고 안전하게 동작하도록 객체를 공유하고 공개하는 방법을 살펴본다. 

  앞에서 synchronized를 사용해 단일 연산인 것처럼 동작하게 했지만, 반드시 synchronized 키워드를 사용해야 하는건 아니다. 특정 블록을 동기화시키고자 할 때는 항상 메모리 가시성memory visibility 문제가 발생하는데, 특정 변수의 값을 사용하고 있을 때 다른 스레드가 해당 변수의 값을 사용하지 못하도록 막아야 할 뿐만 아니라, 값을 사용한 다음 동기화 블록을 빠져나가고 나면 다른 스레드가 변경된 값을 즉시 사용할 수 있게 해야된다는것이다. 동기화가 없다면 다른 스레드에서 값을 제대로 사용하지 못하니, 항상 특정 객체를 명시적으로 동기화시키거나, 객체 내부에 적절한 동기화 기능을 내장시켜야 한다.
  
  <br/> 
  <br/>

  

## 3.1 가시성

  일반적으로는 특정 변수의 값을 가져갈 때 다른 스레드가 작성한 값을 가져갈 수 있다는 보장도 없고, 값을 읽지 못할 수도 있다. 다음 예제에서는 동기화 작업이 되어 있지 않은 상태에서 여러 스레드가 동일한 변수를 사용할 때 어떤 문제가 생기는지 볼 수 있다.

```java
// 예제 3.1 변수를 공유하지만 동기화되지 않은 예제 [Bad Example]
public class NoVisibility {
	private static boolean ready;
	private static int number;

	private static class ReaderThread extends Thread {
		public void run() {
			while(!ready)
				Thread.yield();
			System.out.println(number);
		}
	}
	
	public static void main(String[] args) {
		new ReaderThread().start();
		number = 42;
		ready = true;
	}
}
```

  위의 예제에서는 메인스레드에서 number변수과 ready 변수에 지정한 값을 읽기 스레드에서 사용할 수 없는 상황이며 두 개 스레드에서 변수를 공유해 사용함에도 불구하고 적절한 동기화 기법을 사용하지 않았기 때문에 발생했다.

  읽기 스레드가 ready 변수를 영영 못읽어서 무한 반복에 빠질 수 도 있지만, 읽기 스레드가 메인스레드에서 number 변수에 값을 저장하는 것보다 ready 변수의 값을 먼저 읽어가는 상황도 가능하다. 이는 ‘재배치reordering’ 라고 하는 현상이며, 특정 메소드의 소스코드가 100% 코딩된 순서로 동작한다는 점을 보장할 수 없다는 점에 기인하여 여러 스레드가 동시에 동작하는 경우에 확연하게 나타나는 문제이다. 
  
  <br/> 

### 3.1.1 스테일 데이터

예제 3.1에서는 스테일stale 데이터라는 결과를 체험했다. 변수를 사용하는 모든 경우에 동기화를 시켜두지 않으면 해당 변수에 대한 최신 값이 아닌 다른 값을 사용하게 되는 경우가 발생할 수 있다. 특정 스레드가 어떤 변수를 사용할 때 정상적인 최신 값을 사용할 ‘수’ 도 있고, 올바르지 않은 값을 사용할 ‘수’도 있다.

```java
// 예제 3.2 동기화되지 않은 상태로 정수 값을 보관하는 클래스[Not so good Example]
@NotThreadSafe
public class MutableInteger {
	private int value;

	public int get() { return value; }
	public void set(int value) { this.value = value; }
}
```

위의 예제는 스테일 현상이 두드러지게 나오는 예시이며 특정 스레드가 set 메소드를 호출하고 다른 스레드에서 get 메소드를 호출했을 때, set 메소드에서 지정한 값을 get 메소드에서 제대로 읽어가지 못할 가능성이 있다.

다음과 같이 get 메소드와 set 메소드를 동기화 시켜 문제점을 제거해보자. get, set 중 하나만 동기화 했다면 동기화 하지 않은 메소드에서 스테일 상황이 발생할 가능성이 생기게 되니까 주의하자.

```java
// 예제 3.3 동기화된 상태로 정수 값을 보관하는 클래스 [Good Example]
@ThreadSafe
public class SynchronizedInteger {
	@GuardedBy("this") private int value;

	public synchronized int get() { return value; }
	public synchronized void set(int value) { this.value = value; }
}
```

<br/>

### 3.1.2 단일하지 않은 64비트 연산

동기화하지 않은 상태에서 스테일 상태의 값을 읽어갈 가능성이 있긴 하지만 전혀 다른 값을 가져가는것이 아니라 다른 스레드에서 설정한 이전의 값을 가져간다. 하지만 64비트를 사용하는 숫자형(double, long 등)에 volatile 키워드를 사용하지 않은 경우에는 난데없는 값이 생길 가능성이 있다. 

자바 메모리 모델은 메모리에서 값을 가져오고fetch 저장store하는 연산이 단일해야 한다고 정의하고 있지만, volatile로 지정되지 않은 long, double 형의 64비트 값은 메모리에 읽고 쓸때 두 번의 32비트 연산을 사용할 수 있도록 허용하고 있다. 따라서 읽는 경우와 쓰는 기능이 서로 다른 스레드에서 동작한다면, 이전 값과 최신 값에서 각각 32비트를 읽어올 가능성이 생긴다.

<br/>

### 3.1.3 락과 가시성

내장된 락lock을 적절히 활용하면 그림3.1과 같이 특정 스레드가 특정 변수를 사용하려 할 때 이전에 동작한 스레드가 해당 변수를 사용하고 난 결과를 상식적으로 예측할 수 있는 상태에서 사용할 수 있다.

[그림3-1](https://user-images.githubusercontent.com/101235867/181454801-3ca7ab04-7ec1-427d-9655-0360df70d286.png)

락은 상호 배제mutual exclusion뿐만 아니라 정상적인 메모리 가시성을 확복하기 위해서도 사용한다. 변경 가능하면서 여러 스레드가 공유해 사용하는 변수를 각 스레드에서 각자 최신의 정상적인 값으로 활용하려면 동일한 락을 사용해 모두 동기화시켜야 한다.

<br/>

### 3.1.4 volatile 변수

volatile로 선언된 변수의 값을 바꿨을 때 다른 스레드에서 항상 최신 값을 읽어갈 수 있도록 해준다. 프로세서의 레지스터에 캐시되지도 않고, 프로세서 외부의 캐시에도 들어가지 않기 때문에 volatile 변수의 값을 읽으면 항상 다른 스레드가 보관해둔 최신의 값을 읽어갈 수 있다. 하지만 volatile 변수는 아무런 락이나 동기화 기능이 동작하는것은 아니라서 synchronized 동기화보다는 강도가 약하다.

메모리 가시성의 입장에서 본다면 volatile을 사용하는 것과 synchronized 키워드로 특정 코드를 묶는게 비슷해서 메모리 가시성에 효과가 있지만, volatile 변수만 사용해 가시성 확보한 코드는 synchronized 보다 읽기가 어렵고 오류 발생 가능성도 높아진다.

> 동기화 하고자 하는 부분을 명확하게 볼 수 있고, 간단한 구현일 경우만 volatile 변수를 활용하자. 반대로 가시성을 조금이라도 추론해봐야 하는 경우는 사용하지 말자.
> 

<br/>

다음 예제에서는 특정 변수의 값을 확인해 반복문을 빠져나갈 상황인지 확인하는 예이다. synchronized를 사용해 락을 걸어도 되지만 코드가 volatile 보단 좋지 않을 것이다.

```jsx
// 예제 3.4 양 마리 수 세기
volatile boolean asleep;
...
   while(!asleep)
			countSomeSheep();
```


<br/>
간편하게 사용할 수 있는 반면 제약 사항도 있다. 작업 완료, 인터럽트 걸리거나, 기타 상태를 보관하는 플래그 변수에 주로 volatile 키워드를 지정한다.

> 락을 사용하면 가시성과 연산의 단일성을 모두 보장받을 수 있다.
volatile 변수는 가시성은 보장하나 연산의 단일성은 보장하지 못한다.
> 


<br/>
결론적으로 volatile 변수는 다음과 같은 상황에서만 사용하자

- 변수의 값을 저장하는 작업이 현재 변수와 관련이 없거나 하나의 스레드에서만 변경하는 경우
- 해당 변수가 불변 조건에 관련되어 있지 않은 경우
- 해당 변수 사용 중 락이 필요 없는경우

<br/>
<br/>

## 3.2 공개와 유출

특정 객체를 현재 코드의 스코프 범위 밖에서 사용할 수 있도록 만들면 공개published 되었다고 한다. (스코프 밖 코드에서 볼 수 있는 변수에 내부 객체에 대한 참조 저장, private이 아닌 메소드에서 호출한 메소드가 내부 생성 객체 리턴 등) 일반적으로는 특정 객체, 해당 객체 내부의 구조가 공개되지 않도록 작업하는데, 특정 객체를 공개해서 공유해 사용하게 된다면 반드시 객체를 동기화 시켜야 한다. 의도적이지 않지만 외부에서 사용할 수 있게 공개된 경우를 유출 상태escaped라고 한다. 

  다음 예제에서는 객체를 외부에 안전하는 공개하는 방법이다. public static 변수에 객체를 설정해서 가장 직접적인 방법으로 모든 클래스와 스레드에 공개한것이다.

 

```jsx
// 예제 3.5 객체 공개
public static Set<Secret> knownSecreats;

public void initialize() {
	knownSecrets = new HashSet<Secret>();
}
```

특정 객체를 공개했으나 관련된 다른 객체까지 공개되는 경우도 있다. private 키워드로 숨겨져있어도 다음 예제 처럼 공개 하게 되면 states 변수의 값을 직접 변경할수도 있게 되어 유출 상태에 놓여 있게 된다. 

```jsx
// 예제 3.6 내부적으로 사용할 변수를 외부에 공개 [Bad Example]
class UnsafeStates {
	private String[] states = new String[] { "AK", "AL", ... };
	private String[] getStates() { return states; }
}
```

객체를 공개했을 때 그 객체 내부의 private이 아닌 변수나 메소드를 통해 불러올 수 있는 모든 객체는 함께 공개된다는 점을 알아두자. 

객체, 객체 내부의 값이 외부에 공개되는 또 다른 예는 다음과 같으며 내부 클래스의 인스턴스를 외부에 공개하는 경우이다. 

```jsx
// 예제 3.7 this 클래스에 대한 참조를 외부에 공개. [Bad Example]
public class ThisEscape {
	public ThisEscape(EventSource source) {
		source.registerListener(
			new EventListener() {
				public void onEvent(Event e) {
					doSomething(e);
				}
		});
	}
}
```

내부 클래스는 항상 부모 클래스에 대한 참조를 갖고 있기 때문에 ThisEscape 클래스가 EventListner 객체를 외부에 공개하면 EventListener 클래스를 포함하고 있는 ThisEscape 클래스도 함께 외부에 공개된다.

<br/>

### 3.2.1 생성 메소드 안전성

예제 3.7에서는 생성 메소드를 실행하는 과정에 this 변수가 외부에 유출되는 것을 보았다. 생성 메소드 실행 도중에 this 변수가 외부에 공개되어 정상적으로 객체가 생성 완료가 되지 않을 수도 있는것이다. 또 다른 this가 유출되는 경우는 생성 메소드에서 스레드를 새로 만들어 시작시키는 것이다. 대부분의 경우 생성 메소드의 클래스와 새로운 스레드가 this 변수를 직접 공유하거나 자동으로 공유되기도 한다. 생성까지는 큰 문제는 없지만, 생성과 동시에 시작시키는것은 문제가 생긴다. 

다음 예제처럼 생성 메소드를 private으로 지정하고 public으로 지정된 팩토리 메소드를 만들어 사용하는 방법이 좋다.

```jsx
// 예제 3.8 생성 메소드에서 this 변수가 외부로 유출되지 않도록 팩토리 메소드를 사용하는 모습
public class SafeListener {
	private final EventListener listener;

	private SafeListener() {
		listener = new EventListener() {
			public void onEvent(Event e) {
				doSomething(e);
			}
		};
	}
	
	public static SafeListener newInstance(EventSource source) {
		SafeListener safe = new SafeListener();
		source.registerListener(safe.listener);
		return safe;
	}
}
```

<br/>
<br/>

## 3.3 스레드 한정

객체를 사용하는 스레드를 한정confine하는 방법으로 스레드 안전성을 확보할 수 있다. 언어적인 차원에서 특정 변수를 대상으로 락을 걸 수 있는 기능을 제공하지 않은 것처럼, 임의의 객체를 특정 스레드에 한정시키는 기능도 제공하지 않는다.

<br/>

### 3.3.1 스레드 한정 - 주먹구구식

구현 단계에서 임시방편으로 스레드 한정 기법을 적용할 수 있다. 스레드 한정 기법을 사용할 것인지를 결정하는 일은 GUI 모듈과 같은 특정 시스템을 단일 스레드로 동작하도록 만들 것이냐에 달려있다. 특정 모듈의 기능을 단일 스레드로 동작하도록 구현한다면 오류 가능성을 최소화할 수 있다. 

<br/>

### 3.3.2 스택 한정

스택 한정 기법은 특정 객체를 로컬 변수를 통해서만 사용할 수 있는 특별한 경우의 스레드 한정 기법이다. 로컬 변수는 현재 실행 중인 스레드 내부의 스택에만 존재하기 때문이며, 스레드 내부의 스택은 외부 스레드에서 볼 수 없다.

```jsx
// 예제 3.9 기본 변수형의 로컬 변수와 객체형의 로컬 변수에 대한 스택 한정
public int loadTheArk(Collection<Animal> candidates) {
	SortedSet<Animal> animals;
	int numPairs = 0;
	Animal candidate = null;
	
	// animals 변수는 메소드에 한정되어 있으며, 유출돼서는 안된다.
	animals = new TreeSet<Animal>(new SpeciesGenderComparator());
	animals.addAll(candidates);
	for(Animal a : animals) {
		if(candidate == null || !candidate.isPotentialMate(a))
			candidate = a;
		else {
			ark.load(new AnimalPair(candidate, a));
			++numPairs;
			candidate = null;
		}
	}
	return numPairs;
}
```

기본 변수형을 사용하는 로컬 변수는 객체와 같이 참조되는 값이 아니기 때문에 언어적으로 스택 한정 상태가 보장된다. 


<br/>

### 3.3.3 ThreadLocal

스레드 내부의 값과 값을 갖고 있는 객체를 연결해 스레드 한정 기법을 적용할 수 있도록 도와주는 형식적인 방법으로 ThreadLocal이 있다. ThreadLocal 클래스에는 get, set 메소드가 있는데 get 메소드를 호출하면 현재 실행 중인 스레드에서 최근에 set 메소드를 호출해 저장했던 값을 가져올 수 있다. 

```jsx
// 예제 3.10 ThreadLocal을 사용해 스레드 한정 상태를 유지
private static TreadLocal<Connection> connectionHolder 
	= new ThreadLocal<Connection() {
		public Connection initialValue() {
			return DriverManager.getConnection(DB_URL);
		}
};

public static Connection getConnection() {
	return connectionHolder.get();
}
```

이런 방법은 임시로 사용할 객체를 매번 새로 생성하는 대신 이미 만들어진 객체를 재활용하고자 할 때 많이 사용한다. 

<br/>
<br/>

## 3.4 불변성

객체의 상태를 변하지 않게 해서 연산의 단일성이나 가시성에 대한 문제를 해결해주는 방법이다.  불변 객체는 맨 처음 생성되는 시점을 제외하고는 그 값이 전혀 바뀌지 않는 객체를 말하며 태생부터 스레드에 안전한 상태이다. 

다음 조건을 만족하면 해당 객체는 불변 객체이다.

- 생성되고 난 이후에는 객체의 상태를 변경할 수 없다.
- 내부의 모든 변수는 final로 설정돼야 한다.
- 적절한 방법으로 생성돼야 한다.(ex: this 변수 참조가 외부에 유출 X)

```jsx
// 예제 3.11 일반 객체를 사용해 불변 객체를 구성한 모습
@Immutable
public final class ThreeStooges {
	private final Set<String> stooges = new HashSet<String>();

	public ThreeStooges() {
		stooges.add("Moe");
		stooges.add("Larry");
		stooges.add("Curly");
	}

	public boolean isStooge(String name) {
		return stooges.contains(name);
	}
}
```

위의 예제는 상태 관리를 위해서 내부적으로 일반 변수나 객체를 사용할수있는 것을 볼 수 있다. Set 변수는 변경 가능 객체지만, 생성 메소드 실행 이후 값을 변경할수 없도록 되어있다. 객체의 모든 상태는 final 변수를 통해 사용해야 하며, 생성 메소드에서 this 변수 참조가 외부로 유출될만한 일이 없으므로 이 클래스는 불변 객체라고 할 수 있다. 

<br/>

### 3.4.1 final 변수

final을 지정한 변수의 값은 변경할 수 없다.(물론 변수가 가리키는 객체가 불변 객체가 아니라면 해당 객체에 들어있는 값은 변경할 수 있다.) final 키워드는 초기화 안전성initialization safety을 보장하기 때문에 별다른 동기화 작업 없이도 불변 객체를 자유롭게 사용하고 공유할 수있다.

<br/>

### 3.4.2 예제 : 불변 객체를 공개할 때 volatile 키워드 사용

앞서 volatile 키워드를 사용했으나 두 값이 단일 연산으로 작동하지 않았기에 스레드에 안전하지 않았던 예제를 불변 객체를 활용하면 연산의 단일성을 보장할 수 있게 된다. 

다음 예제에서는 단일 연산 처리를 해야 하는 두가지 작업을 하나의 불변 클래스로 묶어서 경쟁 조건을 방지해주는 예시이다. 

```jsx
// 예제 3.12 입력 값과 인수분해 된 결과를 묶는 불변 객체
@Immutable
class OneValueCache {
	private final BigInteger lastNumber;
	private final BigInteger[] lastFactors;

	public OnValueCache(BigInteger i, BigInteger[] factors) {
		lastNumber = i;
		lastFactors = Arrays.copyOf(factors, factors.length);
	}

	public BigInteger[] getFactors(BigInteger i) {
		if(lastNumber == null || !lastNumber.equals(i))
			retrun null;
		else
			return Arrays.copyOf(lastFactors, lastFactors.length);
	}
}
```

아래 예제는 위의 클래스를 사용해 입력 값과 결과를 캐시한다. 스레드 하나가 volatile로 선언된 cache 변수에 새로 생성한 OneValueCache 인스턴스를 설정하면, 다른 스레드에서도 cache 변수에 설정된 새로운 값을 즉시 사용할 수 있다. 

```jsx
// 예제 3.13 최신 값을 불변 객체에 넣어 volatile 변수에 보관
@ThreadSafe
public class VolatileCachedFactorizer implements Servlet {
	private volatile OneValueCache cache = new OneValueCache(null, null);

	public void service(ServletRequest req, ServletResponse resp) {
		BigInteger i = extractFromRequest(req);
		BigInteger[] factors = cache.getFactors(i);
		if(factors == null) {
			factors = factor(i);
			cache = new OneValueCache(i, factors);
		}
		encodeIntoResponse(resp, factors);
	}
}
```


<br/>
<br/>

## 3.5 안전 공개

지금까지는 객체를 숨기는 방법을 알아봤다. 여기서는 안전하게 여러 스레드에게 공유하는 방법을 알아보고자 한다. 다음 예제에서는 객체에 대한 참조를 public변수에 넣어 공개하여 안전하지 않은 공개를 보여준다.

```jsx
// 예제 3.14 동기화하지 않고 객체를 외부에 공개[Bad Example]
public Holder holder;

public void initialize() {
	holder = new Holder(42);
}
```

<br/>

### 3.5.1 적절하지 않은 공개 방법 : 정상적인 객체도 문제를 일으킨다

생성 메소드가 실행되고 있는 상태의 인스턴스를 다른 스레드가 사용하려 한다면 비정상적인 상태임에도 사용되어질것이고 값이 다르거나 하는 여러 문제가 발생하게 된다. 다음과 같은 예시가 그러하다. Holder 객체를 다른 스레드가 사용할 수 있도록 작성하면서 적절한 동기화 방법을 적용하지 않았으므로 올바르게 공개되지 않았다고 할 수 있다.

```jsx
// 예제 3.15 올바르게 공개하지 않으면 문제가 생길 수 있는 객체
public class Holder {
	private int n;
	public Holder(int n) { this.n = n; }
	public void assertSanity() {
		if(n != n)
			throw new AssertionError("This statement is false");
	}
}
```

올바르게 공개하지 않으면 두가지 문제가 발생할 수 있는데, 첫 번째 문제는 holder 변수에 스테일 상태가 발생할 수 있으며, 두 번째는 다른 스레드는 정상적인 참조 값을 가져갔으나, Holder 클래스의 입장에서 스테일 상태에 빠질 수 있다. 

<br/>

### 3.5.2 불변 객체와 초기화 안정성

자바 메모리 모델에는 불변 객체를 공유하고자 할 때 초기화 작업을 안전하게 처리할 수 있는 방법이 만들어져 있다. 불변 객체를 사용하면 객체의 참조를 외부에 공개할 때 추가적인 동기화 방법을 사용하지 않았다 해도, 항상 안전하게 올바른 참조 값을 사용할 수 있다. 

final로 선언된 모든 변수는 별다른 동기화 작업 없이도 안전하게 사용할 수 있으나, final로 선언된 변수에 변경 가능한 객체가 지정되어 있다면 해당 변수에 들어 있는 객체의 값을 사용하려고 하는 부분을 모두 동기화시켜야 한다.

<br/>

### 3.5.3 안전한 공개 방법의 특성

공개한 값을 불러다 사용하는 스레드에서 공개한 스레드가 공개했던 상태를 정확하게 볼 수 있도록 만드는 방법을 우선 살펴보자.  해당 객체에 대한 참조와 객체 내부의 상태를 외부의 스레드에게 동시에 볼 수 있어야 하며, 올바르게 생성 메소드가 실행되고 난 객체는 다음과 같은 방법으로 안전하게 공개할 수 있다. 

- 객체에 대한 참조를 static 메소드에서 초기화시킨다.
- 객체에 대한 참조를 volatile 변수 또는 AtomicReference 클래스에 보관한다.
- 객체에 대한 참조를 올바르게 생성된 클래스 내부의 final 변수에 보관한다.
- 락을 사용해 올바르게 막혀 있는 변수에 객체에 대한 참조를 보관한다.

<br/>

### 3.5.4 결과적으로 불변인 객체

처음 생성 이후 그 내용이 바뀌지 않도록 만들어진 클래스에 안전한 공개 방법을 사용하면 별다른 동기화 방법 없이도 다른 스레드에서 얼마든지 사용해도 문제가 없다. 한번 공개된 이후에는 그 내용이 변경되지 않는다고 하면 결과론적으로 해당 객체도 불변 객체라고 볼 수 있다. 즉, 안전하게 공개한 결과적인 불변 객체는 별다른 동기화 작업 없이도 여러 스레드에서 안전하게 호출해 사용할 수 있다.

<br/>

### 3.5.3 가변 객체

객체의 생성 메소드를 실행한 이후에 그 내용이 변경될 수 있다면, 안전하게 공개했다하더라도 그저 공개한 상태를 다른 스레드가 볼 수 있다는 정도만 보장할 수 있다. 가변 객체mutable object를 안전하게 사용하려면 안전하게 공개해야만 하고, 동기화와 락을 사용해 스레드 안전성을 확보해야만 한다.

<br/>

### 3.5.6 객체를 안전하게 공유하기

언제든 객체에 대한 참조를 사용하는 부분이 있다면, 그 객체로 어느 정도의 일을 할 수 있는지를 정확하게 알고 있어야 한다. 또한, 반대로 객체를 외부에서 사용할 수 있도록 공개할 때에는 해당 객체를 어떤 방법으로 사용할 수 있고, 사용해야 하는지에 대해서 정확하게 설명해야 한다.