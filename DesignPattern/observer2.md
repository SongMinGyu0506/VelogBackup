## 🪢 Observer 패턴 - Java.util 사용

## ⛱ 서론
기존의 옵저버 패턴 포스팅에서는 직접 옵저버, 주제에 대한 인터페이스를 만들어 느슨한 연결로 관리하도록 만들었다.  

Java util 에서 해당 작업을 라이브러리로 만들어놓은 것이 있고, 이번 포스팅에서는 해당 라이브러리를 이용하여
Observer 패턴을 실습해본다.

## 🖥 구현
### 주제(Subject) 부분
```java
//Java.util Observer pattern
public class WeatherData extends Observable {
    private float temperature;
    private float humidity;
    private float pressure;

    public WeatherData() {
    }

    public void measurementsChange() {
        setChanged();
        notifyObservers();
    }

    public void setMeasurements(float temperature, float humidity, float pressure) {
        this.temperature = temperature;
        this.humidity = humidity;
        this.pressure = pressure;

        measurementsChange();
    }
    // Getter/Setter ...
}
```
```java
//기존의 옵저버 패턴
public class WeatherData implements Subject {
    private ArrayList observers;
    private float temperature;
    private float humidity;
    private float pressure;

    public WeatherData() {
        observers = new ArrayList();
    }

    @Override
    public void registerObserver(Observer o) {
        observers.add(o);
    }

    @Override
    public void removeObserver(Observer o) {
        int i = observers.indexOf(o);
        if (i >= 0) {
            observers.remove(i);
        }
    }

    @Override
    public void notifyObservers() {
        for (int i=0; i < observers.size(); i++) {
            Observer observer = (Observer) observers.get(i);
            observer.update(temperature,humidity,pressure);
        }
    }

    public void measurementsChanged() {
        notifyObservers();
    }

    public void setMeasurements(float temperature, float humidity, float pressure) {
        this.temperature = temperature;
        this.humidity = humidity;
        this.pressure = pressure;
        measurementsChanged();
    }
}
```
확실히 보이는 것은, 기존 라이브러리를 활용한 옵저버 패턴의 라인이 상당히 줄어들었음을 볼 수 있다.  
그리고 오버라이드 된 부분 즉, 기존의 Observer Interface의 부분은 전부 자바 라이브러리가 담당하고 있다.  

### 작동 방식
#### 객체가 옵저버가 되는 방법
기존의 옵저버 패턴에서는 직접 registerObserver 메소드를 호출하여 ArrayList에 등록하는 방식이었지만,  
자바 라이브러리를 활용한 방식에서는 옵저버 인터페이스의 addObserver 메소드를 호출하여 등록한다.

#### Observable에서 연락을 돌리는 방법
__Observable 이란?__  
단어 자체를 해석하면 관찰 할 수 있는, 주목할만한 이라는 뜻의 단어인데 주제가 될 객체에 사용한다.  
Observable 클래스에서 setChanged 메소드를 호출하여 객체의 상태가 바뀌었다는 것을 먼저 알린다.  
그리고 notifyObservers(), notifyObservers(Object arg)를 이용하여 전달한다.

#### Observer가 연락을 받는 방법
```java
public class CurrentConditionsDisplay implements Observer, DisplayElement {
    Observable observable;
    private float temperature;
    private float humidity;

    public CurrentConditionsDisplay(Observable observable) {
        this.observable = observable;
        observable.addObserver(this);
    }

    @Override
    public void display() {
        System.out.println("Current conditions: "+temperature+"F degrees and "+humidity+ "% humidity");
    }

    @Override
    public void update(Observable obs, Object arg) {
        if(obs instanceof WeatherData) {
            WeatherData weatherData = (WeatherData) obs;
            this.temperature = weatherData.getTemperature();
            this.humidity = weatherData.getHumidity();
            display();
        }
    }
}
```
기존의 방식과 같이 update 메소드를 이용한다.  
하지만 메소드의 파라미터가 조금 다른 것을 볼 수 있는데, 
첫 번째 파라미터가 어떤 주제, 두번째 파라미터가 notifyObserver에서 전달된 데이터이다.

즉, 정리하자면 주제 객체에서 setChange() -> notifyObserver() -> update() -> display() 순서로 진행된다.  
여기서 setChange() 메소드는 옵저버들의 상태를 true로 바꿔주는 역할을 한다. __왜 이런 메소드가 필요할까?__  
만약 해당 플래그 없이 옵저버 패턴이 가용된다면, 수시로 변화하는 값을 수시로 전달하게된다. 그렇다면 소요가 매우 크게 늘어날 수 있다.
이러한 상황을 막기 위해서 특정 조건을 만드는 것이다.

## 단점?
### 1. 상속
Observable은 __클래스__ 다. 즉 Observable 사용하기 위해서는 상속을 사용해야하는데, '상속' 이라는 행동 자체가 객체지향에서는 단점으로 작용할 수 있다.  
그리고 Observable을 사용하기 위해서는 사용하려는 클래스가 아무것도 상속받지 않아야 한다. 이러한 이유로 사용에 제약을 받게 된다.
### 2. 메소드 보호
데이터를 전달하는데 관여하는 메소드가 protected 로 되어있다. 즉, 상속받지 못하면 사용하지 못한다. 바로 위에서 상속의 제약이 있다고 했으므로 단점 중 하나이다.

## ✅ 마무리
옵저버 패턴은 겉으로 볼 때는 각 주제 객체가 생산한 데이터를 각 옵저버 객체에 전달하여 자동으로 옵저버 객체만 이용하여 파악 할 수 있게하는 패턴으로 보였다.  
옵저버 패턴의 내부에서 각 데이터를 전달하기 위해서는 주제 객체, 옵저버 객체의 각 메소드를 서로 이용하여 데이터를 전달하고, 옵저버를 관리하는 것 처럼 보인다.

