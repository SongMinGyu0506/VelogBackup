## 📗 데코레이터 패턴이란?
### 음식점을 예로들면...
어떤 음식점에서는 기본 메뉴에 여러가지 사이드 토핑들을 추가금을 받고 제공하는 경우가 있다.
만약 하나의 메뉴에 여러가지 토핑들을 추가해서 그 요금을 산출하는 것을 구현해야 할 때, 각 경우의 수를 전부 클래스로 만들어서
제공할 수 없다.  
그렇다면 상위 클래스에 has토핑, set토핑.... 만들어서 사용하면 될까? 당장의 문제해결은 가능하겠지만,
유연한 코드 작성이 불가능해지고, 메뉴가 바뀔 때 마다 기존 코드를 항상 수정해야한다. 즉, 객체지향 원칙인 OCP(Open-Closed)를 지키기 어렵다.  
 그래서, 해당 예를 해결하기 위한 방법 중 하나인 하나의 객체를 수정하지않고 감싸서 꾸미는 디자인 패턴 데코레이터 패턴을 사용한다.
### 그래서 데코레이터 패턴의 정의는?
__데코레이터 패턴__ 에서는 객체에 추가적인 요건을 동적으로 첨가할 수 있게한다.
데코레이터는 서브클래스를 만드는 것을 통해서 기능을 유연하게 확장할 수 있는 방법을 제공한다.(HeadFirst DesignPattern)  
![](http://lh5.ggpht.com/-LUmfWf0wOxo/UcAs6vqe7nI/AAAAAAAAFoI/w7gkcmgC4Ds/s1600/decorator%25255B8%25255D.png)

## 🖥 구현
```java
package Decorater;

public abstract class Beverage {
    String description = "제목 없음";

    public String getDescription() {
        return description;
    }

    public abstract double cost();
}
```
```java
package Decorater;

public abstract class CondimentDecorator extends Beverage{
    public abstract String getDescription();
}

```
```java
package Decorater;

public class Espresso extends Beverage{
    public Espresso() {
        description = "에스프레소";
    }

    @Override
    public double cost() {
        return 1.99;
    }
}

```
```java
package Decorater;

public class Mocha extends CondimentDecorator{
    Beverage beverage;

    public Mocha(Beverage beverage) {
        this.beverage = beverage;
    }

    @Override
    public double cost() {
        return beverage.cost() + .20;
    }

    @Override
    public String getDescription() {
        return beverage.getDescription() + ", 모카 추가";
    }
}
```
```java
package Decorater;

public class StarbuzzCoffee {
    public static void main(String[] args) {
        Beverage beverage = new Espresso();
        System.out.println(beverage.getDescription() + " $"+ beverage.cost());
        
        Beverage beverage1 = new Espresso();
        beverage1 = new Mocha(beverage1);
        beverage1 = new Mocha(beverage1);
        beverage1 = new Mocha(beverage1);
        System.out.println(beverage1.getDescription() + " $"+ beverage1.cost());
    }
}

```
## 📑 코드 분석
토핑, 음료 한가지씩 구현한 코드이다. 해당 코드에서는 상속을 이용했지만,
Interface를 사용해도 무방하다. 위의 코드를 읽고, main을 실행하는 시뮬레이터 코드를 살펴보자.
첫번째 beverage 객체는 아무런 토핑을 추가시키지 않았으므로 Espresso 가격 그대로 나올 것이다.  

하지만 beverage1 객체를 살펴보면, Espresso 객체를 생성하고, 해당 객체를 Mocha객체를 생성하는 매개변수로 사용하면서,
beverage1 객체에다가 Mocha 객체로 다시 덮어 씌우고있다.  

이것이 어떻게 가능한 일일까? 각 클래스의 관계를 살펴보면 알 수 있는데, Espresso 객체는 Beverage 객체를 상속받고 있고,
Mocha 객체는 Mocha -> CondimentDecorator -> Beverage 순서로 상속받고 있다.
결국 같은 Beverage라는 구성을 사용하기 때문에 기본 음료를 만들고, 그 기본 음료를 매개변수로 받아 토핑과 함께 합쳐진 객체를 다시 받도록한다.

즉, 해당 방법은 기존의 객체를 수정하지 않고, 새로운 객체를 만들어 추가할 수 있도록 한다.

## 📚 마무리
### 단점?
데코레이터 패턴은 분명 디자인, 즉 설계나 객체간 관계를 유연하게 해주는 것은 맞다. 하지만 해당 패턴을 과하게 사용한다면,
자잘한 클래스들이 매우 많이 늘어나, 다른사람들이 보기에 이해하기 어렵게 되는 경우도 있다. 즉, 패턴을 과하게 사용한다면, 코드가 필요 이상으로 복잡해질 수 있다는 것이다.

### 결론
데코레이터 패턴은 과하게 사용하면 오히려 독이되지만 기존의 객체 내부 수정없이 그 객체를 감싸는 객체를 만들어 해당 객체를 꾸며주는 기능 자체는 매우 편리하다고 생각한다.  
자바 코드 상에서 구현하기 위해서는 최상단의 상속 클래스 또는 인터페이스를 데이터 타입 형태로 만들고, 그 타입에 상속 받는 객체를
데코레이션 하는 객체로 계속 감싸서 만든다는 느낌을 가지면 될 것 같다.
