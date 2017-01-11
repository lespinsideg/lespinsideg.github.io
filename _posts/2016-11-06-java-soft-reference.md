Java Reference - SoftReference
===

자바에는 객체에 대하여 strong, soft, weak, phantom 4가지 레퍼런스가 존재하며 각 레퍼런스의 유무에 따라 객체의 reachability가 결정된다. 객체의 reachability는 가비지 컬렉터의 수집 기준이 되는데 여기에 대해서는 네이버 D2의 [Java Reference와 GC](http://d2.naver.com/helloworld/329631)에 잘 설명되어 있다. 그렇다면 각 레퍼런스 타입은 언제 사용될까? SoftReference부터 찾아보았다.

## SoftReference
### joda-time
[joda-time](https://github.com/JodaOrg/joda-time)은 Java의 날짜와 시간에 대한 클래스들을 대체하기 위해 만들어진 오픈소스 라이브러리이다. joda-time에서는 `DateTimeZone` 캐싱에 `SoftReference`를 사용한다.

```java
public DateTimeZone getZone(String id) {
    if (id == null) {
        return null;
    }

    Object obj = iZoneInfoMap.get(id);
    if (obj == null) {
        return null;
    }

    if (obj instanceof SoftReference<?>) {

        @SuppressWarnings(“unchecked”)
        SoftReference ref = (SoftReference) obj;
        DateTimeZone tz = ref.get();
        if (tz != null) {
            return tz;
        }
        // Reference cleared; load data again.
        return loadZoneData(id);
    } else if (id.equals(obj)) {
        // Load zone data for the first time.
        return loadZoneData(id);
    }

    // If this point is reached, mapping must link to another.
    return getZone((String)obj);
}
```

위의 코드는 joda-time의 [ZoneInfoProvider.java](https://github.com/JodaOrg/joda-time/blob/master/src/main/java/org/joda/time/tz/ZoneInfoProvider.java)의 일부분이다. `getZone`은 파라미터로 받은 id에 대한 `DateTimeZone`을 반환하는 메서드이다. `ZoneInfoProvider`의 프라이빗 필드인 `iZoneInfoMap`에는 id를 키 값으로 이미 로드한 `DateTimeZone`에 대한 `SoftReference`나 아직 로드하지 않은 id 문자열이 값으로 가지고 있다. 그리고 아직 로드하지 않은 `DateTimeZone`이나 이미 로드했지만 GC된 `DateTimeZone`에 대해서 `loadZoneData`를 호출하여 반환한다.

### spring framework
spring framework에서는 `ConcurrentReferenceHashMap` 이라는 자체 `ConcurrentHashMap`을 제공한다. `Collections.synchronizedMap(new WeakHashMap<K, Reference<V>>())`을 대체하기 만들어진 클래스로 내부적으로 기본 `SoftReference`와 `WeakReference`를 상속받은 `SoftEntryReference`와 `WeakEntryReference`를 사용하고 있다.

Spring framework 전반에서는 `CocurrentReferenceHashMap`을 캐시로 사용하고 있는데 주로 비용이 큰 reflection에 사용하고 있다.

> org.springframework.core.GenericTypeResolver
```java
/** Cache from Class to TypeVariable Map */
	@SuppressWarnings("rawtypes")
	private static final Map<Class<?>, Map<TypeVariable, Type>> typeVariableCache =
			new ConcurrentReferenceHashMap<>();
```

> org.springframework.core.ResolvableType
```java
private static final ConcurrentReferenceHashMap<ResolvableType, ResolvableType> cache =
			new ConcurrentReferenceHashMap<>(256);
```

> org.springframework.util.ReflectionUtils
```java
	/**
	 * Cache for {@link Class#getDeclaredMethods()} plus equivalent default methods
	 * from Java 8 based interfaces, allowing for fast iteration.
	 */
	private static final Map<Class<?>, Method[]> declaredMethodsCache =
			new ConcurrentReferenceHashMap<>(256);
	/**
	 * Cache for {@link Class#getDeclaredFields()}, allowing for fast iteration.
	 */
	private static final Map<Class<?>, Field[]> declaredFieldsCache =
			new ConcurrentReferenceHashMap<>(256);
```

### 캐시로 사용해도 될까?
[oracle javadoc](https://docs.oracle.com/javase/8/docs/api/java/lang/ref/SoftReference.html)에는 `SoftReference`가 주로 memory sensitive cache에 사용된다고 적혀있다.
> Soft reference objects, which are cleared at the discretion of the garbage collector in response to memory demand. Soft references are most often used to implement memory-sensitive caches.

[안드로이드 개발 참조문서](https://developer.android.com/reference/java/lang/ref/SoftReference.html)에서는 `SoftReference`를 캐싱에 사용하하는 것은 비효율적이라고 말한다.

> Avoid Soft References for Caching
In practice, soft references are inefficient for caching. The runtime doesn't have enough information on which references to clear and which to keep. Most fatally, it doesn't know what to do when given the choice between clearing a soft reference and growing the heap.
The lack of information on the value to your application of each reference limits the usefulness of soft references. References that are cleared too early cause unnecessary work; those that are cleared too late waste memory.

대신 [android.util.LruCache](https://developer.android.com/reference/android/util/LruCache.html)를 사용할 것을 권장하는데 `LruCache`는 생성 시 지정한 갯수만큼 strong reference를 유지하는 방식으로 캐시를 구현하였다.

 [안드로이드 개발자](http://developer.android.com/training/displaying-bitmaps/cache-bitmap.html#memory-cache) 페이지에는 Bitmap caching과 관련하여 다음과 같은 내용도 있다.
> Note: In the past, a popular memory cache implementation was a SoftReference or WeakReference bitmap cache, however this is not recommended. Starting from Android 2.3 (API Level 9) the garbage collector is more aggressive with collecting soft/weak references which makes them fairly ineffective. In addition, prior to Android 3.0 (API Level 11), the backing data of a bitmap was stored in native memory which is not released in a predictable manner, potentially causing an application to briefly exceed its memory limits and crash.

모든 경우에 `SoftReference`를 캐시로 사용하는 것이 부적절하다기 보다는 안드로이드의 적극적인 GC 특성 상 `SoftReference`가 효과적이지 못하다는 이야기인 것 같다.
