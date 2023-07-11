---
title: Spring事务的隔离级别和传播行为以及常见的事务注解失效的原因
date: 2020-05-29 00:21:52
tags:
- Java
- 面试
- Spring
- 事务
categories:
- Java
---
> 在日常开发中事务的重要性不言而喻,而Spring作为日常使用的最多的Java框架对事务的处理有一套自己的准则,现在我们就来看以下Spring中事务的隔离级别和传播行为.
<!-- more -->

## Spring的事务隔离级别和传播行为

这个问题也是Java面试中的常考察的题目.

### 事务的4种特性

#### 一致性

事务执行前后都要保持一致性状态.在一致性状态下,所有事务对一个数据的读取结果都是相同的.

#### 原子性

事务被视为不可分割的最小单元,要么全部成功,要么全部失败回滚.

#### 持久性

一旦事务提交,则其所做的修改将会永久保存到数据库中.即使系统发生崩溃,事务的执行结果也不能丢失.可以通过数据库备份和恢复来保证持久性.

#### 隔离性

一个事务所做的修改在最终提交前,对其他事务是不可见的.

### 隔离级别

**四种隔离级别**, 是基于数据库隔离级别的封装(✔️表示允许,❌表示不允许):

|隔离级别|脏读|不可重复读|幻读|
|--|:--:|:--:|:--:|
|**默认(Default使用数据库默认的隔离级别)**|-|-|-|
|**未提交读(Read_UnCommited)**|✔️|✔️|✔️|
|**已提交读(Read_Commited)**|❌|✔️|✔️|
|**可重复读(Repeatable_Read)**|❌|❌|✔️|
|**串行化的(Serializable)**|❌|❌|❌|

### 传播行为

**七种传播行为**:

- **Propagation.REQUIRED**
支持当前事务,如果当前存在事务则加入当前事务;如果当前没有事务,则新建一个事务进行.

- **Propagation.SUPPORTS**
支持当前事务,如果当前存在事务则加入当前事务,如果当前没有事务,则以非事务方式运行.

- **Propagation.MANDATORY**
支持当前事务,如果当前存在事务则加入当前事务,如果当前没有事务则抛出异常.

- **Propagation.REQUIRES_NEW**
不支持当前事务,如果当前存在事务,则当前事务挂起, 另起一个新的事务运行,这两个事务是相互独立的,不会相互影响.

- **Propagation.NOT_SUPPORT**
不支持当前事务,如果当前存在事务,则当前事务挂起, 自身以非事务方式运行.

- **Propagation.NEVER**
不支持当前事务,如果当前存在事务,则抛出异常,如果当前不存在事务,则以非事务方式运行.

- **Propagation.NESTED**
嵌套事务, 和requires_new的区别是当前事务的提交依赖于父事务的提交, 若父事务回滚, 则当前事务也会回滚.另外当前方法调用子方法时,子方法的发生异常被捕获了,则只有子方法回滚事务,当前事务仍然可以运行.基于数据库的**SAVEPOINT**技术,若数据库不支持此技术,则会新建一个事务去运行,此时就相当于REQUIRES_NEW.

## Spring的@Transactional注解不生效的原因

### 1. 数据库引擎不支持事务

以MySQL为例, MyISAM引擎不支持事务, 需要InnoDB引擎才支持.

### 2. 当前Bean没有被Spring管理

没有被Spring管理的类,这个类不会被Spring加载,其方法再怎么加注解当然会失效.

### 3. 方法不是public的

Spring官方文档说明@Transactional注解只能作用于**pubilic**方法上

### 4. 自身调用问题

若自身非事务方法调用了一个事务方法,获取一个自身默认传播行为的事务方法调用自身另一个REQUIRES_NEW传播行为的方法,则当前事务会被挂起,导致不生效.

### 5. 数据源未配置transactionManager

```java
@Bean
public PlatformTransactionManager transactionManager(DataSource dataSource) {
    return new DataSourceTransactionManager(dataSource);
}
```

需要如上配置

### 6. 调用了不支持事务的方法Propagation.NOT_SUPPORT

```java
@Service
public class AServiceImpl implements AService {

    @Transactional
    public void a() {
        b();
    }

    @Transactional(propagation = Propagation.NOT_SUPPORTED)
    public void b() {
        // do sth
    }

}
```

如上a的事务会被挂起,造成事务不生效.

### 7. 异常被catch,且未抛出异常

```java
@Service
public class AServiceImpl implements AService {

    @Transactional
    public void a() {
        try {
            // do sth
        } catch(Exception e) {
            // not throw exception
        }
    }

}
```

### 8. 指定异常类型错误

```java
@Service
public class AServiceImpl implements AService {

    @Transactional
    public void a() {
        try {
            // do sth
        } catch(Exception e) {
            // throw new RuntimeException("出现错误"); // 事务生效
            throw new Exception("出现错误"); // 事务不会生效
        }
    }

}
```

第一种抛出`RuntimeException`是可以触发事务的回滚的, 因为Spring默认回滚的事务就是`RuntimeException`, 第二种`Exception`就不会生效了,想让其生效必须指定`@Transactional(rollbackFor=Exception.class)`
> 注意: rollbackFor仅限**Throwable异常**及其**子类**

