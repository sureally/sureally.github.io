---
layout: article
title: "mybatis druid demo"
date: 2019-07-20 14:26:08 +0800
categories: demo java
---

# 前言
java/mysql/mybatis/druid的一个使用demo项目。

- mybatis 持久层框架

- druid 数据库连接池

- mysql 数据库

# 项目demo

## 项目结构

![image]({{site.url}}/assets/images/2019/07/mybatis-druid-demo-structure.png)

## 文件详情
### 配置文件
#### mybatis-config.xml

```XML
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>

  <!--  引入配置文件 -->
  <properties resource="mysql.properties"/>

  <!--  注意顺序，settings environments 等这些的位置是存在先后顺序的 -->
  <settings>
    <setting name="callSettersOnNulls" value="true"/>
    <setting name="cacheEnabled" value="true"/>
    <setting name="lazyLoadingEnabled" value="true"/>
    <setting name="aggressiveLazyLoading" value="true"/>
    <setting name="multipleResultSetsEnabled" value="true"/>
    <setting name="useColumnLabel" value="true"/>
    <setting name="useGeneratedKeys" value="false"/>
    <setting name="autoMappingBehavior" value="PARTIAL"/>
    <setting name="defaultExecutorType" value="SIMPLE"/>
    <setting name="mapUnderscoreToCamelCase" value="true"/>
    <setting name="localCacheScope" value="SESSION"/>
    <setting name="jdbcTypeForNull" value="NULL"/>
  </settings>

  <typeAliases>
    <!--    自定的数据库连接池配置 -->
    <typeAlias type="com.demo.config.DruidDataSourceFactory" alias="DRUID"/>
  </typeAliases>

  <environments default="mysql-druid">
    <environment id="mysql-druid">

      <!-- 配置数据库连接信息 -->
      <transactionManager type="JDBC"/>
      <!-- value属性值引用db.properties配置文件中配置的值 -->
      <dataSource type="DRUID">
        <property name="driver" value="${driver}"/>
        <property name="url" value="${url}"/>
        <property name="username" value="${username}"/>
        <property name="password" value="${password}"/>

        <!-- 配置初始化大小、最小、最大 -->
        <property name="initialSize" value="${druid.initialSize}"/>
        <property name="minIdle" value="${druid.minIdle}"/>
        <property name="maxActive" value="${druid.maxActive}"/>
        <!-- 配置从连接池获取连接等待超时的时间 -->
        <property name="maxWait" value="${druid.maxWait}"/>

        <property name="testOnBorrow" value="${druid.testOnBorrow}"/>
        <!-- 设置往连接池归还连接时是否检查连接有效性，true时，每次都检查;false时，不检查 -->
        <property name="testOnReturn" value="${druid.testOnReturn}"/>
        <!-- 设置从连接池获取连接时是否检查连接有效性，
          true时，如果连接空闲时间超过minEvictableIdleTimeMillis进行检查，否则不检查;
          false时，不检查-->
        <property name="testWhileIdle" value="${druid.testWhileIdle}"/>

        <!-- 配置间隔多久启动一次DestroyThread，对连接池内的连接才进行一次检测，单位是毫秒。
            检测时:1.如果连接空闲并且超过minIdle以外的连接，如果空闲时间超过minEvictableIdleTimeMillis设置的值则直接物理关闭。2.在minIdle以内的不处理。-->
        <property name="timeBetweenEvictionRunsMillis" value="${druid.timeBetweenEvictionRunsMillis}"/>

        <!-- 检验连接是否有效的查询语句。
          如果数据库Driver支持ping()方法，则优先使用ping()方法进行检查，否则使用validationQuery查询进行检查。
          (Oracle jdbc Driver目前不支持ping方法)-->
        <property name="validationQuery" value="${druid.validationQuery }"/>

        <!-- 打开后，增强timeBetweenEvictionRunsMillis的周期性连接检查，
          minIdle内的空闲连接，每次检查强制验证连接有效性.
          参考：https://github.com/alibaba/druid/wiki/KeepAlive_cn-->
        <property name="keepAlive" value="${druid.keepAlive}"/>

      </dataSource>
    </environment>
  </environments>

  <!-- mybatis的mapper文件，每个xml配置文件对应一个接口 -->
  <mappers>
    <!--   指定mapper路径 -->
    <package name="com.demo.dao"/>
  </mappers>
</configuration>
```
#### mysql.properties

**注意不要在该文件的配置项中添加引号 "**

```properties
driver = com.mysql.cj.jdbc.Driver
url = jdbc:mysql://localhost:3306/test?useUnicode=true&characterEncoding=utf8&useSSL=false&serverTimezone=Asia/Shanghai&jdbcCompliantTruncation=false
username = root
password = 55555555

druid.initialSize = 1
druid.minIdle = 1
druid.maxActive = 5
druid.maxWait = 60000

druid.testOnBorrow = false
druid.testOnReturn = false
druid.testWhileIdle = true

druid.timeBetweenEvictionRunsMillis = 600000
druid.validationQuery = SELECT 1 from dual
druid.keepAlive = true
```

#### DruidDataSourceFactory.java

```java
package com.demo.config;

import javax.sql.DataSource;
import java.sql.SQLException;
import java.util.Properties;

import com.alibaba.druid.pool.DruidDataSource;
import org.apache.ibatis.datasource.DataSourceFactory;


/**
 * @ClassName com.demo.config.DruidDataSourceFactory
 * @Desciption 数据库连接池自定义属性
 * @Author Shu WJ
 * @DateTime 2019-07-20 00:01
 * @Version 1.0
 **/
public class DruidDataSourceFactory implements DataSourceFactory {

  private Properties properties;

  @Override
  public void setProperties(Properties properties) {
    this.properties = properties;
  }

  @Override
  public DataSource getDataSource() {
    DruidDataSource dataSource = new DruidDataSource();
    dataSource.setDriverClassName(this.properties.getProperty("driver"));
    dataSource.setUrl(this.properties.getProperty("url"));
    dataSource.setUsername(this.properties.getProperty("username"));
    dataSource.setPassword(this.properties.getProperty("password"));
    dataSource.setInitialSize(Integer.valueOf(this.properties.getProperty("initialSize")));
    dataSource.setMinIdle(Integer.valueOf(this.properties.getProperty("minIdle")));
    dataSource.setMaxActive(Integer.valueOf(this.properties.getProperty("maxActive")));
    dataSource.setMaxWait(Integer.valueOf(this.properties.getProperty("maxWait")));
    dataSource.setTestOnBorrow(Boolean.valueOf(this.properties.getProperty("testOnBorrow")));
    dataSource.setTestOnReturn(Boolean.valueOf(this.properties.getProperty("testOnReturn")));
    dataSource.setTestWhileIdle(Boolean.valueOf(this.properties.getProperty("testWhileIdle")));

    dataSource.setTimeBetweenEvictionRunsMillis(Long.valueOf(this.properties.getProperty("timeBetweenEvictionRunsMillis")));
    dataSource.setValidationQuery(this.properties.getProperty("validationQuery"));
    dataSource.setKeepAlive(Boolean.valueOf(this.properties.getProperty("keepAlive")));
    try {
      dataSource.init();
    } catch (SQLException e) {
      e.printStackTrace();
    }

    return dataSource;
  }
}
```

#### MyBatisUtil.java
```java
package com.demo.util;

import java.io.IOException;
import java.io.InputStream;

import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;

/**
 * @ClassName com.demo.util.MyBatisUtil
 * @Desciption
 * @Author Shu WJ
 * @DateTime 2019-07-20 00:26
 * @Version 1.0
 **/
public class MyBatisUtil {
  private static final String CONFIG_FILE = "mybatis-config.xml";

  public static SqlSessionFactory sqlSessionFactory;

  static {
    try (InputStream inputStream = Resources.getResourceAsStream(CONFIG_FILE)) {
      sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
    } catch (IOException e) {
      e.printStackTrace();
    }
  }

  // 推荐用法
  // try (SqlSession sqlSession = sqlSessionFactory.openSession()) {}
}
```


### 使用demo
#### User.java

```java
package com.demo.model;

import java.util.Date;
import lombok.Data;

/**
 * @ClassName com.demo.model.User
 * @Desciption 实体类
 * @Author Shu WJ
 * @DateTime 2019-07-20 00:08
 * @Version 1.0
 **/
@Data
public class User {
  private Long id;
  private String name;
  private String password;
  private Date createdTime;
  private Date updatedTime;
}

```

#### UserDao.java

```java
package com.demo.dao;

import com.demo.model.User;
import org.apache.ibatis.annotations.Insert;
import org.apache.ibatis.annotations.Options;
import org.apache.ibatis.annotations.Select;
import org.apache.ibatis.annotations.Update;

/**
 * @InterfaceName com.demo.dao.UserDao
 * @Desciption dao 层数据库交互
 * @Author Shu WJ
 * @DateTime 2019-07-20 00:10
 * @Version 1.0
 **/
public interface UserDao {

  /**
   * 插入一条语句, 插入单一记录，返回自增主键
   * @param user
   * @return
   */
  @Options(useGeneratedKeys = true, keyProperty = "id", keyColumn = "id")
  @Insert({" insert into user(name, password) values(#{name}, #{password}) "})
  int insert(User user);

  /**
   * 根据Id查询，据某书说最好不要用*通配即可。
   * @param id
   * @return
   */
  @Select({" select * from user where id=#{id}"})
  User selectById(Long id);


  /** 部分更新，这种其实写xml比较舒服 */
  @Update({
    "<script>",
    " update user",
    " <trim prefix='SET' suffixOverrides=','> ",
    "   <if test='name != null'>name=#{name},</if>",
    "   <if test='password != null'>`password`=#{password},</if>",
    " </trim>",
    " where id=#{id} ",
    "</script>"
  })
  int updateSelection(User user);
}
```

# 链接

-[github](https://github.com/sureally/demo/tree/99d0a22e5c1dd45defaa6215a2b98fe7ae59fa7b/mybatis-druid-demo)

# 参考文档
-
-
