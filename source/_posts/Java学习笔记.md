---
title: Java学习笔记
date: 2018-08-21 15:22:48
tags: 
 - Java
 - Spring
categories: IT
---
# Spring框架概念

- Spring是一个开源容器框架，可以接管web层，业务层，dao层，持久层的组件，并且可以配置各种bean,和维护bean与bean之间的关系。其核心就是控制反转(IOC)，和面向切面(AOP)，简单的说就是一个分层的轻量级开源框架。
## 控制反转(IoC) 依赖注入(DI)
依赖注入(Dependency Injection)和控制反转(Inversion of Control)是同一个概念。具体含义是:当某个角色(可能是一个Java实例，调用者)需要另一个角色(另一个Java实例，被调用者)的协助时，在传统的程序设计过程中，通常由调用者来创建被调用者的实例。但在Spring里，创建被调用者的工作不再由调用者来完成，因此称为控制反转;创建被调用者 实例的工作通常由Spring容器来完成，然后注入调用者，因此也称为依赖注入。
## 面向切面(AOP)
AOP主要实现的目的是针对业务处理过程中的切面进行提取，它所面对的是处理过程中的某个步骤或阶段，以获得逻辑过程中各部分之间低耦合性的隔离效果。

# Spring 事务机制总结

- Spring两种事务处理机制，一是声明式事务，二是编程式事务

## 声明式事务

Spring的声明式事务管理在底层是建立在AOP的基础之上的。其本质是对方法前后进行拦截，然后在目标方法开始之前创建或者加入一个事务，在执行完目标方法之后根据执行情况提交或者回滚事务。

声明式事务最大的优点就是不需要通过编程的方式管理事务，这样就不需要在业务逻辑代码中掺杂事务管理的代码，只需在配置文件中做相关的事务规则声明（或通过等价的基于标注的方式），便可以将事务规则应用到业务逻辑中。

因为事务管理本身就是一个典型的横切逻辑，正是AOP的用武之地。Spring开发团队也意识到了这一点，为声明式事务提供了简单而强大的支持。Spring强大的声明式事务管理功能，这主要得益于Spring依赖注入容器和Spring AOP的支持。

依赖注入容器为声明式事务管理提供了基础设施，使得Bean对于Spring框架而言是可管理的；而Spring AOP则是声明式事务管理的直接实现者。

和编程式事务相比，声明式事务唯一不足地方是，后者的**最细粒度**只能作用到方法级别，无法做到像编程式事务那样可以作用到代码块级别。但是即便有这样的需求，也存在很多变通的方法，比如，可以将需要进行事务管理的代码块独立为方法等等。

- 5种配置方式：
Spring配置文件中关于事务配置总是由三个组成部分，分别是DataSource、TransactionManager和代理机制这三部分，无论哪种配置方式，一般变化的只是代理机制这部分。

DataSource、TransactionManager这两部分只是会根据数据访问方式有所变化，比如使用Hibernate进行数据访问时，DataSource实际为SessionFactory，TransactionManager的实现为HibernateTransactionManager。
关系图如下：
![image](http://images.blogjava.net/blogjava_net/robbie/WindowsLiveWriter/Spring_9C9C/Spring%E4%BA%8B%E5%8A%A1%E9%85%8D%E7%BD%AE%20%282%29.jpg)


```
<bean id="sessionFactory"
	class="org.springframework.orm.hibernate3.LocalSessionFactoryBean">
	<property name="configLocation" value="classpath:hibernate.cfg.xml" />
	<property name="configurationClass" value="org.hibernate.cfg.AnnotationConfiguration" />
</bean>
 
<!-- 定义事务管理器（声明式的事务） -->
<bean id="transactionManager" class="org.springframework.orm.hibernate3.HibernateTransactionManager">
	<property name="sessionFactory" ref="sessionFactory" />
</bean>

```
注意：sessionFactorty和transactionManager是下面5中配置方式的基本配置

### 第一种方式：每个Bean都有一个代理

```
<!-- 配置DAO -->
<bean id="userDaoTarget" class="com.test.spring.dao.UserDaoImpl">
	<property name="sessionFactory" ref="sessionFactory" />
</bean>
 
<bean id="userDao" class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean">
	<!-- 配置事务管理器 -->
	<property name="transactionManager" ref="transactionManager" />
	<property name="target" ref="userDaoTarget" />
	<property name="proxyInterfaces" value="com.test.spring.dao.GeneratorDao" />
	<!-- 配置事务属性 -->
	<property name="transactionAttributes">
		<props>
			<prop key="*">PROPAGATION_REQUIRED</prop>
		</props>
	</property>
</bean>

```

### 第二种方式：所有Bean共享一个代理基类

```
<bean id="transactionBase" class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean" lazy-init="true" abstract="true">
	<!-- 配置事务管理器 -->
	<property name="transactionManager" ref="transactionManager" />
	<!-- 配置事务属性 -->
	<property name="transactionAttributes">
		<props>
			<prop key="*">PROPAGATION_REQUIRED</prop>
		</props>
	</property>
</bean>
 
<!-- 配置DAO -->
<bean id="userDaoTarget" class="com.test.spring.dao.UserDaoImpl">
	<property name="sessionFactory" ref="sessionFactory" />
</bean>
 
<bean id="userDao" parent="transactionBase">
	<property name="target" ref="userDaoTarget" />
</bean>

```

### 第三种方式：使用拦截器

```
<bean id="transactionInterceptor" class="org.springframework.transaction.interceptor.TransactionInterceptor">
	<property name="transactionManager" ref="transactionManager" />
	<!-- 配置事务属性 -->
	<property name="transactionAttributes">
		<props>
			<prop key="*">PROPAGATION_REQUIRED</prop>
		</props>
	</property>
</bean>
 
<bean class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator">
	<property name="beanNames">
		<list>
			<value>*Dao</value>
		</list>
	</property>
	<property name="interceptorNames">
		<list>
			<value>transactionInterceptor</value>
		</list>
	</property>
</bean>
 
<!-- 配置DAO -->
<bean id="userDao" class="com.test.spring.dao.UserDaoImpl">
	<property name="sessionFactory" ref="sessionFactory" />
</bean>
```
### 第四种方式：使用tx标签配置的拦截器    

```
<tx:advice id="txAdvice" transaction-manager="transactionManager">
	<tx:attributes>
		<tx:method name="*" propagation="REQUIRED" />
	</tx:attributes>
</tx:advice>
 
<aop:config>
	<aop:pointcut id="interceptorPointCuts"
		expression="execution(* com.test.spring.dao.*.*(..))" />
	<aop:advisor advice-ref="txAdvice" pointcut-ref="interceptorPointCuts" />
</aop:config>    

```

### 第五种方式：全注解

```
public class test {
	@Transactional
	public class UserDaoImpl extends HibernateDaoSupport implements UserDao {
 
		public List<User> listUsers() {
			return null
		}
	}
}

```

## 编程式事务

Spring的编程式事务即在代码中使用编程的方式进行事务处理，可以做到比声明式事务更细粒度。有两种方式一是使用TransactionManager，另外就是TransactionTemplate。

### TransactionManager使用方式

```
public class UserDaoImpl extends HibernateDaoSupport implements UserDao {
	private HibernateTransactionManager transactionManager;
	private DefaultTransactionDefinition def;
 
	public HibernateTransactionManager getTransactionManager() {
		return transactionManager;
	}
 
	public void setTransactionManager(HibernateTransactionManager transactionManager) {
		this.transactionManager = transactionManager;
	}
 
	public void createTransactionDefinition() {
		def = new DefaultTransactionDefinition();
		def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
		def.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
	}
 
	public void saveOrUpdate(User user) {
		TransactionStatus status = transactionManager.getTransaction(def);
		try {
			this.getHibernateTemplate().saveOrUpdate(user);
		} catch (DataAccessException ex) {
			transactionManager.rollback(status);
			throw ex;
		}
		transactionManager.commit(status);
	}
}

```

### TransactionTemplate方式


```
ResultDto ret = null;
ret = (ResultDto) this.transactionTemplate.execute(new TransactionCallback() {
	@Override
	public Object doInTransaction(TransactionStatus status) {
		ResultDto ret = null;
		try {
			drillTaskDao.deleteByKey(taskid);
		} catch (Exception e) {
			logger.error("delDrillTask:" + e.getMessage(), e);
			ret = ResultBuilder.buildResult(ResultBuilder.FAIL_CODE, null, ErrorCode.COM_DBDELETEERROR);
			return ret;
		}
		finally {
			status.setRollbackOnly();
		}
		 
		ret = cleartaskrelativedata(taskid, appid, true);
		return ret;
	}
});
return ret;

```


# 同步与异步
## java线程 同步与异步
多线程并发时，多个线程同时请求同一个资源，必然导致此资源的数据不安全，A线程修改了B线程的处理的数据，而B线程又修改了A线程处理的数理。显然这是由于全局资源造成的，有时为了解决此问题，优先考虑使用局部变量，退而求其次使用同步代码块，出于这样的安全考虑就必须牺牲系统处理性能，加在多线程并发时资源挣夺最激烈的地方，这就实现了线程的同步机制。
### 同步：
A线程要请求某个资源，但是此资源正在被B线程使用中，因为同步机制存在，A线程请求不到，怎么办，A线程只能等待下去；
### 异步：
A线程要请求某个资源，但是此资源正在被B线程使用中，因为没有同步机制存在，A线程
仍然请求的到，A线程无需等待；

显然，同步最最安全，最保险的。而异步不安全，容易导致死锁，这样一个线程死掉就会导致整个进程崩溃，但没有同步机制的存在，性能会有所提升。

java中实现多线程
> 1）继承Thread,重写里面的run方法

> 2）实现runnable接口
比较推荐后者，第一，java没有单继承的限制；第二，还可以隔离代码。

### Q&A
#### 1、 什么时候必须同步？什么叫同步？如何同步？
要跨线程维护正确的可见性，只要在几个线程之间共享非 final变量，就必须使用synchronized（或 volatile）以确保一个线程可以看见另一个线程做的更改。为了在线程之间进行可靠的通信，也为了互斥访问，同步是必须的。这归因于java语言规范的内存模型，它规定了：一个线程所做的变化何时以及如何变成对其它线程可见。因为多线程将异步行为引进程序，所以在需要同步时，必须有一种方法强制进行。例如：如果2个线程想要通信并且要共享一个复杂的数据结构，如链表，此时需要确保它们互不冲突，也就是必须阻止B线程在A线程读数据的过程中向链表里面写数据（A获得了锁，B必须等A释放了该锁）。为了达到这个目的，java在一个旧的的进程同步模型——监控器（Monitor）的基础上实现了一个巧妙的方案：监控器是一个控制机制，可以认为是一个很小的、只能容纳一个线程的盒子，一旦一个线程进入监控器，其它的线程必须等待，直到那个线程退出监控为止。通过这种方式，一个监控器可以保证共享资源在同一时刻只可被一个线程使用。这种方式称之为同步。（一旦一个线程进入一个实例的任何同步方法，别的线程将不能进入该同一实例的其它同步方法，但是该实例的非同步方法仍然能够被调用）。
- 错误的理解：同步嘛，就是几个线程可以同时进行访问。

同步和多线程关系：没多线程环境就不需要同步;有多线程环境也不一定需要同步。
锁提供了两种主要特性：**互斥**（mutual exclusion）和**可见性**（visibility）。
互斥即一次只允许一个线程持有某个特定的锁，因此可使用该特性实现对共享数据的协调访问协议，这样，一次就只有一个线程能够使用该共享数据。
可见性要更加复杂一些，它必须确保释放锁之前对共享数据做出的更改对于随后获得该锁的另一个线程是可见的——如果没有同步机制提供的这种可见性保证，线程看到的共享变量可能是修改前的值或不一致的值，这将引发许多严重问题
- 小结：为了防止多个线程并发对同一数据的修改，所以需要同步，否则会造成数据不一致（就是所谓的：线程安全。如java集合框架中Hashtable和Vector是线程安全的。我们的大部分程序都不是线程安全的，因为没有进行同步，而且我们没有必要，因为大部分情况根本没有多线程环境）。

#### 2、什么叫原子的（原子操作）？
Java原子操作是指：不会被打断的操作。

（就是做到互斥和可见性？）

那难道原子操作就可以真的达到线程安全同步效果了吗？实际上有一些原子操作不一定是线程安全的。
那么，原子操作在什么情况下不是线程安全的呢？也许是这个原因导致的：java线程允许线程在自己的内存区保存变量的副本。允许线程使用本地的私有拷贝进行工作而非每次都使用主存的值是为了提高性能（虽然原子操作是线程安全的，可各线程在得到变量（读操作）后，就是各自玩弄自己的副本了，更新操作（写操作）因未写入主存中，导致其它线程不可见）。

那该如何解决呢？因此需要通过java同步机制。

在java中，32位或者更少位数的赋值是原子的。在一个32位的硬件平台上，除了double和long型的其它原始类型通常都是使用32位进行表示，而double和long通常使用64位表示。另外，对象引用使用本机指针实现，通常也是32位的。对这些32位的类型的操作是原子的。

这些原始类型通常使用32位或者64位表示，这又引入了另一个小小的神话：原始类型的大小是由语言保证的。这是不对的。java语言保证的是原始类型的表数范围而非JVM中的存储大小。因此，int型总是有相同的表数范围。在一个JVM上可能使用32位实现，而在另一个JVM上可能是64位的。在此再次强调：在所有平台上被保证的是表数范围，32位以及更小的值的操作是原子的。
#### 3、不要搞混了：同步、异步
举个例子：普通B/S模式（同步）AJAX技术（异步）
- 同步：提交请求->等待服务器处理->处理完返回这个期间客户端浏览器不能干任何事
- 异步：请求通过事件触发->服务器处理（这是浏览器仍然可以作其他事情）->处理完毕

可见，彼“同步”非此“同步”——我们说的java中的那个共享数据同步（synchronized）

一个同步的对象是指行为（动作），一个是同步的对象是指物质（共享数据）。

#### 4、Java同步机制有4种实现方式：
① ThreadLocal ② synchronized( ) ③ wait() 与 notify() ④ volatile

目的：都是为了解决多线程中的对同一变量的访问冲突
##### ThreadLocal
ThreadLocal 保证不同线程拥有不同实例，相同线程一定拥有相同的实例，即为每一个使用该变量的线程提供一个该变量值的副本，每一个线程都可以独立改变自己的副本，而不是与其它线程的副本冲突。

- 优势：提供了线程安全的共享对象

与其它同步机制的区别：同步机制是为了同步多个线程对相同资源的并发访问，是为了多个线程之间进行通信；

而 ThreadLocal是隔离多个线程的数据共享，从根本上就不在多个线程之间共享资源，这样当然不需要多个线程进行同步了。
##### volatile
volatile 修饰的成员变量在每次被线程访问时，都强迫从共享内存中重读该成员变量的值。

而且，当成员变量发生变化时，强迫线程将变化值回写到共享内存。

优势：这样在任何时刻，两个不同的线程总是看到某个成员变量的同一个值。

缘由：Java 语言规范中指出，为了获得最佳速度，允许线程保存共享成员变量的私有拷贝，而且只当线程进入或者离开同步代码块时才与共享成员变量的原始值对比。这样当多个线程同时与某个对象交互时，就必须要注意到要让线程及时的得到共享成员变量的变化。而 volatile关键字就是提示VM：对于这个成员变量不能保存它的私有拷贝，而应直接与共享成员变量交互。

使用技巧：在两个或者更多的线程访问的成员变量上使用 volatile 。当要访问的变量已在synchronized 代码块中，或者为常量时，不必使用。

线程为了提高效率，将某成员变量(如A)拷贝了一份（如B），线程中对A的访问其实访问的是B。只在某些动作时才进行A和B的同步，因此存在A和B不一致的情况。volatile就是用来避免这种情况的volatile告诉jvm，它所修饰的变量不保留拷贝，直接访问主内存中的（读操作多时使用较好；线程间需要通信，本条做不到）

Volatile 变量具有synchronized的可见性特性，但是不具备原子特性。这就是说线程能够自动发现 volatile 变量的最新值。Volatile变量可用于提供线程安全，但是只能应用于非常有限的一组用例：多个变量之间或者某个变量的当前值与修改后值之间没有约束。只能在有限的一些情形下使用 volatile 变量替代锁。要使volatile变量提供理想的线程安全，必须同时满足下面两个条件：

> 对变量的写操作不依赖于当前值；

> 该变量没有包含在具有其他变量的不变式中。

##### sleep() vs wait()
sleep是线程类（Thread）的方法，导致此线程暂停执行指定时间，把执行机会给其他线程，但是监控状态依然保持，到时后会自动恢复。调用sleep不会释放对象锁。

wait是Object类的方法，对此对象调用wait方法导致本线程放弃对象锁，进入等待此对象的等待锁定池，只有针对此对象发出notify方法（或notifyAll）后本线程才进入对象锁定池准备获得对象锁进入运行状态。

（如果变量被声明为volatile，在每次访问时都会和主存一致；如果变量在同步方法或者同步块中被访问，当在方法或者块的入口处获得锁以及方法或者块退出时释放锁时变量被同步。）

# Mybatis

## jdbc
### jdbc编程步骤：
1、加载数据库驱动

2、创建并获取数据库链接

3、创建jdbc statement对象

4、设置sql语句

5、设置sql语句中的参数(使用preparedStatement)

6、通过statement执行sql并获取结果

7、对sql执行结果进行解析处理

8、释放资源(resultSet、preparedstatement、connection)

### jdbc问题总结如下：
1、数据库链接创建、释放频繁造成系统资源浪费从而影响系统性能，如果使用数据库链接池可解决此问题。

2、Sql语句在代码中硬编码，造成代码不易维护，实际应用sql变化的可能较大，sql变动需要改变java代码。

3、使用preparedStatement向占有位符号传参数存在硬编码，因为sql语句的where条件不一定，可能多也可能少，修改sql还要修改代码，系统不易维护。

4、对结果集解析存在硬编码（查询列名），sql变化导致解析代码变化，系统不易维护，如果能将数据库记录封装成pojo对象解析比较方便。

## Mybatis介绍
MyBatis 本是apache的一个开源项目iBatis,2010年这个项目由apache software foundation迁移到了google code，并且改名为MyBatis，实质上Mybatis对ibatis进行一些改进。

MyBatis是一个优秀的持久层框架，它对jdbc的操作数据库的过程进行封装，使开发者只需要关注 SQL 本身，而不需要花费精力去处理例如注册驱动、创建connection、创建statement、手动设置参数、结果集检索等jdbc繁杂的过程代码。

Mybatis通过xml或注解的方式将要执行的各种statement（statement、preparedStatemnt、CallableStatement）配置起来，并通过java对象和statement中的sql进行映射生成最终执行的sql语句，最后由mybatis框架执行sql并将结果映射成java对象并返回。

### Mybatis架构

1、mybatis配置SqlMapConfig.xml，此文件作为mybatis的全局配置文件，配置了mybatis的运行环境等信息。

mapper.xml文件即sql映射文件，文件中配置了操作数据库的sql语句。此文件需要在SqlMapConfig.xml中加载。

2、通过mybatis环境等配置信息构造SqlSessionFactory即会话工厂

3、由会话工厂创建sqlSession即会话，操作数据库需要通过sqlSession进行。

4、mybatis底层自定义了Executor执行器接口操作数据库，Executor接口有两个实现，一个是基本执行器、一个是缓存执行器。

5、Mapped Statement也是mybatis一个底层封装对象，它包装了mybatis配置信息及sql映射信息等。mapper.xml文件中一个sql对应一Mapped Statement对象，sql的id即是Mapped statement的id。

6、Mapped Statement对sql执行输入参数进行定义，包括HashMap、基本类型、pojo，Executor通过Mapped Statement在执行sql前将输入的java对象映射至sql中，输入参数映射就是jdbc编程中对preparedStatement设置参数。

7、Mapped Statement对sql执行输出结果进行定义，包括HashMap、基本类型、pojo，Executor通过Mapped Statement在执行sql后将输出结果映射至java对象中，输出结果映射过程相当于jdbc编程中对结果的解析处理过程。

###  Mybatis下载
mybaits的代码由github.com管理，地址：https://github.com/mybatis/mybatis-3/releases

mybatis-3.2.7.jar----mybatis的核心包
lib----mybatis的依赖包

mybatis-3.2.7.pdf----mybatis使用手册
HashMap实现原理分析

# HashMap的数据结构

数据结构中有数组和链表来实现对数据的存储，但这两者基本上是两个极端。
## 常见数据结构

### 数组
数组存储区间是连续的，占用内存严重，故空间复杂的很大。但数组的二分查找时间复杂度小，为O(1)；

数组的特点是：寻址容易，插入和删除困难；
    
## 链表
链表存储区间离散，占用内存比较宽松，故空间复杂度很小，但时间复杂度很大，达O(N)。

链表的特点是：寻址困难，插入和删除容易。

## 哈希表

那么我们能不能综合两者的特性，做出一种寻址容易，插入删除也容易的数据结构？答案是肯定的，这就是我们要提起的哈希表。

哈希表（(Hashtable）既满足了数据的查找方便，同时不占用太多的内容空间，使用也十分方便。

哈希表是由数组+链表组成的，一个长度为16的数组中，每个元素存储的是一个链表的头结点。那么这些元素是按照什么样的规则存储到数组中呢。一般情况是通过hash(key)%len获得，也就是元素的key的哈希值对数组长度取模得到。比如上述哈希表中，12%16=12,28%16=12,108%16=12,140%16=12。所以12、28、108以及140都存储在数组下标为12的位置。

HashMap其实也是一个线性的数组实现的,所以可以理解为其存储数据的容器就是一个线性数组。这可能让我们很不解，一个线性的数组怎么实现按键值对来存取数据呢？这里HashMap有做一些处理。

HashMap里面实现一个静态内部类Entry，其重要的属性有key,value,next，从属性key,value我们就能很明显的看出来Entry就是HashMap键值对实现的一个基础bean，我们上面说到HashMap的基础就是一个线性数组，这个数组就是Entry[]，Map里面的内容都保存在Entry[]里面。

# REST 
- REST-- REpresentational State Transfer，英语的直译就是“表现层状态转移”。如果看这个概念，估计没几个人能明白是什么意思。那下面就让我来用一句人话解释一下什么是RESTful:URL定位资源，用HTTP动词（GET,POST,PUT,DELETE)描述操作。

**Resource**：资源，即数据。
Representational：某种表现形式，比如用JSON，XML，JPEG等；
**State Transfer**：状态变化。通过HTTP动词实现。

所以RESTful API就是REST风格的API。那么在什么场景下使用RESTful API呢？

在当今的互联网应用的前端展示媒介很丰富。有手机、有平板电脑还有PC以及其他的展示媒介。那么这些前端接收到的用户请求统一由一个后台来处理并返回给不同的前端肯定是最科学和最经济的方式，RESTful API就是一套协议来规范多种形式的前端和同一个后台的交互方式。

## RESTful API设计原则和规范

RESTful API由后台也就是SERVER来提供前端来调用。前端调用API向后台发起HTTP请求，后台响应请求将处理结果反馈给前端。也就是说RESTful 是典型的基于HTTP的协议。那么RESTful API有哪些设计原则和规范呢？

### 1，资源

首先是弄清楚资源的概念。资源就是网络上的一个实体，一段文本，一张图片或者一首歌曲。资源总是要通过一种载体来反应它的内容。文本可以用TXT，也可以用HTML或者XML、图片可以用JPG格式或者PNG格式，JSON是现在最常用的资源表现形式。

### 2，统一接口
RESTful风格的数据元操CRUD（create,read,update,delete）分别对应HTTP方法：GET用来获取资源，POST用来新建资源（也可以用于更新资源），PUT用来更新资源，DELETE用来删除资源，这样就统一了数据操作的接口。

### 3，URI
可以用一个URI（统一资源定位符）指向资源，即每个URI都对应一个特定的资源。要获取这个资源访问它的URI就可以，因此URI就成了每一个资源的地址或识别符。一般的，每个资源至少有一个URI与之对应，最典型的URI就是URL。

### 4，无状态
所谓无状态即所有的资源都可以URI定位，而且这个定位与其他资源无关，也不会因为其他资源的变化而变化。

有状态和无状态的区别，举个例子说明一下，例如要查询员工工资的步骤为：
> 第一步：登录系统。

> 第二步：进入查询工资的页面。

> 第三步：搜索该员工。

> 第四步：点击姓名查看工资。

这样的操作流程就是有状态的，查询工资的每一个步骤都依赖于前一个步骤，只要前置操作不成功，后续操作就无法执行。

如果输入一个URL就可以得到指定员工的工资，则这种情况就是无状态的，因为获取工资不依赖于其他资源或状态，且这种情况下，员工工资是一个资源，由一个URL与之对应可以通过HTTP中的GET方法得到资源，这就是典型的RESTful风格。

## 实例

说了这么多，到底RESTful长什么样子的呢？
```
GET:http://www.xxx.com/source/id
```
 获取指定ID的某一类资源。例如GET:http://www.xxx.com/friends/123表示获取ID为123的会员的好友列表。如果不加id就表示获取所有会员的好友列表。


```
POST:http://www.xxx.com/friends/123
```
表示为指定ID为123的会员新增好友。其他的操作类似就不举例了。

## 其他规范

RESTful API还有其他一些规范。

1：应该将API的版本号放入URL。

```
GET:http://www.xxx.com/v1/friend/123。
```
或者将版本号放在HTTP头信息中。我个人觉得要不要版本号取决于自己开发团队的习惯和业务的需要，不是强制的。

2：URL中只能有名词而不能有动词，操作的表达是使用HTTP的动词GET,POST,PUT,DELETEL。URL只标识资源的地址，既然是资源那就是名词了。

3：如果记录数量很多，服务器不可能都将它们返回给用户。API应该提供参数，过滤返回结果。?limit=10：指定返回记录的数量、?page=2&per_page=100：指定第几页，以及每页的记录数。