# 왜 ConcurrentMap은 Thread Safety 할까

책 관련 이벤트 페이지를 처리하는 메소드 쪽에서 ConcurrentMap 콜렉션을 발견하였다. 처음보는 콜렉션이라 당황했지만 여기에는 히스토리가 있었다. 책을 기반으로 신청하는 페이지에서 수량을 전부 소진했음에도 동시신청에 대한 준비가 안되어서 중복 신청 및 수량이 마이너스까지 처리되는 것이였다.

물론 수량을 체크하는 로직이 없었고 세션 체크 등을 하지 않아 당연히 이중 신청 및 여러 사람이 동시신청에 대한 문제를 해소하지 못한 것으로 판단되지만 어째든 ConcurrentMap을 Thread Safety 목적으로 사용한 것으로 판단되어 공부하고자 글을 추가한다



## HashMap, HashTable, HashConcurrentMap

Map 콜렉션에서 가장 많이 사용되는 3가지를 비교하면서 소스를 분석해보고자 한다.



```java
Map<String, Integer> hashMap = new HashMap<>();
		Map<String, Integer> hashTable = new Hashtable<>();
		Map<String, Integer> concurrentHashMap = new ConcurrentHashMap<>();

		new Thread(()->{
			for (int i = 0 ; i < 1000; i ++) {
				hashMap.put("map"+i, 1);
			}
		}).start();


		new Thread(()->{
			for (int i = 0 ; i < 1000; i ++) {
				hashMap.put("map"+i, 2);
			}
		}).start(); 

		new Thread(()->{
			for (int i = 0 ; i < 1000; i ++) {
				hashMap.put("map"+i, 3);
			}
		}).start();

		try {
			
			Thread.sleep(1000);
			System.out.println("----테스트 결과-------");
			System.out.println(hashMap.size());
			
			hashMap.entrySet().stream().sorted(Map.Entry.comparingByKey()).forEach(e ->{
				System.out.println(e.getKey() + ":" + e.getValue());
			});
		
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
```

각각 HashMap, HashTable, ConcurrentMap으로 교체 후 테스트를 진행해보았다.

참고로 size를 리턴할 경우 중복값이 들어갈 경우 size가 더 나오는데 이는 다음 사이트에 이유를 알 수 있다.

[HashMap resize 및 hashcode stackoverflow](https://stackoverflow.com/questions/47176918/why-hashmap-resize-based-on-total-size-instead-of-filled-buckets)



### 결과

Map 갯수 : 1035

HashTable 갯수 : 1000

ConcurrentHashMap : 1000



결과적으로 HashTable과 ConcurrentHashMap은 Thread Safety하다는 것을 알 수 있다.

둘이 차이가 비교해보면 실제 소스를 통해 알 수 있었는데 다음과 같다

- HashTable의 put 메소드

  ```java
  public synchronized V put(K key, V value) {
   	... 생략 ...  
  }
  ```

- ConcurrentMap의 put 메소드

  ```java
   public V put(K key, V value) {
   	return putVal(key, value, false);
   }
   
   final V putVal(K key, V value, boolean onlyIfAbsent) {
   	... 중략 ...
   	
   	 synchronized  (f) {
   		...생략 ...
   	}
   	
   }
  ```

  

HashTable의 경우 메소드에서 Synchronized 처리되어있으며 ConcurrentMap의 putVal 내부에서 Synchronized 처리되어있는데 이는 속도차이를 발생시킨다



- HashTable 시간 (500000개 )  = 110 m/s

- ConcurrentHashMap 시간 (500000개) = 78 m/s

