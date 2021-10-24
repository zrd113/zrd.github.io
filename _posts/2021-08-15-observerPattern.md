---
layout: post
title: 观察者模式
date: 2021-08-15
Author: zrd
tags: [Java]
toc: true
---

## 介绍

一个软件系统常常要求在某一个对象的状态发生变化的时候，某些其他的对象做出相应的改变。做到这一点的设计方案有很多，但是为了使系统能够易于复用，应该选择低耦合度的设计方案。减少对象之间的耦合有利于系统的复用，但是同时设计师需要使这些低耦合度的对象之间能够维持行动的协调一致，保证高度的协作。观察者模式是满足这一要求的各种设计方案中最重要的一种。它定义了对象之间的一对多的依赖关系——当一个对象改变状态时，它的所有依赖者都会受到通知并自动更新状态。

观察者模式涉及的角色有：

1. 主题角色：抽象主题角色把所有对观察者对象的引用保存在一个集合里，每个主题都可以有任意数量的观察者。抽象主题提供一个接口，可以增加和删除观察者对象，抽象主题角色又叫做抽象被观察者(Observable)角色。

2. 观察者角色： 为所有的具体观察者定义一个接口实现 update 行为，在得到主题的通知时，update 被调用，从而能够更新自己。

## 案例：气象观测系统

案例包含两个系统：一个 WeatherData 系统，负责计算、追踪目前的天气状况（温度，湿度，气压），三种电子显示器，分别显示给用户目前的天气状况。

### 版本1

```
public class WeatherData {
    public float getTemperature() {
        ...
    }

    public float getHumidity() {
        ...
    }

    public float getPressure() {
        ...
    }

    public void measurementsChanged() {
        float temp = getTemperature();
        float humidity = getHumidity();
        float pressure = getPressure();

        // 三种显示器的实现类的对象：
        // currentConditionsDisplay 当前天气状态实时显示
        // statisticsDisplay 天气数据统计信息展示
        // forecastDisplay 天气预报展示
        currentConditionsDisplay.update(temp, humidity, pressure);
        statisticsDisplay.update(temp, humidity, pressure);
        forecastDisplay.update(temp, humidity, pressure);
    }
}
```

问题：
1. 扩展性很差，如果产品需要增加新的显示器类型，那么每次增加删除，都要打开 measurementsChanged 方法修改代码。
2. 无法运行时动态增加删除显示器。

### 版本2：基于push的观察者模式

```
// 主题接口
public interface Subject {
    void registerObserver(Observer o);
    void removeObserver(Observer o);
    void notifyObservers();
}
 
// 观察者接口
public interface Observer {
    void update(float temp, float humidity, float pressure);
}

// 显示器接口
public interface DisplayElement {
    void display();
}
 
public class WeatherData implements Subject {
    private List<Observer> observers; 
    private float temperature;
    private float humidity;
    private float pressure;

    public WeatherData() {
        observers = new ArrayList<>();
    }

    public float getTemperature() {
        return temperature;
    }

    public float getHumidity() {
        return humidity;
    }

    public float getPressure() {
        return pressure;
    }

    // 注册观察者
    public void registerObserver(Observer o) {
        observers.add(o); 
    }

    // 删除观察者
    public void removeObserver(Observer o) {
        observers.remove(o);
    }

    // 通知观察者
    public void notifyObservers() {
        for (Observer obs : observers) {
            obs.update(temperature, humidity, pressure);
        }
    }

    private void measurementsChanged() {
        notifyObservers();
    }

    // 被动接收气象站的数据更新，或者主动抓取，这里可以实现爬虫等功能
    public void setMeasurements(float temperature, float humidity, float pressure) {
        this.temperature = temperature;
        this.humidity = humidity;
        this.pressure = pressure;
        // 一旦数据更新了，就立即同步观察者们，这是push模型的观察者设计模式的实现，对应的还有一种基于pull模型的实现
        measurementsChanged();
    }
}

// 各个显示器类也是具体的观察者们
public class CurrentConditionsDisplay implements DisplayElement, Observer {
    private float temperature;
    private float humidity;
    private Subject weatherData; 

    public CurrentConditionsDisplay(Subject weatherData) {
        this.weatherData = weatherData;
        // 将自己注册到主题中
        weatherData.registerObserver(this);
    }

    @Override
    public void display() {
        System.out.println("Current conditions: " + temperature
                + "degrees and " + humidity + "is % humidity");
    }

    @Override
    public void update(float temp, float humidity, float pressure) {
        this.temperature = temp;
        this.humidity = humidity;
        display();
    }
}
 
// 其余显示器类似
public class ForecastDisplay implements DisplayElement, Observer {
    ...
}

// 其余显示器类似
public class StatisticsDisplay implements Observer, DisplayElement {
    ...
}
 
public class WeatherStation {
    public static void main(String[] args) {
        WeatherData weatherData = new WeatherData(); // 主题
        // 各个观察者注册到主题
        Observer currentDisplay = new CurrentConditionsDisplay(weatherData);
        Observer statisticsDisplay = new StatisticsDisplay(weatherData);
        Observer forecastDisplay = new ForecastDisplay(weatherData);
        // 本设备会给气象观测系统推送变化的数据
        weatherData.setMeasurements(80, 65, 30.4f);
        weatherData.setMeasurements(82, 70, 29.2f);
        weatherData.setMeasurements(78, 90, 29.2f);
    }
}
```

push模型的问题：
1. 主题有时候并不能完全掌握每个观察者的需求，让观察者主动pull数据会更合适一些。
2. 如果仅仅是某些个别的观察者需要一点儿数据，那么主题仍然会通知全部的观察者，导致和该业务无关的观察者都要被通知，这是没有必要的。
3. 以后观察者需要扩展一些状态，如果采用推模型，那么主题除了要新增必要的状态属性外，还要修改通知的代码逻辑。如果基于拉模型，主题只需要提供一些对外的getter方法，让观察者主动拉取数据。

### 版本3：基于pull的观察者模式

使用 JDK 内置的观察者 API 实现
```
public class WeatherData extends Observable {
    private float temperature;
    private float humidity;
    private float pressure;

    public WeatherData() {
    }

    public float getTemperature() {
        return temperature;
    }

    public float getHumidity() {
        return humidity;
    }

    public float getPressure() {
        return pressure;
    }

    private void measurementsChanged() {
        // 从java.util.Observable继承，线程安全
        // 拉模型实现，只是设置一个状态，如果状态位变了，就说明数据变更
        setChanged();
        // 从java.util.Observable继承，线程安全,只有当状态位为true，通知才有效，之后观察者们会主动拉取数据
        notifyObservers();
    }

    public void setMeasurements(float temperature, float humidity, float pressure) {
        this.temperature = temperature;
        this.humidity = humidity;
        this.pressure = pressure;
        measurementsChanged();
    }
}

public class CurrentConditionsDisplay implements Observer, DisplayElement {
    private Observable observable;
    private float temperature;
    private float humidity;

    public CurrentConditionsDisplay(Observable observable) {
        this.observable = observable;
        this.observable.addObserver(this);
    }

    public void removeObserver() {
        this.observable.deleteObserver(this);
    }

    @Override
    public void display() {
        System.out.println("Current conditions: " + temperature
                + "F degrees and " + humidity + "% humidity");
    }

    // 从java.util.Observer实现来的，实现的拉模型，arg是空
    // 额外的多了 Observable o 参数，让观察者能知道：到底是哪个主题通知的
    @Override
    public void update(Observable o, Object arg) {
        // 可以指定观察者只响应特定主题的通知，而不是默认强制的全部被通知
        if (o instanceof WeatherData) {
            WeatherData weatherData = (WeatherData) o;
            // 拉数据，让观察者具有了很大灵活性——自己决定需要什么数据
            this.temperature = weatherData.getTemperature();
            this.humidity = weatherData.getHumidity();
            display();
        }
    }
}
```
