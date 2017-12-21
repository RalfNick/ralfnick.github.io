---
layout: post
title: "Observer Design Pattern"
date: 2017-12-17
description: "学习设计模式，提升内功"
tag: Java
---


### 1. 什么是观察者模式？
观察者模式，又称为【发布-订阅模式】，可以理解为报刊社发布新刊，订阅者获取新期刊，订阅者就相当于是观察者，而且可以有很多观察者，报刊社就是被观察的对象。
用现实中的例子比喻一下，学校里面有一个小报刊亭（被观察者），有些学生（观察者）在报刊亭订阅报刊。那具体怎么订阅呢？就是你交了钱，然后留下姓名和手机号给老板（注册过程）。当期刊有更新，报刊亭老板就给你打电话，告诉你期刊更新了，
然后你就可以去取报刊（通过过程）。当然，看了大概半年后你不想再订阅，那么你告诉老板说不再订阅了，也就是不再续费了。老板会将你的姓名和电话从他的小本本中删除了，即以后不再通知你报刊更新的信息了（取消注册过程）。
这就是观察者模式的一个小例子，应该很好理解吧！

### 2. 观察者模式练习例子

(1)举《Head First设计模式》中的例子，天气情况更细。不过这里我做了更改，改为天气更新，学生和工人接收天气信息变化。
气象站--->被观察者，学生--->观察者，工人--->观察者，天气若更新，学生和工人就能获取到最新的气象信息，观察者模式的好处就是，不管有多少观察者只要注册后，就可以获取被观察者对象的信息，即被观察者的代码可以不用跟着变化做修改了。


(2)两个接口类设计

仿照源码中的模式，也设计Observer接口和Subject接口，然后由具体的类去实现。
下面看一下类图关系



Observer接口，只有一个天气更新的接口函数

```java
public interface Observer {
	public void update(Subject subject,Object o);
}
```

Subject接口，设计注册，移除，通知观察者三个函数。

```java

public interface Subject {

	public void registerObserver(Observer o);
	public void removeObserver(Observer o);
	public void notifyObservers(Object args);
}
```

(3)设计实现子类，其中WeatherSender实现Subject借口哦，Student和Worker实现Observer接口

WeatherSender类

```java
public class WeatherSender implements Subject{

	private Weather weather;
	private List<Observer> observers;
	private static WeatherSender weatherSender;
	private boolean isChanged;

	private WeatherSender(){
		if(this.observers == null){
			this.observers = new ArrayList<>();
		}
	}
	public static WeatherSender getWeatherSender(){

		if(weatherSender == null){
			return new WeatherSender();
		}
		else return weatherSender;

	}

	public void registerObserver(Observer o) {

		this.observers.add(o);
	}

	public void removeObserver(Observer o) {
		if(observers.contains(o)){
			this.observers.remove(o);
		}
	}

	private void setChanged(){
		isChanged = true;
	}

	/**
	 * 采用“push”方式通知观察者（主动式）
	 * @param obj
	 */
	public void notifyObservers(Object obj) {
		if(isChanged){
			for(Observer observer : observers){
				observer.update(this, obj);
			}
		}
	}

	/**
	 * 采用“pull”方式通知观察者（被动式）
	 */
	public void notifyObservers(){
		notifyObservers(null);
	}

	/**
	 * "pull"方式下供观察者调用获取天气
	 * @return
	 */
	public Weather getWeather(){
		return this.weather;
	}

	public void setWeather(Weather weather){
		this.weather = weather;
	}

	/**
	 * 供外面调用的接口
	 * @param weather
	 */
	public void setWeatherChanged(Weather weather){

		setChanged();
		notifyObservers(weather);//push方式

		/*
		setWeather(weather);
		notifyObservers();//pull方式
		*/
	}
}
```
**注意**：在notifyObserver()方法设计上，在head First设计模式中提到了两种方式，push和pull方式。这里我把它们称为主动式（push）和被动式（pull）
>* 主动式(push):由Subject将weather通过notifyObservers(Object obj)方法传递给Observer，即将weather推给Observer

>* 被动式(pull):Subject不传递weather变量，而是Observer通过获取WeatherSender，通过它提供的方法自己获取天气信息，即由Observer自己将weather拉过来

---
Student类

```java
public class Student implements Observer{

	private Weather weather;
	private Subject subject;

	public Student(Subject subject){
		this.subject = subject;
		subject.registerObserver(this);
	}
	public Weather getWeather() {
		return weather;
	}

	@Override
	public void update(Subject subject, Object o) {

		//push方式
		if(o instanceof Weather){
			this.weather = (Weather)o;
		}

		/*
		//pull方式
		if(subject instanceof WeatherSender){
			this.weather = ((WeatherSender) subject).getWeather();
		}
		*/

	}

	public void displayWeather(){
		System.out.println("我是Student，当前天气情况：温度 "
				+ weather.getTemperature()+ " 压力 " + weather.getPressure());
	}

}
```
---
Worker 类

```java
public class Worker implements Observer{

	private Weather weather;
	private Subject subject;

	public Worker(Subject subject){
		this.subject = subject;
		subject.registerObserver(this);
	}
	public Weather getWeather() {
		return weather;
	}

	@Override
	public void update(Subject subject, Object o) {
		if(o instanceof Weather){
			this.weather = (Weather)o;
		}

	}

	public void displayWeather(){
		System.out.println("我是Worker，当前天气情况：温度 "
				+ weather.getTemperature()+ " 压力 " + weather.getPressure());
	}

}

```
(4)剩下就是一个Weather类和测试类

WeatherSender类

```java
public class Weather {

	private String temperature;
	private String pressure;
	public String getTemperature() {
		return temperature;
	}
	public Weather(String temperature,String pressure){

		this.temperature = temperature;
		this.pressure = pressure;
	}
	public void setTemperature(String temperature) {
		this.temperature = temperature;
	}
	public String getPressure() {
		return pressure;
	}
	public void setPressure(String pressure) {
		this.pressure = pressure;
	}

}
```
---
测试类Test

```java

public class Test {

	/**
	 * @param args
	 */
	public static void main(String[] args) {

		//创建被观察者
		WeatherSender weatherSender = WeatherSender.getWeatherSender();

		//创建观察者,并注册
		Student student = new Student(weatherSender);
		Worker worker = new Worker(weatherSender);

		//第一天
		Weather weatherFirstDay = new Weather("23℃","1.01*10^5 Pa");
		weatherSender.setWeatherChanged(weatherFirstDay);
		System.out.println("第一天 天气情况");
		student.displayWeather();
		System.out.println("------------------");
		worker.displayWeather();
		System.out.println("------------------");
		//第二天
		Weather weatherSecondDay = new Weather("24℃","1.02*10^5 Pa");
		weatherSender.setWeatherChanged(weatherSecondDay);

		System.out.println("第二天 天气情况");
		student.displayWeather();
		System.out.println("------------------");
		worker.displayWeather();
		System.out.println("------------------");
		//第三天
		Weather weatherThirdDay = new Weather("25℃","1.03*10^5 Pa");
		weatherSender.setWeatherChanged(weatherThirdDay);

		System.out.println("第三天 天气情况");
		student.displayWeather();
		System.out.println("------------------");
		worker.displayWeather();
		System.out.println("------------------");
	}

}
```
### 3.源码分析

源码Observer接口,这个和上面自己写的例子是一样的
```java
/**
 * A class can implement the <code>Observer</code> interface when it
 * wants to be informed of changes in observable objects.
 *当一个类在它所观察的对象变化时能够得到通知，它需要实现Observer接口
 */
public interface Observer {
	void update(Observable o, Object arg);
}
```
---

源码Observable抽象类，源码中并没有将Observable设计成接口，个人认为主要是里面公用的方法不要由子类去实现，减少代码重复吧！
下面把源码粘贴过来，看一下。

```java

/**
 * This class represents an observable object, or "data"
 * in the model-view paradigm. It can be subclassed to represent an
 * object that the application wants to have observed.
 这个类是被观察的对象，或者说是MV模型中的数据。它可以被子类化，作为程序中需要观察的对象
 * <p>
 * An observable object can have one or more observers. An observer
 * may be any object that implements interface <tt>Observer</tt>. After an
 * observable instance changes, an application calling the
 * <code>Observable</code>'s <code>notifyObservers</code> method
 * causes all of its observers to be notified of the change by a call
 * to their <code>update</code> method.
 * 被观察的对象有一个或者多个观察者。观察者可能需要实现Observer接口，当被观察对象实例发生变化时
 * ，程序会调用Observable的notifyObservers方法，并通过回调观察者的update方法，来通知观察者
 * <p>
 * The order in which notifications will be delivered is unspecified.
 * The default implementation provided in the Observable class will
 * notify Observers in the order in which they registered interest, but
 * subclasses may change this order, use no guaranteed order, deliver
 * notifications on separate threads, or may guarantee that their
 * subclass follows this order, as they choose.
 * 通知发送的顺序是不固定的。Observable 类中所提供的默认实现将按照其注册的顺序来通知观察者，
 * 但是子类可能改变此顺序，从而使用非固定顺序在单独的线程上发送通知，
 * 或者也可能保证其子类遵从其所选择的顺序。
 * <p>
 * Note that this notification mechanism has nothing to do with threads
 * and is completely separate from the <tt>wait</tt> and <tt>notify</tt>
 * mechanism of class <tt>Object</tt>.
 * 注意这个通知机制和线程无关，并且要和Object的等待唤醒机制区分开
 * <p>
 * When an observable object is newly created, its set of observers is
 * empty. Two observers are considered the same if and only if the
 * <tt>equals</tt> method returns true for them.
 *当一个被观察对象创建后，它的观察者集合时空的。
 *两个观察者当且仅当equals方法返回true时认为他们是同一个观察者
 */
public class Observable {
    private boolean changed = false;
    private Vector<Observer> obs;//源码中使用Vector，保证线程安全

    /** Construct an Observable with zero Observers. */

    public Observable() {
        obs = new Vector<>();
    }

    /**
     * Adds an observer to the set of observers for this object, provided
     * that it is not the same as some observer already in the set.
     * The order in which notifications will be delivered to multiple
     * observers is not specified. See the class comment.
     *
     * @param   o   an observer to be added.
     * @throws NullPointerException   if the parameter o is null.
     */
    public synchronized void addObserver(Observer o) {
        if (o == null)
            throw new NullPointerException();
        if (!obs.contains(o)) {
            obs.addElement(o);
        }
    }

    /**
     * Deletes an observer from the set of observers of this object.
     * Passing <CODE>null</CODE> to this method will have no effect.
     * @param   o   the observer to be deleted.
     */
    public synchronized void deleteObserver(Observer o) {
        obs.removeElement(o);
    }

		//这个方法就是采用“pull”的方法
    public void notifyObservers() {
        notifyObservers(null);
    }

		//这个方法是采用“push”的方法
    public void notifyObservers(Object arg) {
        /*
         * a temporary array buffer, used as a snapshot of the state of
         * current Observers.
         */
        Object[] arrLocal;

        synchronized (this) {
            /* We don't want the Observer doing callbacks into
             * arbitrary code while holding its own Monitor.
             * The code where we extract each Observable from
             * the Vector and store the state of the Observer
             * needs synchronization, but notifying observers
             * does not (should not).  The worst result of any
             * potential race-condition here is that:
             * 1) a newly-added Observer will miss a
             *   notification in progress
             * 2) a recently unregistered Observer will be
             *   wrongly notified when it doesn't care
						 * 从被观察对象的Vector中取出的code，包含观察者的状态，需要线程同步，但是在
						 * 通知观察者时，不需要同步。潜在的资源争夺下，最糟糕的情况：
						 * （1）一个新加入的观察者将会错过正在进行消息通知
						 * （2）一个刚刚取消注册的观察者将会“很无奈的”接收他不想接受的消息
             */
            if (!changed)
                return;
            arrLocal = obs.toArray();
            clearChanged();
        }

        for (int i = arrLocal.length-1; i>=0; i--)
            ((Observer)arrLocal[i]).update(this, arg);
    }

    /**
     * Clears the observer list so that this object no longer has any observers.
     */
    public synchronized void deleteObservers() {
        obs.removeAllElements();
    }

    /**
     * Marks this <tt>Observable</tt> object as having been changed; the
     * <tt>hasChanged</tt> method will now return <tt>true</tt>.
     */
    protected synchronized void setChanged() {
        changed = true;
    }

    /**
     * Indicates that this object has no longer changed, or that it has
     * already notified all of its observers of its most recent change,
     * so that the <tt>hasChanged</tt> method will now return <tt>false</tt>.
     * This method is called automatically by the
     * <code>notifyObservers</code> methods.
     *
     * @see     java.util.Observable#notifyObservers()
     * @see     java.util.Observable#notifyObservers(java.lang.Object)
     */
    protected synchronized void clearChanged() {
        changed = false;
    }

    /**
     * Tests if this object has changed.
     *
     * @return  <code>true</code> if and only if the <code>setChanged</code>
     *          method has been called more recently than the
     *          <code>clearChanged</code> method on this object;
     *          <code>false</code> otherwise.
     * @see     java.util.Observable#clearChanged()
     * @see     java.util.Observable#setChanged()
     */
    public synchronized boolean hasChanged() {
        return changed;
    }

    /**
     * Returns the number of observers of this <tt>Observable</tt> object.
     *
     * @return  the number of observers of this object.
     */
    public synchronized int countObservers() {
        return obs.size();
    }
}
```
### 4. 总结
源码和上面自己写的例子中的基本思路是一致，源码中设计的更加全面，加入了线程安全。

观察者模式在java源码和android源码中都有很多应用，如在android中的监听器，控件设置点击事件，注册监听器，就是观察者模式。所以理解设计模式对于读源码有很大帮助！

**参考** 《Head First设计模式》
