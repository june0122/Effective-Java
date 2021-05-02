### `아이템 28`

# 배열보다는 리스트를 사용하라

배열과 제네릭 타입에는 중요한 두 가지 차이가 있다.

### 첫 번째, 배열은 공변 <sup>covariant; 共變</sup>이다.

`Sub`가 `Super`의 하위 타입이라면 배열 `Sub[]`는 배열 `Super[]`의 하위 타입이 된다. <sup>공변, 즉 함께 변한다는 뜻</sup> 반면, **제네릭은 불공변 <sup>invariant; 不共變</sup>이다.** 즉, 서로 다른 타입 `Type1`과 `Type2`가 있을 때, `List<Type1>`은 `List<Type2>`의 하위 타입도 아니고 상위 타입도 아니다.

이것만 보면 제네릭에 문제가 있다고 생각할 수도 있지만, 사실 문제가 있는 건 배열 쪽이다.

> 런타임에 실패 (문법상 허용되는 코드)

```java
Object[] objectArray = new Long[1];
objectArray[0] = "타입이 달라 넣을 수 없다."; // ArayStoreException을 던진다.
```

> 컴파일타임에 실패 (문법에 맞지 않는 코드)

```java
List<Object> ol = new ArrayList<Long>(); // 호환되지 않는 타입이다.
ol.add("타입이 달라 넣을 수 없다.");
```

어느 쪽이든 Long용 저장소에 String을 넣을 수는 없다. 다만 배열에서는 그 실수를 런타임에야 알게 되지만, **리스트를 사용하면 컴파일할 때 실수를 바로 알 수 있다.**

### 두 번째, 배열은 실체화 <sup>reify</sup>된다.

배열은 런타임에도 자신이 담기로 한 원소의 타입을 인지하고 확인한다. 그래서 위의 '런타임에 실패' 코드에서 보듯 Long 배열에 String을 넣으려 하면 ArayStoreException이 발생한다. 반면, **제네릭은 타입 정보가 런타임에는 소거 <sup>erasure</sup>된다.** 원소 타입을 컴파일타임에만 검사하며 런타임에는 알 수조차 없다는 뜻이다. 소거는 제네릭이 지원되기 전의 레거시 코드와 제네릭 타입을 함께 사용할 수 있게 해주는 메커니즘으로, 자바 5가 제네릭으로 순조롭게 전환될 수 있도록 해줬다.(아이템 26)

이상의 주요 차이로 인해 배열과 제네릭은 잘 어우러지지 못한다. 예컨데 배열은 제네릭 타입, 매개변수화 타입, 타입 매개변수로 사용할 수 없다. 즉, 코드를 `new List<E>[]`, `new List<String>[]`, `new E[]` 식으로 작성하면 컴파일할 때 제네릭 배열 생성 오류를 일으킨다.

제네릭 배열을 만들지 못하게 막은 이유는 무엇일까? 타입 안전하지 않기 때문이다. 이를 허용한다면 컴파일러가 자동 생성한 형변환 코드에서 런타임에 ClassCastException이 발생할 수 있다. 런타임에 ClassCastException이 발생하는 일을 막아주겠다는 제네릭 타입 시스템의 취지에 어긋나는 것이다.

> 제네릭 배열 생성을 허용하지 않는 이유 - 컴파일되지 않는다.

```java
List<String>[] stringLists = new List<String>[1];  // (1)
List<Integer> intList = List.of(42);               // (2)
Object[] objects = stringLists;                    // (3)
objects[0] = intList;                              // (4)
String s = stringLists[0].get(0)                   // (5)
```

제네릭 배열을 생성하는 (1)이 허용된다고 가정해보자. (2)는 원소가 하나인 `List<Integer>`를 생성한다. (3)은 (1)에서 생성한 `List<String>`의 배열을 `Object` 배열에 할당한다. 배열은 공변이니 아무 문제 없다. (4)는 (2)에서 생성한 `List<Integer>`의 인스턴스를 `Object` 배열의 첫 원소로 저장한다. 제네릭은 소거 방식으로 구현되어서 이 역시 성공한다. 즉, 런타임에는 `List<Integer>` 인스턴스 타입은 단순히 `List`가 되고, `List<Integer>[]` 인스턴스의 타입은 `List[]`가 된다. 따라서 (4)에서도 ClassCastException을 일으키지 않는다.

이제부터가 문제다. `List<String>` 인스턴스만 담겠다고 선언한 `stringLists` 배열에는 지금 `List<Integer>` 인스턴스가 저장돼 있다. 그리고 (5)는 이 배열의 처음 리스트에서 첫 원소를 꺼내려한다. 컴파일러는 꺼낸 원소를 자동으로 String으로 형변환하는데, 이 원소는 Integer이므로 런타임에 ClassCastException이 발생한다. 이렇게 제네릭 배열이 생성되지 않도록 방지하려면 (1)에서 컴파일 오류를 내야 한다.

`E`, `List<E>`, `List<String>` 같은 타임을 실체화 불가 타입 <sup>non-reifiable type</sup>이라 한다. 쉽게 말해, 실체화되지 않아서 런타임에는 컴파일타임보다 타입 정보를 적게 가지는 타입이다. 소거 매커니즘 때문에 매개변수화 타입 가운데 실체화될 수 있는 타입은 `List<?>`와 `Map<?, ?>` 같은 와일드카드 타입뿐이다. 배열을 비한정적 와일드카드 타입으로 만들 수는 있지만, 유용하게 쓰일 일은 거의 없다.

배열을 제네릭으로 만들 수 없어 귀찮을 때도 있다. 예컨데 제네릭 컬렉션에서는 자신의 원소 타입을 담은 배열을 반환하는 게 보통은 불가능하다. <sup>아이템 33에서 완벽하지는 않지만 대부분의 상황에서 이 문제를 해결해주는 방법을 소개한다.</sup> 또한 제네릭 타입과 가변인수 메서드 <sup>varargs method, 아이템 53</sup>를 함께 쓰면 해석하기 어려운 경고 메시지를 받게 된다. 가변인수 메서드를 호출할 때마다 가변인수 매개변수를 담을 배열이 하나 만들어지는데, 이 때 그 배열의 원소가 실체화 불가 타입이라면 경고가 발생하는 것이다. 이 문제는 `@SafeVarargs` 애너테이션으로 대처할 수 있다. <sup>아이템 32</sup>

배열로 형변환할 때 제네릭 배열 생성 오류나 비검사 형변환 경고가 뜨는 경우 대부분은 배열인 `E[]` 대신 컬렉션인 `List<E>`를 사용하면 해결된다. 코드가 조금 복잡해지고 성능이 살짝 나빠질 수도 있지만, 그 대신 타입 안전성과 상호운용성은 좋아진다.

## 예시

생성자에서 컬렉션을 받는 Chooser 클래스를 예로 살펴보자. 이 클래스는 컬렉션 안의 원소 중 하나를 무작위로 선택해 반환하는 choose 메서드를 제공한다. 생성자에 어떤 컬렉션을 넘기느냐에 따라 이 클래스를 주사위판, 매직 8볼, 몬테카를로 시뮬레이션용 데이터 소스 등으로 사용할 수 있다.

> Chooser - 제네릭을 시급히 적용해야 한다!

```java
public class Chooser {
    private final Object[] choiceArray;

    public Chooser(Collection choices) {
        choiceArray = choises.toArray();
    }

    public Object choose() {
        Random rnd = ThreadLocalRandom.current();
        return choiceArray[rnd.nextInt(choiceArray.length)];
    }
}
```

이 클래스를 사용하려면 choose 메서드를 호출할 때마다 반환된 Object를 원하는 타입으로 형변환해야 한다. 혹시나 타입이 다른 원소가 들어 있었다면 런타임에 형변환 오류가 날 것이다. 뒤에 나올 아이템 29 <sup>이왕이면 제네릭 타입으로 만들라</sup>의 조언을 가슴에 새기고 이 클래스를 제네릭으로 만들어보자.

> Chooser를 제네릭으로 만들기 위한 첫 시도 - 컴파일되지 않는다.

```java
public class Chooser<T> {
    private final T[] choiceArray;

    public Chooser(Collection<T> choices) {
        choiceArray = choices.toArray();
    }

    // choose 메서드는 그대로
}
```

`choiceArray = (T[]) choices.toArray();` 와 같이 Object 배열을 T로 형변환하여도 T가 무슨 타입인지 알 수 없으니 컴파일러는 이 형변환이 런타임에도 안전한지 보장할 수 없다는 경고 메시지를 띄운다. 제네릭에서는 원소의 타입 정보가 소거되어 런타임에는 무슨 타입인지 알 수 없음을 명심하자! 컴파일러가 경고를 띄우지만 위 프로그램은 동작한다. 단지 컴파일러가 안전을 보장하지 못할 뿐이다. 코드를 작성하는 사람이 안전하다고 확신한다면 주석을 남기고 애너테이션을 달아 경고를 숨겨도 된다. 하지만 애초에 경고의 원인을 제거하는 편이 훨씬 낫다. <sup>아이템 27</sup>

비검사 형변환 경고를 제거하려면 배열 대신 리스트를 사용하면 된다. 다음 Chooser는 오류나 경고 없이 컴파일된다.

> 리스트 기반 Chooser - 타입 안정성 확보!

```java
public class Chooser<T> {
    private final T[] choiceArray;

    public Chooser(Collection<T> choices) {
        choiceArray = new ArrayList<>(choices);
    }

    public Object choose() {
        Random rnd = ThreadLocalRandom.current();
        return choiceList.get(rnd.nextInt(choiceList.size));
    }
}
```

위의 버전은 코드양이 조금 늘고 조금 더 느릴 테지만, 런타임에 ClassCastException을 만날 일은 없으니 그만한 가치가 있다.

## 핵심 정리

> 배열과 제네릭에는 매우 다른 타입 규칙이 적용된다. 배열은 공변이고 실체화되는 반면, 제네릭은 불공변이고 타입 정보가 소거된다. 그 결과 **배열은 런타임에는 타입 안전하지만 컴파일타임에는 그렇지 않다**. **제네릭은 반대**다. 그래서 둘을 섞어 쓰기란 쉽지 않다. 둘을 섞어 쓰다가 컴파일 오류나 경고를 만나면, 가장 먼저 배열을 리스트로 대체하는 방법을 적용해보자.