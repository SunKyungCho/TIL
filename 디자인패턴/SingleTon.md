# 싱글톤 패턴

인스턴스를 계속 생성하지 않고 한개의 인스턴스만 제공되어야 하는 경우 필요하다. 환경 세팅과 같은 경우 모두가 공통으로 사용되고 공유되는 환경에서는 하나의 인스턴스로만 공유되어 한다. 

```java
Settings settings = new Settins(); 
```
이렇게 생성하면 안된다. 

## 기본적인 사용방법

### 일반적인 사용

```java
public class Settings {

    private static Settings instance;

    private Settings() {}

    public static Settings getInstance() {
        if(instance == null) {
            instance = new Settings();
        }
        return instance;
    }
}
```

흔히 많이 사용하는 방법이다. 그러나 멀티쓰레드 환경에서도 보장이 될까? 동시에 getInstance() 가 호출될 경우 여러개 생성 될 수 있다. 

### synchronized 키워드 사용

```java
public static synchronized Settings getInstance() {
		if(instance == null) {
					instance = new Settings();
     }
    return instance;
}[
```

- 성능 부하가 생길 여지가 있다.

### 초기화 사용하기

```java
private static final Settings INSTANCE = new Settings();
```

- 미리 만들어 논다는게 단점이다.
- 가볍하면 문제가 되지 않는다. 생성하는 비용이 비싸고 또는 많이 사용되지 않는다면 불필요한 자원이 소모 된다.

### Double checked locking 사용

```java
public class Settings {

    private static volatile Settings instance;

    private Settings() {}

    public static Settings getInstance() {
        if(instance == null) {
            synchronized (Settings.class) {
                if(instance == null) {
                    instance = new Settings();
                }
            }
        }
        return instance;
    }
}
```

- volatile에 대한 이해 필요하다.
- Java 1.5 이상은 되어야 한다.

### static inner class 사용

```java
public class Settings {

    private Settings() {}

    private static class SettingsHolder {
        private static final Settings INSTANCE = new Settings();
    }
    
    public static Settings getInstance() {
        return SettingsHolder.INSTANCE;
    }
}
```

- 권장하는 방법이다.
- 멀티쓰레드 세이프하게 동작할 수 있다.
- getInstance()가 호출될때 로딩되므로 lazy 로딩도 확보 할 수 있다.

## 취약점(?)

- 리플렉션을 사용하여 인스턴스 생성하면 보장해 주지 않는다.
- 파일을 쓰고 읽어서 직렬화, 역직렬화를 사용한다면? 단일 인스턴스를 보장해 주지 않는다.
    - `protected Object readResolve()` 정의 하면 해결가능.

## Eum을 사용한 방법

컴파일 타임에 이미 생성되어 있다. 

eum 로딩하는 순간 이미 만들어졌다. 

- 완벽한 싱글톤을 제공하는 방법이다.
- 직렬화, 역직렬화에도 매우 안정적이다.
- enum은 리플렉션을 통한 인스턴스 생성을 막아고 있다. 만들 수 없다. 


### 단점

- 미리 만들어 진다.
- 마크인터페이스로 사용한다면 제네릭 사용 불가
- 상속 사용 불가

static: 단 한개의 인스턴스가 존재(동시성 문제를 해결해야 한다)
enum: 제한된 수의 인스턴스가 존재
class: 무제한의 인스턴스가 존재
