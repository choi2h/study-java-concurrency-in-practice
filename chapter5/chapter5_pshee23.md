# 5장 구성 단위

병렬 프로그래밍 과정에서 유용하게 사용할 수 있는 몇 가지 도구를 살펴볼 것이다. 그에 덧붙여 병렬 프로그램을 작성할 때 사용하기 좋은 몇 가지 디자인 패턴도 알아보자.


<br/>

## 5.1 동기화된 컬렉션 클래스

동기화되어 있는 컬렉션 클래스의 대표 주자는 Vector와 Hashtable이다. 이와 같은 클래스는 public으로 선언된 메소드를 클래스 내부에 캡슐화해 내부의 값을 한 번에 한 스레드만 사용할 수 있도록 제어 하면서 스레드 안전성을 확보하고 있다. 


<br/>

### 5.1.1 동기화된 컬렉션 클래스의 문제점

동기화된 컬렉션 클래스는 스레드 안전성을 확보하고 있기는 하나, 여러 개의 연산을 묶어 하나의 단일 연산처럼 활용해야 할 필요성이 항상 발생한다. 

두 개 이상의 연산을 묶어 사용하는 예를 들면 반복iteration(컬렉션 내부의 모든 항목을 차례로 가져다 사용), 이동navigation(특정한 순서에 맞춰 현재 보고 있는 항목의 다음 항목 위치로 이동), 없는 경우에만 추가(컬렉션 내부에 추가하고자 하는 값이 있는지 확인 하고 없는 경우만 새로운 값을 추가) 등이 있다. 동기화된 컬렉션을 사용하면 락, 동기화 기법 없이 모두 스레드 안전하다. 

하지만 여러 스레드가 해당 컬렉션 하나를 놓고 동시에 그 내용을 변경하려 한다면 컬렉션 클래스의 동작에 문제가 생길 수 있다. 

```java
 // 예제 5.1 올바르게 동작하지 않을 수 있는 상태의 메소드
public static Object getLast(Vector list) {
	int lastIndex = list.size() - 1; // 확인하는 과정
	return list.get(lastIndex); // 동작하는 형태
}

public static void deleteLast(Vector list) {
	int lastIndex = list.size() - 1; // 확인하는 과정
	list.remove(lastIndex); // 동작하는 형태
}
```

Vector 클래스의 메소드만을 사용하기 때문에 몇 개의 스레드가 동시에 메소드를 호출한다 해도 Vector 내부의 데이터는 깨지지 않는다. 하지만 위의 두 가지 메소드를 호출해 사용하는 외부 프로그램의 입장에서 보면 상황이 달라진다. 


<br/>

![그림5.1.png](5%E1%84%8C%E1%85%A1%E1%86%BC%20%E1%84%80%E1%85%AE%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%83%E1%85%A1%E1%86%AB%E1%84%8B%E1%85%B1%20319ef1801e774252bc6fbb361ac3d619/%25EA%25B7%25B8%25EB%25A6%25BC5.1.png)

위의 그림과 같이 getLast 메소드와 deleteLast 메소드가 절묘하게 겹쳐져 동작해      ArrayIndexOutOfBoundsException이 발생하게 된다. Vector의 입장에서는 스레드 안전성 문제가 없지만, 예외가 발생하게 되어 올바르지 않은 상황이 되었다. 

동기화된 컬렉션 클래스는 대부분 클라이언트 측 락을 사용할 수 있도록 만들어져있기 때문에 컬렉션 클래스가 사용하는 락을 함께 사용한다면 새로 추가하는 기능을 컬렉션 클래스에 들어 있는 다른 메소드와 같은 수준으로 동기화 시킬 수 있다. 동기화된 컬렉션 클래스는 컬렉션 클래스 자체를 락으로 사용해 내부의 전체 메소드를 동기화시키고 있다. 아래의 예제에서 확인해보자.

```java
// 예제 5.2 클라이언트 측 락을 활용해 getLast와 deleteLast를 동기화
public static Object getLast(Vector list) {
	synchronized(list) {
		int lastIndex = list.size() - 1; // 확인하는 과정
		return list.get(lastIndex); // 동작하는 형태
	}
}

public static void deleteLast(Vector list) {
	synchronized(list) {
		int lastIndex = list.size() - 1; // 확인하는 과정
		list.remove(lastIndex); // 동작하는 형태
	}
}
```



<br/>

아래와 같은 예제는 컬렉션 클래스에 들어 있는 모든 값을 반복적으로 가져오는 기능을 구현하고 있는데, 위에서 보았던 문제와 같은 모습을 볼 수 있다. 

```java
// 예제 5.3 ArrayIndexOutOfBoundsException이 발생할 수 있는 반복문
for (int i = 0; i < vector.size(); i++)
	doSomething(vector.get(i));
```

컬렉션 클래스에 구현되어 있는 반복 기능은 메소드들을 호출하는 사이에 변경 기능을 호출하지 않을 것이라는 어설픈 가정하에 만들어져 있다. 단일 스레드에서는 문제가 없겠지만, 여러 스레드에서 동작하게 되는 경우 문제가 발생하게 되는 것이다. 



<br/>

위와 같은 예시들이 예외 상황을 발생시킬 수 있지만, Vector클래스가 스레드 안전하지 않다는 의미는 아니다. Vector 내부의 데이터는 예외 상황이 발생하는 것과 무관하게 안전한 상태이며 클래스의 설계 방향에 의해 예외가 발생되는 것이다. 

반복문 예제에서도 클라이언트 측 락을 사용하면 동기화시킬 수 있으나, 성능적 측면에서 약간의 손해가 발생할 수 있다. 

```java
// 예제 5.4 클라이언트 측 락을 사용해 반복문을 동기화시킨 모습
synchronized (vector) {
	for (int i = 0; i < vetor.size(); i++)
		doSomething(vector.get(i));
}
```

반복문 실행하는 동안 동기화를 진행한다면 반복문 실행 중에는 Vector 클래스 내부의 값을 변경하는 모든 스레드가 대기 상태에 들어가는 동시 작업을 모두 막아버리기 때문에 여러 스레드가 동시에 동작하는 병렬 프로그램의 큰 장점을 잃어버리게 된다.


<br/>
<br/>

### 5.1.2 Iterator와 ConcurrentModificationException

새로 추가된 컬렉션 클래스 역시 다중 연산을 사용할 때에 발생하는 문제점을 해결하지는 못한다. Iterator를 사용해 컬렉션 클래스 내부의 값을 차례로 읽어다 사용한다 해도, 반복문이 실행되는 동안 다른 스레드가 컬렉션 클래스 내부의 값 추가, 제거 등의 변경 작업을 시도할 때 발생할 수 있는 문제를 막아주지는 못한다. 

대신, 즉시 멈춤fail-fast의 형태로 반응하게 되어 있는데, 즉시 멈춤이란 반복문을 실행하는 도중에 컬렉션 클래스 내부의 값을 변경하는 상황이 포착되면 그 즉시 ConcurrentModificationException 예외를 발생시키고 멈추는 처리 방법이다. 이는 쉽게 사용한다기 보다는, 멀티스레드 관련 오류가 있다는 경고 정도에 해당한다고 보는게 좋다. 

Iterator 컬렉션 클래스는 내부에 값 변경 횟수를 카운트 하는 변수를 마련해두고, 반복문이 실행되는 동안 변경 횟수 값이 바뀌면 hasNext 메소드나 next 메소드에서 ConcurrentModificationException을 발생시킨다. 더군다나 변경 횟수 조회 부분이 동기화 되어있지 않아, 세는 과정에서 스테일 값을 사용하게 될 가능성도 있다. 이는 성능을 떨어뜨릴 수 있기 때문에 변경 작업 확인 기능에 정확한 동기화 기법을 적용하지 않았다고 볼 수 있다. 



<br/>

아래의 예제에서는 for-each 반복문을 사용해 컬렉션 클래스의 값을 차례로 읽어들이는 코드가 나타나 있다. 

```java
// 예제 5.5 Iterator를 사용해 List 클래스의 값을 반복해 뽑아내는 모습[Not so good]
List<Widget> widgetList = Collections.synchronizedList(new ArrayList<Widget>());
...
// ConcurrentModificationException 발생 할 수 있다.
for (widget w : widgetList)
	doSomething(w);
```

위의 코드를 javac로 컴파일할 때, Iterator를 사용하면서 위의 Exception 발생 상황이 만들어진다. 반복문 전체를 락으로 동기화 시키는 방법밖에 없으나, 컬렉션에 값이 많은 경우 락으로 잡혀 있는 시간이 길어 대기 시간이 길어진다. 또한 예제 5.4와 같은 상황에서 doSomething 메소드가 실행할 때 또 다른 락을 확보해야 한다면, 데드락이 발생할 가능성도 높아진다.

반복문 실행 중 락을 걸어둔 것과 비슷한 효과를 내려면 clone 메소드로 복사본을 만들어 사용할 수 있다. 복사한 사본은 특정 스레드에 한정되어 있으므로 반복문이 실행되는 동안 다른 스레드에서 사본을 건드리기 어려워 Exception이 발생하지 않는다. 

clone도 결국 작업 시간이 어느정도 필요하기 떄문에 컬렉션 항목의 수, 실행 작업의 작업 시간, 반복 기능의 요청 횟수 응답성, 실행 속도 등을 고려해서 적절하게 적용해야 한다.


<br/>
<br/>

### 5.1.3 숨겨진 Iterator

Iterator사용 시 락으로 동기화 하면 Exception 발생하지 않도록 제어할 수 있지만, 컬렉션을 공유해 사용하는 모든 부분에서 동기화를 맞춰야하는 것을 잊으면 안된다. 

일반적이지 않지만 다음 예제에서는 Iterator가 숨겨져 있는 경우를 볼 수 있다. 

```java
// 예제 5.6 문자열 연결 연산 내부에 Iterator가 숨겨져 있는 상황. [Bad Example]
public class HiddenIterator {
	@GuardedBy("this")
	private final Set<Integer> set = new HashSet<Integer>();

	public synchronized void add(Integer i) { set.add(i); }
	public synchronized void remove(Integer i) { set.remove(i); }

	public void addTenThings() {
		Random r = new Random();
		for (int i = 0; i < 10; i++)
			add(r.nextInt());
		System.out.println(**"DEBUG: added ten elements to " + set**);
	}
}
```

위의 예제에서 iterator 메소드로 Iterator를 뽑아 사용하는 부분은 없지만, println에서 문자열 두 개를 + 연산으로 견결하는데 컴파일러는 이 문장을 StringBuilder.append(Object) 메소드를 사용하는 코드로 변환한다. 그 과정에서 컬렉션 클래스의 toString 메소드를 호출하고 해당 메소드 내부에는 iterator 메소드를 호출해 문자열을 만들어내고 있다. 즉, set 변수의 Iterator를 찾아 사용하기 때문에 ConcurrentModificationException이 발생할 가능성이 생긴다. 

스레드 안전성을 확보하려면 set 변수 사용할 때 락을 확보, 동기화 시켜야 한다. 하지만 디버깅 메세지를 출력하기 위해 락을 사용하는건 성능면에서 적절하진 않다. 

상태 변수와 동기화를 맞춰주는 락이 멀리 떨어져 있을수록 동기화를 맞춰야 한다는 필요성을 잊기 쉽다. HashSet을 직접 사용하지 말고 synchronizedSet 메소드로 동기화된 컬렉션을 사용하면 동기화가 이미 맞춰져 있기 떄문에 Iterator와 관련된 문제가 발생하지 않을 것이다. 

toString 뿐만 아니라 컬렉션 클래스의 hashCode, equals, containsAll, removeAll, retainAll 등의 메소드, 컬렉션 클래스를 넘겨받는 생성 메소드 등도 내부적으로 iterator를 사용한다. 


<br/>
<br/>
<br/>

## 5.2 병렬 컬렉션

동기화된 컬렉션 클래스는 컬렉션의 내부 변수에 접근하는 통로를 일련화해서 스레드 안전성을 확보했다. 하지만 여러 스레드가 한꺼번에 동기화된 컬렉션을 사용하려고 하면 동시 사용성을 상당 부분 손해를 보게 되었다. 

하지만 병렬 컬렉션은 여러 스레드에서 동시에 사용할 수 있도록 설계되어 있다. 자바 5.0 부터는 다음과 같은 병렬 컬렉션이 추가되었다. 

| ConcurrentHashMap | 해시 기반의 HashMap을 대치하면서 병렬성을 확보 |
| --- | --- |
| CopyOnWriteArrayList | 가되어 있는 객체 목록을 반복시키며 열람하는 List 클래스의 하위 클래스 |
| ConcurrentMap 인터페이스 | 기존에 없는 경우만 추가하는put-if-absent연산, 대치replace 연산, 조건부 제거conditional remove 연산을 정의 |
| Queue 컬렉션 인터페이스 | 작업할 내용을 순서대로 쌓아 놓는 구조
Queue 인터페이스를 구현하는 클래스(ConcurrentLinkedQueue, PriorityQueue)
Queue인터페이스에 정의되어 있는 연산은 동기화를 맞추기 위해 대기하는 부분이 없다. (null 리턴) |
| BlockingQueue | Queue를 상속받음
큐에 항목을 추가, 뽑아낼 때 상황에 따라 대기할 수 있도록 구현 |
| ConcurrentSkipListMap | 자바 6에서 SortedMap 클래스의 병렬성을 높임 |
| ConcurrentSkipListSet | 자바 6에서 SortedSet 클래스의 병렬성을 높임 |



<br/>

### 5.2.1 ConcurrentHashMap

동기화된 컬렉션 클래스는 각 연산을 수행하는 시간 동안 항상 락을 확보하고 있어야 하지만 HashMap.get 메소드나 List.contains와 같은 몇가지 연산은 훨씬 많은 일을 하게 되는 상황이 있을 수 있다. HashMap.get에서는 최악의 상황에서 해시 테이블 한쪽에 데이터가 치우쳐서 전체를 다 뒤져봐야 할수도 있으며, List.contains에서는 최악의 상황에서 모든 객체를 대상으로 equals 메소드를 호출하게 된다. 

ConcurrentHashMap은 HashMap과 같이 해시를 기반으로 하는 Map이나, 내부적으로는 이전과 다른 동기화 기법을 사용해 병렬성과 확장성을 개선했다. 

이전에는 모든 연산에서 하나의 락만 사용하여 특정 시점에 하나의 스레드만이 해당 컬렉션을 사용할 수 있었으나, ConcurrentHashMap은 락 스트라이핑lock striping을 사용해 여러 스레드 공유 상태에 잘 대응했다.(11.4.3절 참조)

ConcurrentHashMap이 만들어 낸 Iterator는 ConcurrentModificationException을 발생시키지 않아서 반복문 사용시 따로 락으로 동기화해야 할 필요가 없다. 즉시 멈춤대신 미약한 일관성 전략을 취했는데, 이 전략은 반복문과 동시에 컬렉션의 내용을 변경한다 해도 Iterator를 만들었던 시점의 상황대로 반복을 계속 할 수 있다. 또한 반드시 보장되진 않으나 Iterator 만든 시점 이후에 변경된 내용을 반영해 동작할 수도 있다.

그럼에도 신경써야 할 부분도 있는데, 병렬성 문제 떄문에 size, isEmpty 메소드의 의미가 약간 약해졌다. size는 리턴하는 시점에 실제 객체의 수가 바뀌었을 수도 있어 추정값을 알게 된다. get, put 등의 핵심 연산의 병렬성과 성능을 높이기 위해서라면 약간 변할 수밖에 없는것이다. 

동기화된 Map에는 있으나 ConcurrentHashMap에는 맵을 독점적으로 사용할 수 있도록 막아버리는 기능은 지원하지 않는다. Hashtable, synchronizedMap을 사용해 막을 수 있으나 여러 스레드 동시 접근 하는 Map인 만큼 의미가 없는 단점이긴 하다. 상황에 맞춰 ConcurrentHashMap을 사용할지 Hashtable/SynchronizedMap을 사용할지 판단하자.



<br/>

### 5.2.2 Map 기반의 또 다른 단일 연산

ConcurrentHashMap 클래스는 독점적 사용 락이 없기 때문에 여러 개의 단일 연산을 모아 새로운 단일 연산을 만드는 ‘없을 경우에만 추가하는’ 기능을 만들려고 할때 클라이언트 측 락 기법을 활용할 수 없다(4.4.1절 참고). 

하지만 ConcurrentHashMap 클래스에는 ‘없을 경우에만 추가put-if-absent’, ‘동일한 경우에만 제거remove-if-equal’, ‘동일한 경우에만 대치replace-if-equal’ 연산이 구현되어 있으니 이 외의 다른 기능이 필요하다면 ConcurrentMap을 사용하자.



<br/>

### 5.2.3 CopyOnWriteArrayList

CopyOnWriteArrayList 클래스는 동기화된 List 클래스보다 병렬성을 높이고자 만들어졌으며, List에 있는 값을 Iterator로 불러 사용할 때 List 전체에 락을 걸거나, 복제할 필요가 없다. 

불변 객체를 외부 공유하면 여러 스레드 동시 사용시에도 동기화 작업이 필요없다는 개념을 바탕으로, 컬렉션은 항상 내용이 바뀌므로 내용이 변경될때마다 복사본을 만들어서 안전성을 확보했다. 

Iterator를 뽑아내서 사용할 때 뽑아내는 시점의 컬렉션 데이터를 기준으로 반복하며, 반복하는 동안 컬렉션에 추가, 삭제 되어도 반복문 데이터과 상관 없는 데이터에 반영이 되므로 반복문 동시 사용성에 문제가 없고, ConcurrentModificationException이 발생하지 않는다. 

```java
// 예제 5.7 ConcurrentMap 인터페이스
public interface ConcurrentMap<K,V> extends Map<K,V> {
	// key라는 키가 없는 경우에만 value 추가
	V putIfAbsent(K key, V value);

	// key라는 키가 value값을 갖고 있는 경우 제거
	boolean remove(K key, V value);

	// key라는 키가 oldValue값을 갖고 있는 경우 newValue로 치환
	boolean replace(K key, V oldValue, V newValue);

	// key라는 키가 들어 있는 겨웅에만 newValue로 치환
	V replace(K key, V newValue);
}
```

물론 컬렉션의 데이터가 변경될 때마다 복사본을 만들어내기 때문에 성능의 측면에서 손해가 있으며, 컬렉션 데이커가 크다면 손실이 있을 수 있다. 따라서 변경 작업보다 반복문으로 읽어내는 일이 훨씬 빈번한 경우에 효과적이다. 예를 들어 이벤트 처리 시스템에서 이벤트 리스너를 관리하는 부분에서 유용하게 쓰일 수 있다. 리스너를 등록, 해제하는 기능의 빈도가 리스너 목록 내용을 반복문으로 호출하는 기능보다 빈도가 낮기 때문에 적당한 상황이다.  



<br/>
<br/>
<br/>

## 5.3 블로킹 큐와 프로듀서-컨슈머 패턴

블로킹 큐blocking queue는 put과 take라는 핵심 메소드가 있으며 offer와 poll이라는 메소드도 있다. 큐가 가득 차 있다면 put은 공간이 생길때까지 대기하며 비어있으면 take 메소드는 뽑을 값이 생길때까지 대기한다. 

블로킹 큐는 프로듀서-컨슈머producer-consumer 패턴을 구현할 때 사용하게 좋은데, 이 패턴은 ‘해야 할 일’ 목록을 가운데에 두고, 작업을 만들어 내는 주체와 작업을 처리하는 주체를 분리시키는 설계 방법이다. 이는 개발 과정을 좀 더 명확하게 단순화시킬 수 있고, 작업 생성 부분과 처리 부분이 각각 담당하는 부하를 조절 할 수 있는 장점이 있다. 

이 패턴을 구현할 때 블로킹 큐를 사용하는 경우가 많은데, 프로듀서는 작업을 새로 만들어 큐에 쌓고, 컨슈머는 큐에 쌓여 있는 작업을 가져다 처리하는 구조이다. 서로에 대해서 전혀 신경 쓰지 않고 각자 맡은 일만 하면 되는 구조이다. 

프로듀서나 컨슈머는 상대적인 개념이라 컨슈머의 형태로 동작하는 객체라 해도 다른 측면에서는 프로듀서의 역할을 하고 있을 수 있어서 두번 째 블로킹 큐가 더 생길 수도 있다. 

블로킹 큐를 사용하면 값이 들어올 때까지 take 메소드가 알아서 멈추고 대기하기 때문에 컨슈머 코드를 작성하기가 편리하며, 큐의 크기에 제한을 두어서 빈 공간이 생길 때까지 put 메소드가 대기하게 두어서 프로듀서 코드 또한 작성하기가 간편해진다. 하지만 이러한 대기 상황으로 인해 하드웨어 자원을 효율적으로 사용하지 못하는 것으로 판단할 수도 있다. 


<br/>

또 다른 메소드인 offer 메소드는 큐에 값을 넣을 수 없을 때 대기하지 않고 바로 공간이 모자라 추가할 수 없다는 오류를 알려준다. 이를 활용해 프로듀서가 작업을 많이 만드는 과부하 상태를 효과적으로 처리할 수 있다. 

프로듀서-컨슈머 패턴은 각각 코드는 큐를 기준으로 분리되어 있지만 동작 자체는 간접적으로 연결되어 있다. 작업 큐에 제한을 둘 필요가 없을 것이라고 생각하지만 그렇지 않으므로 블로킹 큐를 사용해 설계 과정 부터 프로그램에 자원 관리 기능을 추가하자. 대다수의 경우는 블로킹 큐만 사용해도 원하는 기능을 구현할 수 있으나, 쉽게 적용할 수 없는 경우는 세마포어Semaphore를 사용해 사용하기 적합한 데이터 구조를 만들어야 한다. 



<br/>

자바 클래스 라이브러리에는 BlockingQueue 인터페이스를 구현한 클래스가 몇가지 있다. 우선 LinkedBlockingQueue와 ArrayBlockingQueue가 있으며 FIFO 형태의 큐이다. LinkedList와 ArrayList에 각각 대응되며, 병렬 프로그램 환경에서는 LinkedList, ArrayList 보다 성능이 좋다.  

PriorityBlockingQueue 클래스는 우선 순위를 기준으로 동작하는 큐이고, FIFO가 아닌 다른 순서로 큐의 항목을 처리해야 하는 경우에 손쉽게 사용할 수 있다. 기본 정렬 순서나 Comparator 인터페이스를 사용해 정렬시킬 수 있다. 

SynchronousQueue 클래스는 큐에 항목이 쌓이지 않으며, 큐 내부에 값을 저장할 수 있도록 공간을 할당하지도 않는다. 대신 큐에 값을 추가하려는 스레드나 값을 읽어가려는 스레드의 큐를 관리한다. 프로듀서와 컨슈머가 직접 데이터를 주고받을 때까지 대기하는것이라 데이터가 넘어가는 순간이 굉장히 짧아진다. 



<br/>
<br/>

### 5.3.1 예제 : 데스크탑 검색

데스크탑 검색 프로그램은 로컬 하드 디스크에 들어 있는 문서를 전부 읽어들이면서 나중에 검색하기 좋게 색인을 만들어 두는 작업을 한다. 

예제 5.8의 FileCrawler 프로그램은 디렉토리 계층 구조를 따라가면서 검색 대상 파일이라고 판단되는 파일을 작업 큐에 모두 쌓아 넣는 프로듀서 역할을 담당한다. Indexer 프로그램은 작업 큐에 쌓여 있는 파일 이름을 뽑아내어 해당 파일의 내용을 색인하는 컨슈머의 역할을 맡고 있다. 

```java
// 예제 5.8 프로듀서-컨슈머 패턴을 활용한 데스크탑 검색 애플리케이션의 구조
public class FileCrawler implements Runnable {
	private final BlockingQueue<File> fileQueue;
	private final FileFilter fileFilter;
	private final File root;
	...
	public void run() {
		try {
			crawl(root);
		} catch (InterruptedException e) {
			Thread.currentThread().interrupt();
		}
	}

	private void crawl(File root) throws InterruptedException {
		File[] entries = root.listFiles(fileFilter);
		if(entries != null) {
			for(File entry : entries)
				if(entry.isDirectory())
					crawl(entry);
				else if(!alreadyIndexed(entry))
					fileQueue.put(entry);
		}
	}
}

public class Indexer implements Runnable {
	private final BlockingQueue<File> queue;
	
	public Indexer(BlockingQueue<File> queue) {
		this.queue = queue;
	}

	public void run() {
		try {
			while(true)
				indexFile(queue.take());
		} catch (InterruptedException e) {
			Thread.currentThread().interrupt();
		}
	}
}
```

두 개의 클래스로 분리해서 코드 자체의 가독성을 높이고, 재사용성도 높였다. 성능 측면에서도 이득인데 서로 독립적으로 실행되므로 프로듀서와 컨슈머의 기능을 단일 스레드에서 순차적으로 실행하는 것보다 성능이 크게 높아질 수 있다. 



<br/>

예제 5.9의 프로그램은 문서 파일을 찾아내는 기능과 파일의 내용을 색인하는 모듈 여러 개를 각각의 스레드를 통해 동작시킨다. 

```java
// 예제 5.9 데스크탑 검색 애플리케이션 동작시키기
public static void startIndexing(File[] roots) {
	BlockingQueue<File> queue = new LinkedBlockingQueue<File>(BOUND);
	FileFilter filter = new FileFilter() {
		public boolean accept(File file) { return true; }
	};

	for (File root : roots)
		new Thread(new FileCrawler(queue, filter, root)).start();

	for (int i = 0; i < N_CONSUMERS; i++)
		new Thread(new Indexer(queue)).start();
}
```

색인을 담당하는 컨슈머 스레드는 계속해서 작업을 기다리느라 종료되지 않는 상황이 발생한다. 이는 7장에서 다시 살펴보도록 한다. 



<br/>
<br/>

### 5.3.2 직렬 스레드 한정

블로킹 큐 관련 클래스는 모두 프로듀서 스레드에서 객체를 가져와 컨슈머 스레드에 넘겨주는 과정이 동기화 되어 있다. 프로듀서-컨슈머 패턴과 블로킹 큐는 가변 객체mutable object를 사용할 때 객체의 소유권을 프로듀서에서 컨슈머로 넘기는 과정에서 직렬 스레드 한정serial thread confinement 기법을 사용한다. 객체를 안전한 방법으로 공개하면 객체에 대한 소유권을 이전 할 수 있으며 프로듀서 스레드는 소유권을 완전히 잃고 컨슈머 스레드가 객체에 대한 유일한 소유권을 가지게된다. 객체 풀object pool은 직렬 스레드 한정 기법을 잘 활용하는 예인데, 풀에서 소유하고 있던 객체를 외부 스레드에게 빌려주는 일이 본업이기 때문이다. 

가변 객체의 소유권을 이전해야 할 필요가 있다면, 위에서 설명한 것과 다른 객체 공개 방법을 사용할 수도 있다. 하지만 항상 소유권을 이전받는 스레드는 단 하나여야 한다는 점을 주의하고, 블로킹 큐를 사용하면 이런 점을 정확하게 지킬 수 있다. 



<br/>
<br/>

### 5.3.3 덱, 작업 가로채기

자바 6.0에는 Deque(덱), BlockingDeque라는 컬렉션이 추가 되었다. 각각 Queue, BlockingQueue를 상속받은 인터페이스이며, Deque는 앞과 뒤 어느쪽에도 객체를 삽입, 제거할 수 있도록 준비된 큐이고, Deque를 상속받은 클래스로는 ArrayDeque와 LinkedBlockingDeque이 있다.

프로듀서-컨슈머 패턴이 블로킹 큐를 가져다 쓴것처럼 작업 가로채기work stealing라는 패턴 적용에는 덱을 그대로 가져다 사용할 수 있다. 작업 가로채기 패턴에서는 모든 컨슈머가 각자의 덱을 갖고 있으며, 컨슈머가 자신의 덱 작업을 다 처리하면 다른 컨슈머의 덱에 있는 작업에서 맨 뒤에 추가된 작업을 가로채 가져올 수 있다. 

작업 가로채기 패턴은 프로듀서-컨슈머 패턴(컨슈머가 하나의 큐를 바라보면서 작업 경쟁을 함)과 다르게 작업 패턴이 규모가 큰 시스템을 구현하기에 적당하다. 다른 컨슈머의 작업을 가져올 때도 원래 소유자는 맨 앞, 다른 컨슈머는 맨 뒤 작업을 가져오기 때문에 경쟁이 없다. 

작업 가로채기 패턴은 컨슈머가 프로듀서의 역할도 갖고 있는 경우에 적용하기 좋은데, 하나의 작업을 처리하면 더 많은 작업이 생기는 케이스와 같은 경우에 자신의 덱에 새로운 작업을 추가하고, 자신의 덱이 비었을 경우 다른 작업 스레드 덱에서 가져와서 쉬는 스레드가 없게 굴러간다.



<br/>
<br/>
<br/>

## 5.4 블로킹 메소드, 인터럽터블 메소드

스레드가 블록되면 동작이 멈춰진 다음 블록된 상태(BLOCKED, WAITING, TIMED_WAITING) 가운데 하나를 갖게 된다. 블로킹 연산은 멈춘 상태에서 특정한 신호를 받아야 계속해서 실행 할 수 있는 연산이며, 외부 신호가 확인 되면 스레드의 상태가 다시 RUNNABLE 상태로 넘어가고 CPU를 사용할 수 있게 되는것이다. 

BlockingQueue 인터페이스의 put, take 메소드는 Thread.sleep 메소드와 같이 InterruptedException을 발생시킬 수 있으며 이는 해당 메소드가 블로킹 메소드라는 의미이고, 메소드에 인터럽트가 걸리면 해당 메소드는 대기 중인 상태에서 풀려나고자 노력한다. 

Thread 클래스에는 스레드 중단을 위한 interrupt 메소드를 제공하며, 중단된 상태인지 확인할 수 있는 불린값이 제공된다. 인터럽트는 스레드가 서로 협력해서 실행하기 위한 방법이며, 스레드에 인터럽트 건다는 것은 즉시 중지가 아닌 실행 중지 ‘요청’일 뿐이라 인터럽트 걸린 스레드는 적절한 때에 작업이 멈추어진다. 

프로그램이 호출하는 메소드에 InterruptedException이 발생할 수 있는 메소드가 있으면 그 메소드를 호출하는 메소드 역시 블로킹 메소드이다. 이에 대한 대처 방법을 마련해둬야 하며, 라이브러리 형태의 코드라면 일반적으로 두 가지 방법이 있다. 

- InterruptedExceptiond을 전달 : 받아낸 Exception을 호출한 메소드에게 넘기는 방법. catch하지 않거나, 호출한 메소드로 throw 하는 방법
- 인터럽트를 무시하고 복구 : 특정 상황에서는 Exception throw가 안되는데(Runnable 인터페이스 구현 시) 이 경우 catch시 현재 스레드의 interrupt 메소드를 호출해 인터럽트 상태를 설정해, 상위 호출 메소드가 인터럽트 상황이 발생했음으로 알도록 하는것이다(예제 5.10)

```java
// 예제 5.10 인터럽트가 발생했음을 저장해 인터럽트 상황을 잊지 않도록 한다. 
public class TaskRunnable implements Runnable {
	BlockingQueue<Task> queue;
	...
	public void run() {
		try {
			processTask(queue.take());
		} catch (InterruptedException e) {
			// 인터럽트가 발생한 사실을 저장
			Thread.currentThread().interrupt();
		}
	}
}
```

위의 방법들로 대부분의 경우에 대응할 수 있으나 InterruptedException을 처리함에 있어서 하지 말아야 할 일이 한가지 있다. catch하고는 무시하고 대응하지 않는 것이다. 대응하지 않으면 인터럽트 발생 증거를 인멸하는것이며, 상위 메소드가 대응, 조취할 수 있는 기회를 주지 않는것이다. 발생한 Exception을 전파하지 않는 경우는 Thread 클래스를 직접 상속하는 경우뿐이다. 



<br/>
<br/>
<br/>

## 5.5 동기화 클래스

상태 정보를 사용해 스레드 간의 작업 흐름을 조절할 수 있도록 만들어진 모든 클래스를 동기화 클래스synchronizer라고 한다. (블로킹 큐, 세마포어, 배리어, 래치 등). 

모든 동기화 클래스는 구조적인 특징을 갖고 있다. 

- 동기화 클래스에 접근하려는 스레드가 어느 경우에 통과하고 어느 경우에는 대기하도록 멈추게 해야 하는지를 결정하는 상태 정보
- 위의 상태를 변경할 수 있는 메소드를 제공
- 동기화 클래스가 특정 상태에 진입할 때 까지 효과적으로 대기할 수 있는 메소드도 제공



<br/>

### 5.5.1 래치

래치는 스스로가 터미널terminal 상태에 이를 때까지의 스레드가 동작하는 과정을 늦출 수 있도록 해주는 동기화 클래스이다. 래치는 일종의 관문과 같은 형태로 동작하며 특정한 단일 동작이 완료되기 이전에는 어떤 기능도 동작하지 않도록 막아내며 한번 열리면 열린 상태로 유지된다. 다음과 같은 상황에서 쓰일 수 있다. 

- 특정 자원을 확보하기 전에는 작업을 시작하지 말아야 하는 경우
- 의존성을 갖고 있는 다른 서비스가 시작하기 전에는 특정 서비스가 실행되지 않도록 막아야 하는 경우
- 특정 작업에 필요한 모든 객체가 실행할 준비를 갖출 때까지 기다리는 경우에도 사용

CountDownLatch는 위에서 소개한 모든 경우에 쉽게 적용할 수 있는 유연한 구조를 갖고 있는데, 하나 또는 둘 이상의 스레드가 여러 개의 이벤트가 일어날 때까지 대기할 수 있도록 되어 있다. 래치의 상태는 양의 정수 값으로 카운터를 초기화하며 이 값은 대기하는 동안 발생해야 하는 이벤트의 건수이다. 


<br/>

예제 5.11의 클래스에서는 래치를 사용하는 두 가지 경우를 볼 수 있다. 

```java
// 예제 5.11 CountDownLatch를 사용해 스레드의 실행과 종료를 확인해 전체 실행 시간을 확인한다.
public class TestHarness {
	public long timeTasks(int nThreads, final Runnable task) throws InterruptedException {
		final CountDownLatch startGate = new CountDownLatch(1);
		final CountDownLatch endGate = new CountDownLatch(nThreads);

		for(int i = 0; i < nThreads; i++) {
			Thread t = new Thread() {
				publid void run() {
					try {
						startGate.wait();
						try {
							task.run();
						} finally {
							endGate.countDown();
						}
					} catch (InterruptedException ignored) { }
				}
			};
			t.start();
		}

		long start = System.nanoTime();
		startGate.countDown();
		endGate.await();
		long end = System.nanoTime();
		return end-start;
	}
}
```

시작하는 관문은 내부 카운트가 1로 초기화 되고, 종료하는 관문은 내부 카운트가 전체 스레드의 개수에 해당하는 값으로 초기화된다. 

작업 스레드가 시작되면 가장 먼저 관문이 열리기를 기다리는 일을 시작한다. 이 과정을 통해 특정 이벤트가 발생한 이후에야 각 작업 스레드가 동작하도록 제어할 수 있다.  작업 스레드가 작업을 마치고 마지막엔 종료하는 관문의 카운트를 감소시킨다. 이 클래스에서는 래치를 사용해 메인스레드에서 모든 작업 스레드가 동시에 작업을 시작하도록 제어해서 n개의 스레드가 동시에 동작할 때 걸리는 전체 작업 시간을 정확히 확인할 수 있게 했다. 


<br/>
<br/>

### 5.5.2 FutureTask

FutureTask 역시 래치와 비슷한 형태로 동작하며 FutureTask가 나타내는 연산 작업은 Callable 인터페이스를 구현하도록 되어 있으며, 시작 전 대기, 시작됨, 종료됨과 같은 세 가지 상태를 가질 수 있다. 종료된 상태는 정상적인 종료, 취소, 예외상황 발생과 같이 연산이 끝나는 모든 종류의 상태를 의미한다. 한번 종료 상태가 되면 더이상 상태가 바뀌는 일은 없다.

Future.get 메소드의 동작 모습도 실행 상태에 따라 다른데, FutureTask 작업이 종료됐다면 get 메소드는 그 결과를 즉시 알려준다. 종료 상태에 이르지 못했다면 get 메소드는 작업이 종료 상태에 이를 때까지 대기하고, 종료된 이후에 연산 결과, 예외 상황을 알려준다. 연산 실행 스레드에서 만들어낸 결과 객체를 실행시킨 스레드에게 객체 안전한 공개 방법으로 넘겨준다. 

FutureTask는 Executor 프레임웍에서 비동기적인 작업을 실행하고자 할 때 사용하며, 기타 시간이 많이 필요한 모든 작업이 있을 때 실제 결과가 필요한 시점 이전에 미리 작업을 실행시켜두는 용도로 사용한다. 



<br/>

예제 5.12의 클래스는 FutureTask를 사용해 결과 값이 필요한 시점 이전에 시간이 많이 걸리는 작업을 미리 실행시켜둔다. 

```java
// 예제 5.12 FutureTask를 사용해 추후 필요한 데이터를 미리 읽어들이는 모습
public class Preloader {
	private final FutureTask<ProductInfo> future = new FutureTask<ProductInfo>(new Callable<ProductInfo>() {
		public ProductInfo call() throws DataLoadException {
			return loadProductInfo();
		}
	});
	private final Thread thread = new Thread(future);
	public void start() { thread.start(); }

	public ProductInfo get() throws DataLoadException, InterruptedException {
		try {
			return future.get();
		} catch (ExecutionException e) {
			Throwable cause = e.getCause();
			if(cause instanceof DataLoadException)
				throw (DataLoadException) cause;
			else
				throw launderThrowable(cause);
		}
	}
}
```

Preloader를 호출한 프로그램에서 실제 제품 정보가 필요한 시점이 되면 Preloader.get 메소드를 호출하면 되는데, get을 호출할 때 제품 정보를 모두 가져온 상태였다면 즉시 ProductInfo를 알려줄 것이고, 아직 데이터를 가져오는 중이라면 작업을 완료할 때까지 대기하고 결과를 알려준다. 



<br/>

Callable의 내부 작업에서 어떤 예외를 발생시키던 같에 그 내용은 Future.get 메소드에서 ExecutionException으로 한 번 감싼 다음 다시 throw 한다. Exception 발생 원인은 세가지 가운데 하나여야 하는데, 첫 번째는 Callable이 던지는 예외, 두 번째는 RuntimeException, 세 번째는 Error이다. Preloader에서는 이 세가지 경우를 구분해 처리해야 하지만 편의상 예외를 처리하는 복잡한 내용을 하나로 묶어 둔 다음 예제의 launderThrowable라는 유틸리티 메소드를 사용하기로 한다. 

```java
// 예제 5.13 Throwable을 RuntimeException으로 변환
/** 변수 t의 내용이 Error라면 그대로 throw한다. 
 *  변수 t의 내용이 RuntimeException이라면 그대로 리턴한다.
 *  다른 모든 경우에는 IllegalStateException을 throw 한다.
 */
public static RuntimeException launderThrowable(Throwable t) {
	if (t instanceof RuntimeException)
		return (RuntimeException) t;
	else if (t instanceof Error)
		throw (Error) t;
	else
		throw new IllegalStateException("RuntimeException이 아님", t);
}
```



<br/>
<br/>

### 5.5.3 세마포어

카운팅 세마포어 counting semaphore는 특정 자원이나 특정 연산을 동시에 사용하거나 호출할 수 있는 스레드의 수를 제한하고자 할 때 사용한다. 카운팅 세마포어의 이런 기능을 활용하면 자원 풀pool이나 컬렉션의 크기에 제한을 두고자 할 때 유용하다. 

세마포어 클래스는 가상의 퍼밋permit을 만들어 내부 상태를 관리하며, 세마포어 생성할 때 생성 메소드에 최초 생성할 퍼밋의 수를 넘겨준다. 외부 스레드는 퍼밋을 요청해 확보하거나, 이전에 확보한 퍼밋을 반납할 수도 있다. 현재 사용할 수 있는 남은 퍼밋이 없는 경우 대기한다. 


<br/>

세마포어는 데이터베이스 연결 풀과 같은 자원 풀에서 요긴하게 쓰이는데, 자원 풀을 만들 때, 모든 자원을 빌려주고 남아 있는 자원이 없을 때 요청이 들어오는 상황 구현 시, acquire를 호출해 퍼밋을 확보하다가, 자원 반납 시에는 release를 호출해 퍼밋도 반납하고, 자원이 없는 경우에는 acquire메소드가 대기하고 있게 된다. 

세마포어를 사용하면 어떤 클래스라도 크기가 제한된 컬렉션 클래스로 활용할 수 있다. 예제 5.14에서 세마포어는 해당하는 컬렉션 클래스가 가질 수 있는 최대 크기에 해당하는 숫자로 초기화한다. 

```java
// 예제 5.14 세마포어를 사용해 컬렉션의 크기 제한하기
public class BoundedHashSet<T> {
	private final Set<T> set;
	private final Semaphore sem;

	public BoundedHashSet(int bound) {
		this.set = Collections.synchronizedSet(new HashSet<T>());
		sem = new Semaphore(bound);
	}

	public boolean add(T o) throws InterruptedException {
		sem.acquire(); // 객체를 내부 데이터에 추가 전 acquire를 호출해 추가할 여유가 있는지 확인
		boolean wasAdded = false;
		try {
			wasAdded = set.add(o);
			return wasAdded;
		}
		finally {
			if(!wasAdded) // add가 실제로 값을 추가하지 못했다면, release 호출해 퍼밋 반납
				sem.release();
		}
	}

	public boolean remove(Object o) {
		boolean wasRemoved = set.remove(o); 
		if(wasRemoved) // 객체 삭제 한 경우 퍼밋 반납
			sem.release();
		return wasRemoved;
	}
}
```

BoundedHashSet의 내부에서 사용하는 Set은 크기가 제한되어 있다는 사실조차 알 필요가 없다. 크기와 관련된 내용은 모두 BoundedHashSet에서 세마포어를 사용해 관리하기 때문.



<br/>
<br/>

### 5.5.4 배리어

배리어barrier는 특정 이벤트가 발생할 때까지 여러 개의 스레드를 대기 상태로 잡아둘 수 있다는 측면에서 래치와 비슷하다고 볼 수 있다. 차이점은 모든 스레드가 배리어 위치에 동시에 이르러야 관문이 열리고 계속해서 실행할 수 있다는 점이다. 래치는 ‘이벤트’를 기다리기 위한 동기화 클래스이고, 배리어는 ‘다른 스레드’를 기다리기 위한 동기화 클래스이다. 

CyclicBarrier 클래스를 사용하면 여러 스레드가 특정한 배리어 포인트에서 반복적으로 서로 만나는 기능을 모델링 할 수 있고, 커다란 문제 하나를 여러 개의 작은 문제로 분리해 반복적으로 병렬 처리하는 알고리즘을 구현하고자 할 때 적용하기 좋다. 스레드는 배리어 포인트에 다다르면 await 메소드를 호출, await 메소드는 모든 스레드가 배리어 포인트에 도달할 때까지 대기한다. 모든 스레드가 도달하면 배리어는 모든 스레드를 통과시키고 배리어는 초기 상태로 돌아가 다시 배리어를 준비한다. 타임아웃, 인터럽트가 생기면 await 대기 스레드는 BrokenBarrierException이 발생한다. 

배리어는 대부분 실제 작업은 모두 여러 스레드에서 병렬로 처리하고, 다음 단계로 넘어가기 전에 이번 단계에서 계산해야 할 내용을 모두 취합해야 하는 등의 작업이 많이 일어나는 시뮬레이션 알고리즘에서 유용하게 사용할 수 있다. 


<br/>

예제 5.15는 배리어를 사용해 셀룰러 오토마타를 시뮬레이션하는 모습을 보여준다. 

```java
// 예제 5.15 CyclicBarrier를 사용해 셀룰러 오토마타의 연산을 제어
public class CelluarAutomata {
	private final Board mainBoard;
	private final CyclicBarrier barrier;
	private final Worker[] workers;

	public CellualarAutomata(Board board) {
		this.mainBoard = board;
		int count = Runtime.getRuntime().availableProcessors();
		this.barrier = new CyclicBarrier(count, new Runnable() {
			public void run() {
				mainBoard.commitNewValues();
			}});
		this.workers = new Worker[count];
		for(int i = 0; i < count; i++)
			workers[i] = new Worker(mainBoard.getSubBoard(count, i));
	}

	private class Worker implements Runnable {
		private final Board board;

		public Worker(Board board) { this.board = board; }
		public void run() {
			while (!board.hasCoverged()) {
				for(int x = 0; x < board.getMaxX(); x++)
					for(int y = 0; y < board.getMaxY(); y++)
						board.setNewValue(x, y, computeValue(x, y));
				try {
					barrier.await();
				} catch (InterruptedException ex) {
					return;
				} catch (BrokenBarrierException ex) {
					return;
				}
			}
		}
	}
	
	public void start() {
		for(int i = 0; i < workers.length; i++)
			new Thread(workers[i]).start();
		mainBoard.waitForConvergence();
	}
}
```



<br/>
<br/>
<br/>

## 5.6 효율적이고 확장성 있는 결과 캐시 구현

대부분의 서버 애플리케이션은 어떤 형태이건 캐시를 사용하므로 이전에 처리했던 작업의 결과를 재사용할 수 있다면, 메모리를 조금 더 사용하긴하지만 대기 시간을 줄이면서 처리 용량을 늘릴 수 있다. 캐시를 대충 만들게 되면 단일 스레드로 처리할 때 성능이 높아질 수는 있겠지만, 나중에는 성능의 병목 현상을 확장성의 병목으로 바꾸는 결과를 얻을 수 있다. 이 절에서는 연산이 오래 걸리는 작업에 적용할 수 있는 효율적이면서 쉽게 확장할 수 있는 결과 캐시를 구현해본다. 


<br/>

먼저 가장 쉽게 생각할 수 있는 방법(단순 HashMap 구현)으로 만들어보자. 예제 5.16의 인터페이스는 A라는 입력값과 V라는 결과값에 대한 메소드를 정의하고 있다. ExpensiveFunction 클래스는 결과를 뽑아 내는데 상당한 시간이 걸리는데 Computable에 한 겹을 덧씌워 이전 결과를 기억하는 캐시 기능을 추가해보자. 이런 방법을 흔히 메모이제이션memoization이라고 한다. 

```java
// 예제 5.16 HashMap과 동기화 기능을 사용해 구현한 첫 번째 캐시 [Not so good] 
public interface Computable<A, V> {
	V compute(A arg) throws InterruptedException;
}

public class ExpensiveFunction implements Computable<String, BigInteger> {
	public BigInteger compute(String arg) {
		// 오랜 작업 진행..
		return new BigInteger(arg);	
	}
}

public class Memoizer1<A, V> implements Computable<A, V> }
	@GuardedBy("this")
	private final Map<A, V> cache = new HashMap<A, V>();
	private final Computable<A, V> c;

	public Memoizer1(Computable<A, V> c) {
		this.c = c;
	}

	public synchronized V compute(A arg) throws InterruptedException {
		V result = cache.get(arg);
		if(result == null) {
			result = c.compute(arg);
			cache.put(arg, result);
		}
		return result;
	}
}
```

![Untitled](5%E1%84%8C%E1%85%A1%E1%86%BC%20%E1%84%80%E1%85%AE%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%83%E1%85%A1%E1%86%AB%E1%84%8B%E1%85%B1%20319ef1801e774252bc6fbb361ac3d619/Untitled.png)

// 그림 5.2 Memoizer1은 병렬성이 좋지 않다.



<br/>

예제 5.16에서는 캐시의 첫번째 버전이며 결과 저장소로는 HashMap을 사용했다. compute 메소드는 원하는 결과가 저장소에 있는지 확인하고, 있다면 그 값을 리턴해주고 없다면 결과를 새로 계산하고, 값을 리턴하기 전에 결과 값을 저장소에 넣어둔다. 

HashMap은 스레드안전하지 않기 때문에 compute 메소드 전체를 동기화 시켰고 이는 확장성의 측면에서 문제가 생긴다. 그림 5.2에서 여러 스레드가 compute 메소드를 실행하려는 상황을 잘 표현하고 있다. 



<br/>

예제 5.17에서는 HashMap대신 ConcurrentHashMap을 사용하는데 병렬 프로그래밍 입장에서 앞선 예제에서의 전체 동기화 성능 문제점을 해소시켜주었다. 하지만 캐시라는 기능에서는 아직 미흡한데 두 개 이상의 스레드가 동시에 같은 값을 넘기면서 compute 메소드를 호출해 같은 결과를 받아갈 가능성이 있기 때문이다. 

```java
// 예제 5.17 HashMap 대신 concurrentHashMap을 적용 [Not so good]
public class Memoizer2<A, V> implements Computable<A, V> {
	private final Map<A, V> cache = new ConcurrentHashMap<A, V>();
	private final Computable<A, V> c;

	public Memoizer2(Computable<A, V> c) {
		this.c = c;
	}

	public V compute(A arg) throws InterruptedException {
		V result = cache.get(arg);
		if(result == null) {
			result = c.compute(arg);
			cache.put(arg, result);
		}
		return result;
	}
}
```

위의 예제의 문제는 특정 스레드가 compute 메소드에서 연산을 시작했을 때, 다른 스레드는 현재 어떤 연산이 이뤄지고 있는지 알 수 없기 때문에 아래의 그림 5.3과 같이 동일한 연산을 시작할 수 있다는 점이다. 여기서 필요한 점은 특정 스레드가 A라는 연산의 값을 알고 싶을 때 A연산을 하고 있는지를 알고 싶다는것이고, A를 다른 스레드가 계산하고 있을 때 A값을 얻을 수 있는 가장 효과적 방법은 작업이 끝나기를 대기했다가 A의 결과 값이 무엇인지 해당 작업 스레드에게 물어보는것이다. 

![Untitled](5%E1%84%8C%E1%85%A1%E1%86%BC%20%E1%84%80%E1%85%AE%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%83%E1%85%A1%E1%86%AB%E1%84%8B%E1%85%B1%20319ef1801e774252bc6fbb361ac3d619/Untitled%201.png)

// 그림 5.3 Memoizer2에서 두 개의 스레드가 같은 값을 계산하고자 하는 경우



<br/>

이런 일은 FutureTask 클래스에서 처리하면 된다. 이미 끝났거나 끝날 예정인 연산 작업을 표현하며 get메소드는 연산 작업이 끝나는 즉시 연산 결과를 리턴해주며 연산 도중이면 작업이 끝날때까지 기다렸다가 그 결과를 알려준다. 예제 5.18에서 이제는 ConcurrentHashMap<A, Future<V>>라고 정의해서 사용해보자. 

```java
// 예제 5.18 FutureTask를 사용한 결과 캐시 [Still not so good]
public class Memoizer3<A, V> implements Computable<A, V> {
	private final Map<A, V> cache = new ConcurrentHashMap<A, Future<V>>();
	private final Computable<A, V> c;

	public Memoizer3(Computable<A, V> c) {
		this.c = c;
	}

	public V compute(A arg) throws InterruptedException {
		Future<V> f = cache.get(arg); // 원하는 값에 대한 연산 작업이 시작됐는지 확인
		if(f == null) {
			Callable<V> eval = new Callable<V>() {
				public V call() throws InterruptedException {
					return c.compute(arg);
				}
			};
			FutureTask<V> ft = new FutureTask<V>(eval); // 없다면 FutureTask를 하나 만들어 Map에 등록
			f = ft;
			cache.put(arg, ft);
			ft.run(); //  c.compute는 이 안에서 호출, 연산작업 시작
		}

		try {
			return f.get();
		} catch (ExecutionException e) {
			throw launderThrowable(e.getCause());
		}
	}
}
```

Memoizer3 클래스는 캐시라는 측면에서 이제 거의 완벽한 모습을 갖췄다. 동시 사용성도 갖고 있고 결과를 이미 알고 있다면 계산 과정 필요없이 즉시 가져갈 수 있고, 진행중인 연산 결과를 기다릴수 있도록 할 수 있다. 하지만 미흡한 점이 있는데, 여전히 여러 스레드가 같은 값에 대한 연산을 시작할 수 있다는 것이다. 이런 타이밍을 그림으로 표현하면 다음과 같다

![Untitled](5%E1%84%8C%E1%85%A1%E1%86%BC%20%E1%84%80%E1%85%AE%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%83%E1%85%A1%E1%86%AB%E1%84%8B%E1%85%B1%20319ef1801e774252bc6fbb361ac3d619/Untitled%202.png)

// 그림 5.4 Memoizer3에서 타이밍이 좋지 않아 같은 값을 두 번 계산하는 경우



<br/>

Memoizer3가 갖고 있는 허점은 Map에 결과를 추가할 때 단일 연산이 아닌 복합 연산을 사용하기 때문이며, 락을 사용해서는 단일 연산으로 구성할 수가 없다. 예제 5.19에서는 ConcurrentMap클래스의 putIfAbsent 라는 단일 연산 메소드를 사용해 결과를 저장한다.

```java
// 예제 5.19 Memoizer 최종 버전
public class Memoizer<A, V> implements Computable<A, V> {
	private final ConcurrentMap<A, Future<V>> cache = new ConcurrentHashMap<A, Future<V>>();
	private final Computable<A, V> c;

	public Memoizer(Computable<A, V> c) {
		this.c = c;
	}

	public V compute(A arg) throws InterruptedException {
		while(true) {
			Future<V> f = cache.get(arg);
			if(f == null) {
				Callable<V> eval = new Callable<V>() {
					public V call() throws InterruptedException {
						return c.compute(arg);
					}
				};
				FutureTask<V> ft = new FutureTask<V>(eval);
				f = cache.putIfAbsent(arg, ft);
				if(f == null) {
					f = ft;
					ft.run();
				}
			}

			try {
				return f.get();
			} catch (CancellationException e) {
				cache.remove(arg, f);
			} catch (ExecutionException e) {
				throw launderThrowable(e.getCause());
			}
		}
	}
}
```

실제 결과 값 대신 Future 객체를 캐시하는 방법은 이른바 캐시 공해를 유발할 수 있다. 예를 들어 특정 시점 시도 연산이 취소, 오류 발생하면 Future 객체 역시 취소되거나 오류가 발생했던 상황을 알려줄 것이다. 이런 문제의 해결을 위해 연산 취소된 경우 캐시에서 해당하는 Future 객체를 제거한다. 



<br/>

캐시된 내용이 만료되는 기능을 갖고 있지 않은데, 이 부분은 FutureTask 클래스를 상속받아 만료된 결과인지 여부를 알 수 있는 새로운 클래스를 만들어 사용하고, 결과 캐시를 주기적으로 돌아다니면서 만료된 결과 항목이 있는지 조사해 제거하는 기능을 구현하는 것으로 해결 할 수 있다.



<br/>

이번에 구현한 병렬 캐시가 마무리되고 나면 앞서 약속했던 대로 2장에서 살펴봤던 인수분해 서블릿에 실제 캐시 기능을 연결할 수 있다. 예제 5.20의 클래스는 Memoizer를 사용해 이전에 계산했던 값을 효율적이면서 확장성있게 관리한다.

```java
// 예제 5.20 Memoizer를 사용해 결과를 캐시하는 인수분해 서블릿
@ThreadSafe
public class Factorizer implements Sevlet {
	private final Computable<BigInteger, BigInteger[]> c = new Computable<BigInteger, BigInteger[]>() {
		public BigInteger[] compute(BigInteger arg) { return factor(arg); }
	};
	private final Computable<BigInteger, BigInteger[]> cache = new Memoizer<BigInteger, BigInteger[]>(c);

	public void service(ServletRequest req, ServletResponse resp) {
		try {
			BigInteger i = extractFromRequest(req);
			encodeIntoResponse(resp, cache.compute(i));
		} catch (InterruptedException e) {
			encodeError(resp, "factorization interrupted");
		}
	}
}
```