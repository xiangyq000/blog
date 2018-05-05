---
title: 《Spring 3.x 企业应用开发实战》：DAO
date: 2016-03-17 20:52:00
tags: Spring
categories: [Spring, 读书笔记]
---

## 1. 概念
1. Spring 的 DAO 异常体系建立在运行期异常的基础上，封装了源异常
2. JDBC 数据访问流程：
    - 准备资源
    - 启动事务
    - 在事务中执行具体数据访问操作
    - 提交/回滚事务
    - 关闭资源，处理异常

<!-- more -->

3. Spring 将相同的数据访问流程固化到模板类中，把数据访问中固定和变化的部分分开，同时保证模板类是线程安全的。Spring 为不同的持久化技术都提供了简化操作的模板和回调。
4. 数据库事务：原子性，一致性，隔离性和持久性（ACID）
5. 5 类数据库并发问题：
    - 脏读：A 事务读取到 B 事务尚未提交的数据
    - 不可重复读：A 事务中读取到 B 事务已经提交的更新数据，即连续两次读取结果不同
    - 幻读：A 事务读取 B 事务的==新增==数据
    - 第一类更新丢失：A 事务撤销时覆盖了 B 事务的提交
    - 第二类更新丢失：A 事务覆盖 B 事务已经提交的数据
6. JDBC 默认情况下自动提交，即每条执行的 SQL 语句都对应一个事务，`AutoCommit = TRUE`
7. Spring基于 `ThreadLocal` 解决有状态的 `Connetion` 的并发问题，事务同步管理器 `org.springframework.transaction.support.TransactionSynchronizationManager` 使用 `ThreadLocal` 为不同事务线程提供独立的资源副本
8. Spring 事务管理基于 3 个接口：`TransactionDefinition`，`TransactionStatus` 和 `PlatformTransactionManager`
9. Spring 为不同持久化技术提供了从 `TransactionSynchronizationManager` 获取对应线程绑定资源的工具类，如 `DataSourceUtils.getConnection(DataSource dataSource)`。模板类在内部通过工具类访问 `TransactionSynchronizationManager`中的线程绑定资源
10. Spring 通过事务传播行为控制当前的事务如何传播到被嵌套调用的目标服务接口方法中
11. 使用 `<tx:annotation-driven transaction-manager="txManager">` 对标注 `@Transactional` 注解的 bean 进行加工处理，织入事务管理切面
12. `@Transactional` 注解的属性
    - 事务传播行为：`propagation`，默认 `PROPAGATION_REQUIRED`，即如果当前没有事务，就新建一个事务；否则加入到当前事务
    - 事务隔离级别：`isolation`，默认 `ISOLATION_DEFAULT`
    - 读写事务属性：`readOnly`
    - 超时时间：`timeout`
    - 回滚设置：`rollbackFor`，`rollbackForClassName`，`noRollbackFor`，`noRollbackForClassName`
13. 在相同线程中进行相互嵌套调用的事务方法工作于相同的事务中；如果在不同线程中，则工作在独立的事务中
14. 特殊方法：
    - 注解不能被继承，所以业务接口中的 `@Transactional` 注解不会被业务实现类继承；方法处的注解会覆盖类定义处的注解
    - 对于基于接口动态代理的 AOP 事务，由于接口方法都是 `public` 的，实现类的实现方法必须是 `public `的，同时不能使用 `static` 修饰符。因此，可以通过接口动态代理实施 AOP 增强、实现 Spring 事务的方法只能是 `public` 或 `public final` 的
    - 基于 CGLib 动态代理实施 AOP 的时候，由于使用 `final`、`static`、`private` 的方法不能被子类覆盖，相应的，这些方法不能实施 AOP 增强，实现事务
    - 不能被 Spring 进行 AOP 事务增强的方法不能启动事务，但是外层方法的事务上下文仍然可以传播到这些方法中

## 2. Spring 中使用 JDBC 编程示例
- 本地 mysql 建表

```sql
CREATE TABLE `t_user` (
  `user_id` varchar(256) NOT NULL,
  `user_name` varchar(256) DEFAULT NULL,
  PRIMARY KEY (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

- `springDAO.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xmlns:p="http://www.springframework.org/schema/p"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/context
    http://www.springframework.org/schema/context/spring-context.xsd
    http://www.springframework.org/schema/tx
    http://www.springframework.org/schema/tx/spring-tx.xsd ">

    <context:component-scan base-package="com.dao" />

    <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource"
    destroy-method="close"
        p:driverClassName="com.mysql.jdbc.Driver"
        p:url="jdbc:mysql://localhost:3306/sampledb"
        p:username="root"
        p:password="123123" />

    <bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate"
        p:dataSource-ref="dataSource" />

    <bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager"
        p:dataSource-ref="dataSource" />
</beans>
```

- `User`

```java
package com.data;

public class User {

    private String userId;

    private String userName;

    public String getUserId() {
        return userId;
    }

    public void setUserId(String userId) {
        this.userId = userId;
    }

    public String getUserName() {
        return userName;
    }

    public void setUserName(String userName) {
        this.userName = userName;
    }

}
```

- `BaseDAO`

```java
package com.dao;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;

public class BaseDAO {

    @Autowired
    protected JdbcTemplate jdbcTemplate;
}
```

- `UserDAO`

```java
package com.dao;

import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import com.data.User;
import com.mapper.UserRowMapper;

@Repository
public class UserDAO extends BaseDAO {

    private static final String SQL_GET_USER = "select * from t_user where " + "user_id = ?;";

    private static final String SQL_INSERT_USER = "insert into t_user values(?, ?);";

    private static final String SQL_CLEAN_USER = "delete from t_user where 1=1;";

    @Transactional
    public User getUserById(String userId) {
    return jdbcTemplate.queryForObject(SQL_GET_USER, new Object[]{userId}, new UserRowMapper());
    }

    @Transactional
    public int insertUser(User user) {
        return jdbcTemplate.update(SQL_INSERT_USER, user.getUserId(), user.getUserName());
    }

    @Transactional
    public int cleanUser() {
        return jdbcTemplate.update(SQL_CLEAN_USER);
    }
}
```

- `UserRowMapper`

```java
package com.dao;

import java.sql.ResultSet;
import java.sql.SQLException;

import org.springframework.jdbc.core.RowMapper;

import com.data.User;

public class UserRowMapper implements RowMapper<User>{

    public User mapRow(ResultSet rs, int rowNumber) throws SQLException {
        User user = new User();
        user.setUserId(rs.getString("user_id"));
        user.setUserName(rs.getString("user_name"));
        return user;
    }
}
```

- `BaseTestCase`

```java
package com;

import org.junit.Assert;
import org.junit.runner.RunWith;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations={"/springDAO.xml"})
public class BaseTestCase extends Assert {

}
```

-  `TestUserDAO`

```java
package com.dao;

import org.junit.After;
import org.junit.Before;
import org.junit.Test;
import org.springframework.beans.factory.annotation.Autowired;

import com.BaseTestCase;
import com.data.User;

public class TestUserDAO extends BaseTestCase{

    @Before
    @After
    public void clean() {
        dao.cleanUser();
    }

    @Autowired
    private UserDAO dao;

    @Test
    public void getUserById() {
        User user = new User();
        String id = "id";
        String name = "name";
        user.setUserId(id);
        user.setUserName(name);
        assertEquals(dao.insertUser(user), 1);

        user = dao.getUserById(id);
        assertEquals(user.getUserId(), id);
        assertEquals(user.getUserName(), name);
    }
}
```
