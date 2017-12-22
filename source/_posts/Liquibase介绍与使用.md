---
title: Liquibase介绍与使用
date: 2017-12-21 18:27:44
layout: tag
tags: Liquibase
---


## Liquibase 简介

liquibase在其[官网首页](http://www.liquibase.org/ "liquibase官网")中有一个很明确的定位，
那就是Source Control For Your Database，Liquibase记录你的数据库变更，
可以在你你迁移时迅速的生成一个与原数据库一致的库出来。还可以与其余数据库做diff，支持多人开发等功能。

<!--more-->

## Liquibase 使用

### Liquibase Maven 配置

Liquibase 支持命令行，maven，ant，spring等方式，我平常使用maven，因此只说一下maven需要的配置。

dependency：
```maven
    <dependency>
        <groupId>org.liquibase</groupId>
        <artifactId>liquibase-core</artifactId>
        <version>3.5.3</version>
    </dependency>
````

提供了maven plugin，可以使用各种构建来使用Liquibase，非常方便

```maven
<plugin>
     <groupId>org.liquibase</groupId>
     <artifactId>liquibase-maven-plugin</artifactId>
     <version>3.5.3</version>
     <dependencies>
         <dependency>
             <groupId>javax.validation</groupId>
             <artifactId>validation-api</artifactId>
             <version>1.1.0.Final</version>
         </dependency>
         <dependency>
             <groupId>org.javassist</groupId>
             <artifactId>javassist</artifactId>
             <version>3.21.0-GA</version>
         </dependency>
         <dependency>
             <groupId>org.liquibase.ext</groupId>
             <artifactId>liquibase-hibernate5</artifactId>
             <version>3.6</version>
         </dependency>
         <dependency>
             <groupId>org.springframework.boot</groupId>
             <artifactId>spring-boot-starter-data-jpa</artifactId>
             <version>1.5.2.RELEASE</version>
         </dependency>
     </dependencies>
     <configuration>
         <changeLogFile>src/main/resources/config/liquibase/master.xml</changeLogFile>
         <diffChangeLogFile>src/main/resources/config/liquibase/changelog/${maven.build.timestamp}_changelog.xml</diffChangeLogFile>
         <driver>com.mysql.jdbc.Driver</driver>
         <url>jdbc:mysql://localhost:3306/test1</url>
         <defaultSchemaName>test1</defaultSchemaName>
         <username>test</username>
         <password>test</password>
         <verbose>true</verbose>
         <logging>debug</logging>
     </configuration>
 </plugin>
```

java代码中需要配置Liquibase的bean，以下是基于Spring boot配置
```java
public SpringLiquibase liquibase(@Qualifier("taskExecutor") TaskExecutor taskExecutor,
            DataSource dataSource, LiquibaseProperties liquibaseProperties) {

        // Use liquibase.integration.spring.SpringLiquibase if you don't want Liquibase to start asynchronously
        SpringLiquibase liquibase = new AsyncSpringLiquibase(taskExecutor, env);
        liquibase.setDataSource(dataSource);
        liquibase.setChangeLog("classpath:config/liquibase/master.xml");
        liquibase.setContexts(liquibaseProperties.getContexts());
        liquibase.setDefaultSchema(liquibaseProperties.getDefaultSchema());
        liquibase.setDropFirst(liquibaseProperties.isDropFirst());
        if (env.acceptsProfiles(SPRING_PROFILE_NO_LIQUIBASE)) {
            liquibase.setShouldRun(false);
        } else {
            liquibase.setShouldRun(liquibaseProperties.isEnabled());
            log.debug("Configuring Liquibase");
        }
        return liquibase;
    }
```
这里添加了根据profile决定是否启用Liquibase的判断，也可以在Liquibase的xml配置中使用preCondition来决定Liquibase是否启用

### Liquibase xml配置元素

#### databaseChangeLog
databaseChangeLog 是配置的顶级元素，跟Spring的beans是一样的，里面可以包含其他的元素
可以有property,preConditions,changeSet,include等元素，这里主要介绍平常使用比较多的这四种元素加loadData元素

#### property
property可以用来声明变量，也可以根据db来决定变量的值是如何绑定的。
在后面的使用中用`${name}`来使用
```xml
<property name="now" value="now()" dbms="h2"></property>
<property name="now" value="now()" dbms="mysql"></property>
<property name="autoIncrement" value="true"></property>
```
上面声明了与数据库相关的now时间获取方法，还声明了一个平常值。

#### preConditions
只有满足了preConditions中的先决条件，Liquibase才会运行相应的配置
譬如我们只想在h2中使用，可以这样配置：
```xml
<databaseChangeLog>
    <preConditions>
      <dbms type="h2"></dbms>
    </preConditions>
</databaseChangeLog>
```
preConditions还有其他的很多选项可以使用，如`<runningAs>` `<columnExists>` `<tableExists>`等，
有兴趣的可以自行查看[官网preconditions介绍](http://www.liquibase.org/documentation/preconditions.html)来获取更全的内容
preConditions也可以在changeSet中使用，来决定一个changeSet是否运行，会在下面给出一个例子

#### changeSet
changeSet意思是更改集，也就是我们数据库变更的主要部分，在这里面可以创建表，添加表行，删除表行，删除某个表，添加索引、主键等等操作,一个xml里面可以包含有多个changeSet，一个changeSet里可以包含多个操作

Liquibase会在数据库中自动创建DATABASECHANGELOG，DATABASECHANGELOGLOCK两个表，其中DATABASECHANGELOG里面每一行代表的就是一个changeSet，里面的元素记录了changeSet的状态，决定后续的执行

创建表：
```xml
<changeSet author="liufengquan" id="1498016931954-6">
    <createTable tableName="testTable">
        <column autoIncrement="true" name="id" type="BIGINT">
            <constraints primaryKey="true"/>
        </column>
        <column name="name" type="VARCHAR(255)">
            <constraints nullable="false"/>
        </column>
        <column name="length" type="INT"/>
    </createTable>
</changeSet>
```
添加表列：
```xml
<changeSet id="11111" author="liufengquan">
    <addColumn tableName="testTable">
        <column name="name" type="varchar(100)"></column>
    </addColumn>
</changeSet>
```
删除表列
```xml
<changeSet id="2222" author="liufengquan">
    <dropColumn tableName="testTable" columnName="name"></dropColumn>
</changeSet>
```
添加索引，添加主键
```xml
<changeSet id="3333" author="liufengquan">
    <addPrimaryKey columnNames=" name" tableName="testTable"/>

    <createIndex indexName="idx_testTable_name"
                 tableName="testTable"
                 unique="false">
        <column name="name" type="varchar(100)"/>
    </createIndex>
</changeSet>
```
id并没有要求必须是唯一的，在DATABASECHANGELOG表中，id,author,filepath（changeSet所在文件路径）三者决定了一个changeSet，id也未要求必须是数字，只要符合自己的习惯就可以，不过在自己书写changeSet(即author为同一人)时，自己定义的id必须不同，不然会出问题。
如果changeSet的执行顺序有要求，可以在上面使用`runOrder`来指定
还有`runAlways` `runOnChange`等决定changeSet的运行时机

在changeSet中使用preConditions决定是否执行
下面是一个官网上的例子，只有当表中数据为空时才把table drop掉
```xml
<changeSet id="1" author="bob">
    <preConditions onFail="WARN">
        <sqlCheck expectedResult="0">select count(*) from oldtable</sqlCheck>
    </preConditions>
    <comment>Comments should go after preCondition. If they are before then liquibase usually gives error.</comment>
    <dropTable tableName="oldtable"/>
</changeSet>
```
更加详细的标签说明请参考[官网changeSet说明](http://www.liquibase.org/documentation/changeset.html)

#### include
所有的变更都写在一个文件里面使得文件后面会不可维护，可以按业务维护不同的database change log file，然后在一个主xml中引用所有的
```xml
<?xml version="1.0" encoding="utf-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.4.xsd">

    <preConditions>
        <dbms type="h2"/>
    </preConditions>
    <include file="classpath:config/liquibase/changelog/20170621.xml" relativeToChangelogFile="false"/>
</databaseChangeLog>
```
`relativeToChangelogFile`表示引入的文件路径是否是相对于主文件路径，默认为false，可以不写

#### loadData
将数据写入到表中，文件可以使用csv格式,第一行是列名以分号分割，后续每行代表数据库中的一行数据，也以分号分割即可
```xml
<loadData encoding="UTF-8"
      file="config/liquibase/testTable.csv"
      separator=";"
      tableName="testTable"/>
```

# h2数据库
## h2数据库简介
h2是一个嵌入式数据库，也就是不用单独安装服务端和客户端，并且h2可以与其他主流的数据库兼容，支持MySQL，Oracle的语法。h2支持内存数据库，特别适合单元测试这种场景，当然h2不限于此，也可以持久化到硬盘上，不过大家在正式上使用的毕竟还是少。
## h2数据库说明
h2数据库的语法之类的大家可以自行找网上资料或者去官网学习，此处不再详述。
配置就是在pom中引入h2的依赖,然后在spring的配置中换成h2的connector就可以了
```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
```
url：
```url
jdbc:h2:mem:Test;DB_CLOSE_DELAY=-1;MODE=MySQL
```
# 其他方法
spring boot在application.yml中的提供了初始化schema和data的配置，可以使用spring.datasource.schema和spring.datasource.data分别指定建表脚本和初始化数据脚本，不过我使用了一下，直接用Navicat MySQL导出数据库脚本，在建表时报错，应该是h2对某些MySQL的语法写法不支持，这样的话去找就比较麻烦，而且后续维护这个脚本也会越来越困难，因此并没有采用这种办法。不过如果项目比较小，又图前期省事的话，这个方案还是值得使用的。


# 总结
使用Liquibase来管理数据库schema，使用h2来随时在内存中创建数据库，以后基本可以不用担心单元测试中的数据问题了，数据库的变更也变得有迹可循，感谢贡献出这些工具的人。