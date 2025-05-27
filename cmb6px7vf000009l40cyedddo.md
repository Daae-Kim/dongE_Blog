---
title: "Java) try catch finally VS try with resources"
seoTitle: "Try-Catch vs Try-With-Resources"
datePublished: Tue May 27 2025 16:15:53 GMT+0000 (Coordinated Universal Time)
cuid: cmb6px7vf000009l40cyedddo
slug: try-catch-finally-vs-try-with-resources
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1748364200990/c4a71537-d99a-4894-bf17-2fd1118be78f.png
tags: java, exceptionhandling, try-catch-finally, try-with-resources

---

예외를 처리할때, 사용한 외부자원은 반드시 반납해야한다. 예외 처리는 try catch finally 와 try with resources 두가지 구문으로 처리할 수 있는데 이 둘의 가장 큰 차이는 자원을 반납하는 방식에 있다.

### try catch finally

예외 처리 시 외부자원은 사용후 반드시 반납되어야 한다. 자바는 이것을 위해 어떤 경우라도 반드시 호출되는 finally 기능을 제공한다.

* try - catch - finally 구조는 정상 흐름, 예외 흐름, 마무리 흐름을 제공한다.
    
* 여기서 try 를 시작하기만 하면, finally 코드 블럭은 어떤 경우라도 호출된다.
    
* 심지어 try, catch 안에 처리 할 수 없는 예외가 발생해도 finally는 반드시 호출된다.
    
    * finally 코드 블럭이 끝난 이후에 예외가 던져진다.
        

```java
try {
      client.connect();
      client.send(data);
} catch (ConnectExceptionV3 | SendExceptionV3 e) {
	System.out.println("[연결 또는 전송 오류] 주소: , 메시지: " + e.getMessage());
} finally {
      client.disconnect(); // 외부 자원 반납
}
```

### try with resources

만약 try catch fianlly 에서 외부 자원을 반납하는 로직을 실수로 넣지 않았다면 어떻게 될까? try 구문에서 사용한 외부 자원을 finally 구문까지 유지해야할 필요가 있을까?

try with resources 는 자바 7부터 지원하는 구문으로 AutoCloseable 인터페이스의 close()를 사용해 별도의 로직 없이 외부자원을 반납하도록 한다.

```java
public interface AutoCloseable{
	void close() throws Exception;
}
```

```java
public class NetworkClientV5 implements AutoCloseable {
	...
	@Override
	public void close(){
		System.out.println("NetworkClientV5.close");
		disconnet();
		}
	}
```

* 먼저 AutoCloseable 인터페이스를 구현해야한다.
    
* try안에 사용할 자원을 명시한다.
    
    * 이 자원은 try 블럭이 끝나는 즉시 AutoCloseable.close()를 호출해서 자원을 해제한다.
        
* Implements 를 통해 AutoCloseable 을 구현한다.
    
* close () : AutoCloseable 인터페이스가 제공하는 close 메소드는 try가 끝나면 자동으로 호출된다. 종료 시점에 자원을 반납하는방법을 여기에 정의하면 된다.
    

```java
public class NetworkServiceV5 {
	public void sendMessage(String data) {
		String address = "https://example.com";
		
		try (NetworkClientV5 client = new NetworkClientV5(address)) {
			client.initError(data);
			client.connect();
			client.send(data);
		} catch (Exception e) {
				System.out.println("[예외 확인]: " + e.getMessage());
				throw e;
		}
	}
}
```

### try catch fianlly 와 비교한 try with resources 의 장점

* 리소스 누수 방지 : 모든 리소스가 제대로 닫히도록 보장한다.
    
    * fianlly 블럭안에서 자원해제 코드를 누락하는 문제를 예방할 수 있다.
        
* 코드 간결성 : close 호출이 없어 간결하고 읽기 쉬워진다.
    
* 자원 스코프 범위 한정 : 리소스로 사용되는 스코프가 try 블럭안으로 한정된다.
    
* 조금더 빠른 자원 해제 : try catch finally 에서는 catch 이후에 자원을 반납했다. try with resources 구문은 try 블럭이 끝나면 즉시 close()를 호출한다.