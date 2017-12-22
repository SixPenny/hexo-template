---
title: spring boot 配置https
date: 2017-12-21 18:27:44
layout: tag
tags: [spring boot, 模块]
---


1：使用keytool生成 keystore
  keytool -keystore mykey.jks -genkey -alias tomcat -keyalg RSA
  
  记住密钥库口令和密钥口令，后面配置需要使用，这里我直接使用Tomcat
  
2:在application.properties 或 application.yml中配置属性
```yaml
server:
  port: 8443
  ssl:
    key-store: classpath:mykeys.jks
    key-store-password: Tomcat
    key-password: Tomcat
```

3:启动即可看到访问地址变成了https://localhost:8443