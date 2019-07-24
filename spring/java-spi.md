## SPI简介 ##
> SPI: 全称Service Provider Interface，是Java提供的一套用来被第三方实现或者扩展的API，它可以用来启用框架扩展和替换组件。     

Java SPI实际上是**”基于接口的编程+策略模式+配置文件“**组合实现的动态加载机制。
系统里抽象的各个模块，往往 有很多不同的实现方案，在面向对象的设计里，一般荐模块之间基于接口编程，模块之间不对实现类进行硬编码。一旦代码里涉及具体的实现类，就违反了可拔插的原则，如果需要替换一种实现，就需要修改代码。为了实现在模块装配的时候能不在程序里动态指明，这就需要一种服务发现机制。

Java SPI就提供这样的一个机制：为某个接口寻找服务实现的机制。有点类似IOC的思想，就是将装配的控制权移到程序之外，在模块化设计中这个机制尤其重要。所以SPI的核心思想就是解耦。

## SPI约定 ##
**Java SPI的具体约定为**：当服务的提供者，提供了服务接口的一种实现之后，在jar包的META-INF/services/目录里同时创建一个以服务接口命名的文件。该文件里就是实现该服务接口的具体实现类。而当外部程序装配这个模块的时候，就能通过该jar包META-INF/services/里的配置文件找到具体的实现类名，并装载实例化，完成模块的注入。 基于这样一个约定就能很好的找到服务接口的实现类，而不需要再代码里制定。jdk提供服务实现查找的一个工具类：java.util.ServiceLoader。

## SPI示例代码 ##
**项目结构：**
	
- spi-api: 定义一组接口，只提供接口，不提供实现

```
public interface Barking {  
   void barking(); // 动物叫声   
}
```

- spi-cat: 针对spi-api的Barking接口的一个或多个实现，依赖spi-api项目

```
public class Cat implements Barking {
	public void barking() {
		System.out.println("喵喵叫...");
	}
}
```
同事需要在src/main/resources/目录下面添加如下文件  

- META-INF  
  - services  
    - com.zephyr.demo.Barking  

文件“com.zephyr.demo.Barking”中的内容是：“com.zephyr.demo.Cat”

- spi-test: 用来模拟用户测试
```
public class Test {
	public static void main(String[] args) {
		ServiceLoader<Barking> services = ServiceLoader.load(Barking.class);
		Iterator<Barking> iterator = services.iterator();
		while (iterator.hasNext()) {
			Barking barking = iterator.next();
			barking.barking();
		}
	}
}
```

## 应用场景 ##
1. JDBC加载不同类型的驱动
2. SLF4J对log4j/logback的支持
3. Spring中大量使用了SPI,比如：对servlet3.0规范对ServletContainerInitializer的实现、自动类型转换Type Conversion SPI(Converter SPI、Formatter SPI)等
4. Dubbo中也大量使用SPI的方式实现框架的扩展, 不过它对java提供的原生SPI做了封装
5. 更多应用场景需要大家一起去发现，或者自己使用SPI机制实现代码的解耦

## 总结 ##
**优点：**
> 使用Java SPI机制的优势是实现解耦，使得第三方服务模块的装配控制的逻辑与调用者的业务代码分离，而不是耦合在一起。应用程序可以根据实际业务情况启用框架扩展或替换框架组件。

**缺点：**
> 1、虽然ServiceLoader也算是使用的延迟加载，但是基本只能通过遍历全部获取，也就是接口的实现类全部加载并实例化一遍。如果你并不想用某些实现类，它也被加载并实例化了，这就造成了浪费。获取某个实现类的方式不够灵活，只能通过Iterator形式获取，不能根据某个参数来获取对应的实现类。 

> 2、多个并发多线程使用ServiceLoader类的实例是不安全的。


## 参考 ##
[Java中的SPI机制深入及源码解析](https://cxis.me/2017/04/17/Java%E4%B8%ADSPI%E6%9C%BA%E5%88%B6%E6%B7%B1%E5%85%A5%E5%8F%8A%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/)

[Java SPI机制](http://www.spring4all.com/article/260)