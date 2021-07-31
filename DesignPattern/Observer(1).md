## 옵저버(observer) 패턴이란?
* 객체의 상태 변화를 관찰하는 관찰자들 변화가 있을 때 마다 메소드를 통해 객체가 직접 목록을 통보하는 패턴

## 구조
![](https://upload.wikimedia.org/wikipedia/commons/thumb/8/8d/Observer.svg/1708px-Observer.svg.png)
* 각 상위에서는 Observer, Subject의 인터페이스로 구성되어있다.
* Observer는 각 옵저버의 업데이트 방식을 기술하고 있고
* Subject는 데이터를 가지는 객체가 관찰자를 등록하고 제거, 객체 내부의 값이 변화할때 통보하는 메소드들을 가지고 있다.
* notifyObservers() 메소드를 보면 각 옵저버들에게 알리는 방식을 사용하고있다.

## 예제
```java
package ObserverPattern.Interface;

public interface Observer {
    public void update(float temp, float humidity, float pressure);
}
```
```java
package ObserverPattern.Interface;

public interface Subject {
    public void registerObserver(Observer o);
    public void removeObserver(Observer o);
    public void notifyObservers();
}
```
* 해당 두 인터페이스들은 각 옵저버의 상태를 갱신하고, 옵저버를 등록하고 제거하는 행동을 기술한다.

```java
package ObserverPattern;

import ObserverPattern.Interface.Observer;
import ObserverPattern.Interface.Subject;

import java.util.ArrayList;

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
* 옵저버를 관리하고 객체를 생성하고 관리하는 객체이다. 옵저버는 리스트로 추가하고 관리한다.
* 그리고 notifyObservers 메소드를 이용하여, 옵저버가 저장되어있는 리스트들을 순환하면서,
각 옵저버들에 내장되어있는 update 메소드를 이용하여 옵저버로 관찰 할 수 있도록 만든다.
  
```java
package ObserverPattern.Display;

import ObserverPattern.Interface.DisplayElement;
import ObserverPattern.Interface.Observer;
import ObserverPattern.Interface.Subject;

public class CurrentConditionsDisplay implements Observer, DisplayElement {
    private float temperature;
    private float humidity;
    private Subject weatherData;

    public CurrentConditionsDisplay(Subject weatherData) {
        this.weatherData = weatherData;
        weatherData.registerObserver(this);
    }

    @Override
    public void display() {
        System.out.println("Current conditions: " + temperature + "F Degrees and " + humidity + "% humidity");
    }

    @Override
    public void update(float temp, float humidity, float pressure) {
        this.temperature = temp;
        this.humidity = humidity;
        display();
    }
}
```

```java
package ObserverPattern;

import ObserverPattern.Display.CurrentConditionsDisplay;

public class WeatherStation {
    public static void main(String[] args) {
        WeatherData weatherData = new WeatherData();

        CurrentConditionsDisplay currentConditionsDisplay = new CurrentConditionsDisplay(weatherData);

        weatherData.setMeasurements(80,65,30.4f);
        weatherData.setMeasurements(82,70,29.2f);
    }
}
```
* 이런식으로 값을 받는 객체인 weatherData 객체로 값을 받으면 setMeasurements 메소드가
옵저버로 갱신된 값을 전송? 해주는 기능또한 수행한다.