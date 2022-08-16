# 4장 객체 구성

컴포넌트의 스레드 안전성을 안정적으로 확보할 수 있고, 이 과정에서 개발자가 스레드 안전성을 해치지 않도록 도와주는 클래스 구성 방법을 살펴보자.

<br/>

<br/>

## 4.1 스레드 안전한 클래스 설계

구조적인 캡슐화 없이 만든 결과물은 여러 스레드에서 사용해도 안전한지 확인하기도 어렵고, 나중에 변경해야 할 필요가 있을 때에도 스레드 동기화 문제 없이 변경하기 어렵다. 객체가 갖고 있는 여러 정보를 해당 객체 내부에 숨겨두면 객체 단위로 스레드 안전성을 확보할 수 있다. 

<aside>
❓ 스레드 안전성이 확보된 클래스를 설계하려면 고려해야 할 사항

 + 객체의 상태를 보관하는 변수가 어떤 것인가? 
 + 객체의 상태를 보관하는 변수가 가질 수 있는 값이 어떤 종류, 어떤 범위에 해당하는가?
 + 객체 내부의 값을 동시에 사용하고자 할 때, 그 과정을 관리할 수 있는 정책

</aside>

객체의 상태는 항상 객체 내부의 변수를 기반으로 한다. 다음 예제는 클래스에 단 하나의 변수를 갖고 있으며, 따라서 해당 클래스의 상태는 해당 변수만 보면 파악이 가능하다. 또한 해당 객체에 또 다른 객체를 가리키는 변수가 있다면 관련된 객체 전부 상태를 파악해야 하는것이다. 

```java
// 예제 4.1 자바 모니터 패턴을 활용해 스레드 안전성을 확보한 카운터 클래스
@ThreadSafe
public final class Counter {
	@GuardedBy("this") private long value = 0;

	public synchronized long getValue() {
		return value;
	}

	public synchronized long increment() {
		if(value == Long.MAX_VALUE)
			throw new IllegalStateException("counter overflow");
		return ++value;
	}
}
```

객체 내부의 여러 변수가 갖고 있는 현재 상태를 사용하고자 할 때 값이 계속해서 변하는 상황에서도 값을 안전하게 사용할 수 있도록 조절하는 방법을 동기화 정책이라고 한다. 

<br/>

### 4.1.1 동기화 요구사항 정리

여러 스레드가 동시에 클래스를 사용하려 하는 상황에서 클래스 내부의 값을 안정적인 상태로 유지할 수 있다면 스레드 안전성을 확보했다고 할 수 있다. 객체와 변수가 가질 수 있는 가능한 값의 범위를 상태 범위state space라고 하며, 상태 범위가 좁을수록 객체의 논리적인 상태를 파악하기 쉽다.

대부분의 클래스에는 특정 상태가 올바른 상태인지 아닌지 확인할 수 있는 마지노선이 있다. (long 타입: Long.MIN_VALUE ~ Long.MAX_VALUE, 음수 안됨 등). 

상태 변화가 올바른지 아닌지 결정하는 조건도 있다. 클래스의 다음 상태는 이전 상태 기반(17값)에서 바뀌여야(18값)하는데 현재의 값 기준으로 바뀐다면 올바르지 않은 상태변화인 것이다. (다만 현재 온도값 측정을 하는 기능인 경우 이런 조건은 관련 없음)

또한 클래스 내부에서 상태 범위에 두 개 이상의 변수가 연결되어 동시에 관여하고 있다면 이 변수들을 사용하는 모든 부분에서 락을 사용해 동기화를 맞춰야 한다. 

<br/>

### 4.1.2 상태 의존 연산

클래스가 가질 수 있는 값의 범위와 값이 변화하는 여러 가지 조건을 살펴보면 어떤 상황이여야 클래스가 정상적인지를 정의할 수 있고, 특정 객체는 상태를 기반으로 하는 선행 조건precondition도 갖기도 한다. 

현재 조건에 따라 동작 여부가 결정되는 연산을 상태 의존state-dependent이라 하며, 올바른 상태가 아닌 상황에서 단일 스레드는 상시 오류가 발생할 수 있으나, 여러 스레드에서는 실행 이후, 선행 조건이 올바른 상태로 변할 수도 있다. 

특정 상태가 원하는 조건에 다다를 수 있도록 wait과 notify를 사용할 수 있으나, 세마포어나 블로킹 큐와 같이 알려져있는 라이브러리를 사용하는 편이 간단하고 안전하다.

<br/>

### 4.1.3 상태 소유권

변수를 통해 객체의 상태를 정의하고자 할 때에는 해당 객체가 실제로 소유하는 데이터만을 기준으로 삼아야 한다. (HashMap객체 내의 Map.Entry객체와 기타 인스턴스 객체 상태를 한번에 다뤄야 한다)

자바에선 객체의 소유권 설정을 조절할수는 있으나, 객체를 공유하는데 있어 오류가 발생하기 쉬운 부분을 가비지 컬렉터가 대부분 알아서 조절해주기 때문에 소유권 개념이 훨씬 불명확한 경우가 많다. 

대부분의 경우 소유권과 캡슐화 정책은 함께 고려하는 경우가 많은데, 캡슐화 정책은 객체의 상태에 대한 소유권을 갖고 있기 때문에 특정 변수의 상태를 조절하는 락 구조의 움직임에 대해서도 소유권을 갖게 된다. 그러나 특정 변수를 객체 외부로 공개하게 되면 통제권의 일부를 잃으며 공동 소유권 정도를 갖게 된다. 

컬렉션 클래서에서는 소유권 분리의 형태를 사용하기도 하며, 컬렉션 내부의 구조에 대한 소유권은 컬렉션 클래스가, 컬렉션에 추가되어 있는 객체에 대한 소유권은 클라이언트가 갖게 되는 것이다. (ex: ServletContext 클래스)

<br/>

<br/>

<br/>

## 4.2 인스턴스 한정

스레드 한정 기법을 사용해 특정 스레드 내부에서만 사용하거나, 해당 객체 사용 부분에 락을 걸어 동시에 사용되는 경우를 막아 멀티 스레드 프로그램에서 안전하게 사용할 수 있다. 

객체를 적절하게 캡슐화 하면 스레드 안전성을 확보할 수 있는데, 이는 인스턴스 한정 기법을 활용하여 특정 객체를 다른 객체 내부에 완벽하게 숨겨, 해당 객체 활용 방법을 한눈에 파악하고 간편하게 스레드 안전성을 분석할 수 있게 된다. 

```java
// 예제 4.2 한정 기법으로 스레드 안전성 확보
@ThreadSafe
public class PersonSet {
	@GuardedBy("this") private final Set<Person> mySet = new HashSet<Person>();

	public synchronized void addPerson(Person p) {
		mySet.add(p);
	}

	public synchronized boolean containsPerson(Person p) {
		return mySet.contains(p);
	}
}
```

위의 예제에서 HashSet은 스레드 안전 객체는 아니나 private 지정하여 외부 유출을 막았다. 해당 변수를 사용하는 유일한 방법들은 synchronized 키워드를 통해 락을 걸어서 보호 했다. 또 다른 객체인 Person도 등장하나 변경하는 내용이 없으므로 상관없으며, 사용하고자 하면 해당 객체 내에서 스레드 안전성을 확보하는 것이 가장 좋다. 

한정됐어야 할 객체를 공개함으로써 한정 조건이 깨질 가능성은 여전히 존재한다. 

<br/>

### 4.2.1 자바 모니터 패턴

자바 모니터 패턴을 따르는 객체는 변경가능한 데이터를 모두 객체 내부에 숨긴다음 객체의 암묵적인 락으로 데이터에 대한 동시 접근을 막는 것이다. 예제 4.1은 자바 모니터 패턴의 전형적인 예이다.

자바 모니터 패턴은 단순한 관례에 불과하며 일정한 형태로 스레드 안전성을 확보할 수만 있다면 어떤 형태의 락을 사용해도 무방하다. 다음 예제에서는 private과 final로 선언된 객체를 락으로 사용해 자바 모니터 패턴을 활용하는 모습이다.

```java
// 예제 4.3 private이면서 final인 변수를 사용해 동기화
public class PrivateLock {
	private final Object myLock = new Object();
	@GuardedBy("myLock") Widget widget;

	void someMethod() {
		synchronized(myLock) {
			// widget 변수의 값을 읽거나 변경
		}
	}
}
```

객체 자체의 암묵적인 락이 아닌, 위와 같은 private 객체 락은 외부에서 락을 건드릴 수 없기 때문에 외부에서 해당 락으로 동기화 작업에 참여하는것을 막아서 스레드 안전성을 지킬 수 있다. 

<br/>

### 4.2.2 예제 : 차량 위치 추적

더 큰 예제로 자바 모니터 패턴을 알아보자. 차량의 위치를 추적하는 프로그램을 만들것이며, 우선 자바 모니터 패턴에 맞춰 작성한 뒤 스레드 안전성을 보장하는 한도 내에서 객체 캡슐화 정도를 낮추어 볼것이다.

- 차량은 String 형태의 ID로 구분되며 차량의 위치는 (x, y) 좌표로 표시한다. 차량의 ID와 위치를 클래스 객체 내부에 캡슐화해서 보관한다.
- 뷰 스레드는 특정 차량의 ID와 위치를 읽어서 화면상의 특정 위치에 차량에 대한 정보를 표시한다.
    
    ```jsx
    Map<String, Point> locations = vehicles.getLocations();
    for(String key : locations.keySet())
    	renderVehicle(key, locations.get(key));
    ```
    
- 업데이터 스레드는 GPS 장치에서 읽어낸 위치 정보를 자동으로 입력하거나, GUI 화면에서 수동으로 입력한 내용을 새로운 위치 정보로 업데이트 한다.
    
    ```jsx
    void vehicleMoved(VehicleMovedEvent evt) {
    	Point loc = evt.getNewLocation();
    	vehicles.setLocation(eve.getVehicleId(), loc.x, loc.y);
    }
    ```
    

이런 구조라면 뷰 슬드와 업데이터 스레드가 동시 다발적으로 데이터 모델을 사용하므로 데이터 모델에 해당하는 클래스는 반드시 스레드 안전성을 확보해야 한다.

외부에서 데이터를 요청하는 경우 복사본을 넘겨주는 방법을 사용하면 스레드 안전성을 부분적이나마 확보할 수 있지만, 요청의 수가 많아지는 경우 성능에 문제가 발생할 수 있다. 또한 특정 시점의 고정 값을 원한 경우에만 사용해야 한다. 시시각각 변하는 데이터를 원한다면 맞지 않기 때문이다.

```jsx
// 예제 4.4 모니터 기반의 차량 추적 프로그램
@ThreadSafe
public class MonitorVehicleTracker {
	@GuardedBy("this")
	private final Map<String, MutablePoint> locations;

	public MonitorVehicleTraker(Map<String, MutablePoint> locations) {
		this.locations = deepCopy(locations);
	}

	public synchronized Map<String, MutablePoint> getLocations() {
		return deepCopy(locations);
	}

	public synchronized MutablePoint getLocation(String id) {
		MutablePoint loc = locations.get(id);
		return loc == null ? null : new MutablePoint(loc);
	}

	public synchronized void setLocation(String id, int x, int y) {
		MutablePoint loc = locations.get(id);
		if(loc == null)
			throw new IllegalArgumentException("No such ID: " + id);
		loc.x = x;
		loc.y = y;
	}
	private static Map<String, MutablePoint> deepCopy(Map<String, MutablePoint> m) {
		Map<String, MutablePoint> result = new HashMap<String, MutablePoint();
		for(String id : m.keySet())
			result.put(id, new MutablePoint(m.get(id)));
		return Collections.unmodifiableMap(result);
	}
}
```

위의 예제에서는 차량의 위치를 담는 MutablePoint 클래스(4.5)를 활용해 자바 모니터 패턴에 맞춰 만들어진 차량 추적 클래스가 나타나 있다.

```jsx
// 예제 4.5 java.awt.Point와 유사하지만 변경 가능한 MutablePoint 클래스[Not so good Example]
@NotThreadSafe
public class MutablePoint {
	public int x, y;

	public MutablePoint() { x = 0; y = 0; }
	public MutablePoint(MutablePoint p) {
		this.x = p.x;
		this.y = p.y;
	}
}
```

위의 예제는 스레드 안전하지는 않지만 차량 추적 클래스는 스레드 안전성을 확보하고 있다. locations 변수와 Point 인스턴스 모두 외부에 공개되지 않았다. 차량의 위치는 클래스 생성자를 통해 복사본을 만들거나 deepCopy 메소드를 사용해 새로운 lcoations 인스턴스 복사본을 만들어 넘겨준다. 

<br/>

<br/>

<br/>

## 4.3 스레드 안전성 위임

대부분의 객체는 둘 이상의 객체를 조합해 사용하는 합성composite 객체이다. 스레드 안전성을 확보한 객체를 조합하는 경우 스레드 동기화 설정을 더 해야 할까? 답은 ‘상황에 따라 다르다’이다.  

예제 중 하나였던 CountingFactorizer 클래스에는 스레드 안전한 AtomicLong 객체를 제외하고는 상태가 없으며, CF 클래의 상태는 바로 스레드 안전한 AtomicLong 클래스의 상태와 같기 때문이고, AtomicLong에 보관하는 값에 제한 조건이 없기 때문에 스레드 안전하다. 이런 경우 CF 클래스는 스레드 안전성 문제를 AtomicLong 클래스에게 ‘위임delegate’ 한다고 한다.

<br/>

### 4.3.1 예제: 위임 기법을 활용한 차량 추적

차량 추적 프로그램에서 스레드 안전성 위임 기법을 활용해 수정해보고자 한다. Map 클래스에 보관했었던 차량의 위치를 ConcurrentHashMap 클래스를 만들어 적용해보려고 한다. MutablePoint 대신 값을 변경 할 수 없는 Point 클래스를 사용하자.

```jsx
// 예제 4.6 값을 변경할 수 없는 Point 객체. DelegatingVehicleTracker에서 사용
@Immutable
public class Point {
	public final x, y;
	public Point(int x, int y) {
		this.x = x;
		this.y = y;
	}
}
```

Point 클래스는 불변이기 때문에 스레드 안전하다. 불변의 값은 안전하게 공유하고 공개할 수 있으므로 객체 인스턴스를 복사해 줄 필요가 없다. 

```jsx
// 예제 4.7 스레드 안전성을 ConcurrentHashMap 클래스에 위임한 추적 프로그램
@ThreadSafe
public class DelegatingVehicleTracker {
	private final ConcurrentMap<String, Point> locations;
	private final Map<String, Point> unmodifiableMap;

	public DelegatingVehicleTracker(Map<String, Point> points) {
		locations = new ConcurrentHashMap<String, Point>(points);
		unmodifiableMap = Collections.unmodifiableMap(locations);
	}

	public Map<String, Point> getLocations() {
		return unmodifiableMap;
	}

	public Point getLocation(String id) {
		return locations.get(id);
	}
	
	public void setLocation(String id, int x, int y) {
		if(locations.replace(id, new Point(x, y)) == null)
			throw new IllegalArgumentException("Invalid vehicle name:" + id);
	}
}
```

Point 대신 MutablePoint를 사용했다면 getLocations 메소드를 호출한 쪽에 변경 가능한 내부의 상태가 그대로 노출되어 스레드 안전성이 깨졌을 것이다. 위임 기능한 버전은 이전의 고정된 스냅샷 예제와 다르게 실시간 확인 가능한 동적인 데이터를 넘겨주었다. (스레드 A가 getLocations 호출 후 값을 가져단 후 스레드 B가 위치 값을 변경하면 A에서도 B가 변경한 값을 볼 수 있는것)

```jsx
// 예제 4.8 위치 정보에 대한 고정 스냅샷을 만들어 내는 메소드
public Map<String, Point> getLocations() {
	return Collections.unmodifiableMap(new HashMap<String, Point>(locations));
}
```

만약 특정 시점의 고정된 위치 데이터를 갖고 싶다면 위의 예제 처럼 Map 클래스에 대한 단순 복사본을 넘겨 줄 수 있다. 

<br/>

### 4.3.2 독립 상태 변수

이번에는 위임하고자 하는 내부 변수를 두 개 이상하려 한다. 두 개 이상의 변수가 서로 ‘독립적’이라면 클래스 스레드 안전성을 위임할 수 있는데, 독립적이란 변수가 서로의 상태 값에 대한 연관성이 없다는 말이다. 

```jsx
// 예제 4.9 두 개 이상의 변수에게 스레드 안전성을 위임
public class VisualComponent {
	private final List<KeyListener> keyListeners = new CopyOnWriteArrayList<KeyListener>();
	private final List<MouseListener> mouseListeners = new CopyOnWriteArrayList<MouseListener>();

	public void addKeyListener(KeyListener listener) {
		keyListeners.add(listener);
	}

	public void addMouseListener(MouseListener listener) {
		mouseListeners.add(listener);
	}

	public void removeKeyListener(KeyListener listener) {
		keyListeners.remove(listener);
	}

	public void removeMouseListener(MouseListener listener) {
		mouseListeners.remove(listener);
	}
}
```

위의 클래스는 클라이언트가 마우스와 키보드 이벤트를 처리하는 리스너를 등록할 수 있는 화면 컴포넌트이다. 내부적으로 마우스 리스너를 관리하는 목록 변수와 키보드 리스너 관리 목록 변수는 서로 아무런 연관이 없으므로 서로 독립적이다. 따라서 해당 클래스는 스레드 안전한 두 개의 이벤트 리스너 목록에게 클래스의 스레드 안전성을 위임할 수 있다. 

<br/>

### 4.3.3 위임할 때의 문제점

아래의 예제에서는 두 개의 변수가 서로 의존성을 가진 예제이다. 

```jsx
// 예제 4.10 숫자 범위를 나타내는 클래스. 의존성 조건을 정확하게 처리하지 못하고 있다. 
public class NumberRange {
	// 의존성 조건 : lower <= upper
	private final AtomicInteger lower = new AtomicInteger(0);
	private final AtomicInteger upper = new AtomicInteger(0);

	public void setLower(int i) {
		// 주의 - 안전하지 않은 비교문
		if(i > upper.get())
			throw new IllegalArgumentException("can't set lower to " + i + " > upper");
		lower.set(i);
	}

	public void setUpper(int i) {
		// 주의 - 안전하지 않은 비교문
		if(i < lower.get())
			throw new IllegalArgumentException("can't set upper to " + i + " < lower");
		upper.set(i);
	}

	public boolean isInRange(int i) {
		return (i >= lower.get() && i <= upper.get());
	}
}
```

위의 클래스는 upper, lower 변수 주변에 락을 사용하는 등의 방법을 적용해 동기화하면 쉽게 의존성 조건을 충족시킬 수 있다. 이렇게 부 개 이상의 변수를 사용하는 복합 연산 메소드를 갖고 있다면 위임 기법만으로는 스레드 안전성을 확보할 수 없다. 이런 경우에는 내부적으로 락을 활용해서 복합 연산이 단일 연산으로 처리되도록 동기화해야 한다. 

<br/>

### 4.3.4 내부 상태 변수를 외부에 공개

안전성을 위임받은 내부 상태 변수의 값을 외부 프로그램이 변경할 수 있도록 외부에 공개하고자 한다면 어떤 작업이 필요할까? 상태 변수가 스레드 안전하고 클래스 내부에서 상태 변수의 값에 대한 의존성을 갖고 있지 않고, 상태 변수에 대한 어떤 연산을 수행하더라도 잘못된 상태에 이를 가능성이 없다면, 해당 변수는 외부에 공개해도 안전하다. 

<br/>

### 4.3.5 예제: 차량 추적 프로그램의 상태를 외부에 공개

차량 추적 프로그램이 갖고 있는 변경 가능한 내부 상태를 외부에 공개하는 구조로 구현해보자. 

```jsx
// 예제 4.11 값 변경이 가능하고 스레드 안전성도 확보한 SafePoint 클래스
@ThreadSafe
public class SafePoint {
	@GuardedBy("this") private int x, y;
	private SafePoint(int[] a) { this(a[0], a[1]); }
	private SafePoint(SafePoint p) { this(p.get()); }
	public SafePoint(int x, int y) { this.set(x, y); }
	public synchronized int[] get() { return new int[] { x, y }; }
	public synchronized void set(int x, int y) {
		this.x = x;
		this.y = y;
	}
}
```

좌표의 x, y좌표를 배열로 가져오는 get 메소드는 x, y를 따로 두지 않고 함께 두어서 각각 요청 할 때 값이 달라지게 되는 상황을 막아준다.

```jsx
// 예제 4.12 내부 상태를 안전하게 공개하는 차량 추적 프로그램
@ThreadSafe
public class PublishingVehicleTracker {
	private final Map<String, SafePoint> locations;
	private final Map<String, SafePoint> unmodifiableMap;

	public PublishingVehicleTracker(Map<String, SafePoint> locations) {
		this.locations = new ConcurrentHashMap<String, SafePoint>(locations);
		this.unmodifiableMap = Collections.unmodifiableMap(this.locations);
	}

	public Map<String, SafePoint> getLocations() {
		return unmodifiableMap;
	}

	public SafePoint getLocation(String id) {
		return locations.get(id);
	}
	
	public void setLocation(String id, int x, int y) {
		if(!locations.containsKey(id))
			throw new IllegalArgumentException("Invalid vehicle name:" + id);
		locations.get(id).set(x, y);
	}
}
```

Map 내부에 들어 있는 값을 스레드 안전하고 변경가능한 SafePoint 클래스를 사용하고 있다. 

하지만 만약 차량의 위치에 대해 제약 사항을 추가해야 한다면 스레드 안전성을 해칠 수 있으며 앞선 방법으로는 충분하지 않다.

<br/>

<br/>

<br/>

## 4.4 스레드 안전하게 구현된 클래스에 기능 추가

자바의 기본 클래스 라이브러리에는 여러가지 유용한 기반 클래스가 많아서, 이미 만들어져 있는 클래스를 재사용하면 개발에 필요한 시간과 자원을 절약할 수 있고, 개발할 때 오류가 줄어들고, 유지비용 비용도 절감할 수 있다. 간혹 원하는 기능이 없을때는 필요한 기능을 구현해 추가하면서 스레드 안전성도 계속해서 유지하는 방법을 찾아야 할때도 있다. 

단일 연산 하나를 기존 클래스에 추가하고자 한다면 해당하는 단일 연산 메소드를 기존 클래스에 직접 추가하는 방법이 가장 안전하지만 외부 라이브러리를 사용하는 경우는 코드가 없거나, 고칠수 없을 수도 있고, 수정 가능하더라도 동기화 정책을 정확히 이해하고 써야 한다. 

또 다른 방법으로는 기존 클래스를 상속받는 방법인데, 이 방법은 기존 클래스를 외부에서 상속받아 사용할 수 있도록 설계했을 때나 사용할 수 있다. 또한 동기화를 맞춰야 할 대상이 두 개 이상의 클래스에 걸쳐 분산되기 때문에 문제가 생길 위험이 훨씬 많다. 

```jsx
// 예제 4.13 기존의 Vector 클래스를 상속받아 putIfAbsent 메소드를 추가
@ThreadSafe
public class BetterVector<E> extends Vector<E> {
	public synchronized boolean putIfAbsent(E x) {
		boolean absent = !contains(x);
		if(absent)
			add(x);
		return absent;
	}
}
```

<br/>

### 4.4.1 호출하는 측의 동기화

기존 클래스에 메소드를 추가하거나, 상속받은 하위 클래스에서 추가 기능을 구현하는 방법을 적용할 수 없는 경우가 있다. Collections.synchronizedList 메소드를 사용해 동기화시킨 ArrayList가 그렇다. 동기화된 ArrayList를 받아간 외부 프로그램은 받아간 객체가 동기화되었는지를 알 수 없기 때문이다. 클래스를 상속받지 않고도 클래스에 원하는 기능을 추가하는 또 다른 방법은 도우미 클래스를 따로 구현해서 추가 기능을 구현하는 방법이다. 아래는 이러한 방법의 잘못된 예시를 보여준다

```jsx
// 예제 4.14 목록에 없으면 추가하는 기능을 잘못 구현한 예
@NotThreadSafe
public class ListHelper<E> {
	public List<E> list = Collections.synchronizedList(new ArrayList<E>());
...
	public synchronized boolean putIfAbsent(E x) {
		boolean absent = !list.contains(x);
		if(absent)
			list.add(x);
		return absent;
	}
}
```

아무런 의미가 없는 락을 대상으로 동기화가 맞춰졌기 때문에 해당 코드는 제대로 작동하지 않는다. 

제 3의 도우미 클래스를 올바르게 구현하려면 클라이언트측 락client-side lock이나 외부 락external lock을 사용해 list가 사용하는 것과 동일한 락을 사용해야 한다. 아래의 예제에서는 클라이언트 측 락을 사용해 스레드 안전한 방법으로 구현했다.

```jsx
// 예제 4.15 클라이언트 측 락을 사용해 putIfAbsent 메소드를 구현
@ThreadSafe
public class ListHelper<E> {
	public List<E> list = Collections.synchronizedList(new ArrayList<E>());
...
	public boolean putIfAbsent(E x) {
		synchronized(list) { 
			boolean absent = !list.contains(x);
			if(absent)
				list.add(x);
			return absent;
		}
	}
}
```

이렇게 클라이언트 측 락 방법으로 제 3의 클래스를 구현하는 방법은 외부의 락을 가져다 쓰는것이기 때문에 훨씬 위험하기 때문에 충분히 주의를 기울여야 한다.

<br/>

### 4.4.2 클래스 재구성

기존 클래스에 새로운 단일 연산을 추가하고자 할 때 좀 더 안전하게 사용할 수 있는 방법이 있는데, 바로 재구성composition이다. 아래의 예제에서의 ImprovedList는 List클래스의 기능을 구현할 때는 ImprovedList 내부의 List 클래스 인스턴스가 갖고 있는 기능을 불러와 사용하고, 그에 덧붙여 putIfAbsent 메소드를 구현하고 있다. 

```jsx
// 예제 4.16 재구성 기법으로 putIfAbsent 메소드 구현
@ThreadSafe
public class ImprovedList<T> implements List<T> {
	private final List<T> list;
	public ImprovedList(List<T> list) { this.list = list; }

	public synchronized boolean putIfAbsent(T x) {
		boolean contains = list.contains(x);
		if(!contains)
			list.add(x);
		return !contains;
	}
	
	public synchronized void clear() { list.clear(); }
	// ... List 클래스의 다른 메소드로 clear와 비슷하게 구현
}
```

클래스 그 자체를 락으로 사용해 내부의 List 클래스가 스레드 안전한지 아닌지는 중요하지 않고 신경쓸 필요도 없다. 전체적인 성능의 측면에서는 약간 부정적인 영향이 있을 수 있지만, 클라이언트 측 락 보다 훨씬 안전하다. 

<br/>

<br/>

<br/>

## 4.5 동기화 정책 문서화하기

클래스의 동기화 정책에 대한 내용을 문서로 남기는 일은 스레드 안전성을 관리하는 데 있어 가장 강력한 방법 가운데 하나이다. 하지만 그럼에도 동기화와 관련된 충분한 정보를 얻지 못하는 경우가 많다. 

설계 단계에서 부터 동기화 정책을 정의하는 일을 시작하는 것이 가장 좋다. 동기화 정책을 구성하고 결정하고자 할 때에는 여러가지 사항을 고려해야한다. 어떤 변수를 volatile 지정할 지, 어떤 변수를 사용할 때는 락으로 막아야 할지, 어떤 변수를 불변 클래스로 만들지, 어떤 변수를 스레드에 한정시켜야 할지 등..

최소한 클래스가 스레드 안전성에 대해서 어디까지 보장하는지는 문서로 남겨야 하며 라이브러리 사용자가 아무렇게나 추측하는 위험한 상황을 만들지 않도록 해야 한다. 그러나 자바 플랫폼 라이브러리 문서를 포함해 요즘은 문서를 자세히 작성하지 않는다. 사용자들은 추측을 통해 개발을 하게 되고 그러면서 많은 문제가 발생하게 되는것이다. 

<br/>

<br/>