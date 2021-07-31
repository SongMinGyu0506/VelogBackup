## 전략(Strategy) 패턴이란?
__간단하게 말하면 느슨하게 연결해서 도중에 특정 알고리즘을 선택하는 행위 디자인 패턴__  
* 특정한 계열의 알고리즘들을 정의하고
* 각 알고리즘을 캡슐화하며
* 이 알고리즘들을 해당 계열 안에서 상호 교체가 가능하게 만든다.

## 구조
![](https://img1.daumcdn.net/thumb/R800x0/?scode=mtistory2&fname=https%3A%2F%2Ft1.daumcdn.net%2Fcfile%2Ftistory%2F1431503450475A8436)
* 자바 프로그래밍에서는 전략 부분을 인터페이스로 만들고 각 인터페이스를 상속받은 클래스들이 상세 구현, 메인 Context 클래스는
인터페이스 자료형을 선언하고 상세 구현을 객체로 가져온다.
  
## 예제
* 총이라는 이름의 객체를 만들었는데, 이 총 객체 내부에는 사격모드, 사용 탄약이 변수로 들어가있다.  
하지만 총마다 사용 탄약이 다르고, 발사 모드가 다르다. 그리고 지속적으로 새로운 총이 추가될 가능성이 있다.  
  __이 경우에 전략 패턴을 이용하여 만들어 보자__
  
```java
package StrategyPatternEx.Gun;

import StrategyPatternEx.ShotMode.ShotMode;
import StrategyPatternEx.UsingBullet.UsingBullet;

public abstract class Gun {
    ShotMode shotMode;
    UsingBullet usingBullet;
    public Gun() {}
    public abstract void display();

    public void usingShotMode() {
        shotMode.shotmode();
    }
    public void usingBulletMode() {
        usingBullet.usingBullet();
    }

    public void setShotMode(ShotMode sm) {
        shotMode = sm;
    }
    public void setUsingBullet(UsingBullet ub) {
        usingBullet = ub;
    }
}
```
* 총이라는 형태를 가지는 객체이다. 객체 내부에는 ShotMode,UsingBullet의 타입이 변수로 지정되어있다.
* 이 때, 변수들은 뒤에서 설명하겠지만 인터페이스로 되어있다.
* 사용 도중에 전략 변화를 위해서 set메소드를 만들었다.

```java
package StrategyPatternEx.ShotMode;

public interface ShotMode {
    public void shotmode();
}
```
````java
package StrategyPatternEx.UsingBullet;

public interface UsingBullet {
    public void usingBullet();
}
````
* 각 사용될 전략들이다. 해당 인터페이스 내부의 메소드들을 오버라이드하여 각 클래스가 상세하게 구현한다.

```java
package StrategyPatternEx.ShotMode;

public class FullAuto implements ShotMode{
    @Override
    public void shotmode() {
        System.out.println("방아쇠를 놓을 때 까지 사격");
    }
}
```
```java
package StrategyPatternEx.UsingBullet;

public class FiveFiveSixmil implements UsingBullet{
    @Override
    public void usingBullet() {
        System.out.println("5.56mm 탄환 사용");
    }
}
```
```java
package StrategyPatternEx.Gun;

import StrategyPatternEx.ShotMode.FullAuto;
import StrategyPatternEx.UsingBullet.FiveFiveSixmil;

public class GunAk47 extends Gun{
    public GunAk47() {
        shotMode = new FullAuto();
        usingBullet = new FiveFiveSixmil();
    }
    @Override
    public void display() {
        ...
    }
}
```
* 이런 방식으로 전략을 이용하여 만들면 OCP 원칙을 유지할 수 있다.
![](/Users/songmingyu/IdeaProjects/DesignPattern/Blogmdfile/dp1.png)


## 마무리
* 디자인 패턴을 처음으로 접한 패턴 스트래티지 패턴, 디자인 패턴을 공부하고 있는 __'헤드 퍼스트 디자인 패턴'__
에서 가장 앞에 있는 패턴인데, 여기서 중요한 것은 __인터페이스__ 를 활용하는 것이 디자인 패턴에서 중요하지 않을까 생각한다.
  