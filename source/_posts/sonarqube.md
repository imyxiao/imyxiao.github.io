---
title: SonarQube的安装与使用
author: 肖哥
avatar: /yxiao.jpg
authorLink: https://blog.imyxiao.com
authorAbout: https://blog.imyxiao.com/about
authorDesc: Java Developer
date: 2017-09-27 17:50:54
categories: docker
tags:
  - sonarqube
  - sonar
  - docker
description: Sonar是一个用于代码质量管理的开源平台，用于管理源代码的质量，帮助我们从源码中找出潜在的bug，未使用的代码，复杂的表达式，未覆盖的代码，糟糕的代码，重复的代码等等,通过插件形式，可以支持包括java,C#,C/C++,PL/SQL,Cobol,JavaScrip,Groovy等等二十几种编程语言的代码质量管理与检测
---

Sonar是一个用于代码质量管理的开源平台，用于管理源代码的质量，帮助我们从源码中找出潜在的bug，未使用的代码，复杂的表达式，未覆盖的代码，糟糕的代码，重复的代码等等
通过插件形式，可以支持包括java,C#,C/C++,PL/SQL,Cobol,JavaScrip,Groovy等等二十几种编程语言的代码质量管理与检测

更多介绍请看官方网站[SonarQube](https://www.sonarqube.org/),官方文档:[SONAR Documentation](https://docs.sonarqube.org/display/SONAR/Documentation),
## SonarQube安装
推荐使用`docker`安装,官方`docker`地址:[SonarQube](https://hub.docker.com/_/sonarqube/), 简单的运行一个`SonarQube`服务:

```bash
docker run -d --name sonarqube -p 9000:9000 -p 9092:9092 sonarqube
```

打开`http://localhost:9000/`,默认帐号和密码都为`admin`
### 数据库配置
默认情况使用的一个内嵌的`H2`数据库,肯定不适合生产情境,数据库配置的三个变量
- `SONARQUBE_JDBC_USERNAME`
- `SONARQUBE_JDBC_PASSWORD`
- `SONARQUBE_JDBC_URL`

官方示例(需要安装一个`postgresql`):
```bash
docker run -d --name sonarqube \
    -p 9000:9000 -p 9092:9092 \
    -e SONARQUBE_JDBC_USERNAME=sonar \
    -e SONARQUBE_JDBC_PASSWORD=sonar \
    -e SONARQUBE_JDBC_URL=jdbc:postgresql://localhost/sonar \
    sonarqube
```
由于我已经有一个`mysql`容器:
```bash
docker run --name wlcx_mysql -p 3306:3306 -v /srv/mysql:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=password -d mysql
```
所以使用链接容器的方式使用数据库,容器名为`wlcx_mysql`

```bash
docker run -d --link wlcx_mysql:wlcx_mysql --name sonarqube \
    -p 9000:9000 -p 9092:9092 \
    -e SONARQUBE_JDBC_USERNAME=root \
    -e SONARQUBE_JDBC_PASSWORD=password \
    -e "SONARQUBE_JDBC_URL=jdbc:mysql://wlcx_mysql:3306/sonar?useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&useSSL=false" \
    sonarqube
```
也可以使用`docker-compose`的方式,更加的方便,请参考:[docker sonarqube recipes](https://github.com/SonarSource/docker-sonarqube/blob/master/recipes.md)

## SonarQube使用

使用默认的帐号登录之后,可以:
- 生成一个代替帐号的`token`
- 修改一个`admin`的密码
- 可以在`Administration`=>`System`=>`Update Center`,安装中文插件和其它要分析的语言的插件

拿到了`token`后,介绍如何用`sonar`分析`Maven`与`Gradle`项目
### Maven项目
```bash
mvn clean verify sonar:sonar \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.login=token
```
运行这个命令即可,为了固定`sonar`的mvn插件版本,可以配置:
```xml
<build>
  <pluginManagement>
    <plugins>
      <plugin>
        <groupId>org.sonarsource.scanner.maven</groupId>
        <artifactId>sonar-maven-plugin</artifactId>
        <version>3.3.0.603</version>
      </plugin>
    </plugins>
  </pluginManagement>
</build>
```

更多详细配置:[Analyzing with SonarQube Scanner for Maven](https://docs.sonarqube.org/display/SCAN/Analyzing+with+SonarQube+Scanner+for+Maven)

### Gradle项目

配置插件：
```groovy
buildscript {
  repositories {
    maven {
      url "https://plugins.gradle.org/m2/"
    }
  }
  dependencies {
    classpath "org.sonarsource.scanner.gradle:sonarqube-gradle-plugin:2.5"
  }
}

apply plugin: "org.sonarqube"
```
对于`Gradle 2.1+` ，可以这样配置：
```
plugins {
  id "org.sonarqube" version "2.5"
}
```

运行命令：
```bash
gradle sonarqube \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.login=token
```
或将`sonar`地址与`token`，配置到项目下的`gradle.properties`或`~/.gradle/gradle.properties`当中
```properties
systemProp.sonar.host.url=http://localhost:9000

systemProp.sonar.login=token
```
然后执行`gradle sonarqube`即可,更多详细配置请看:[Analyzing with SonarQube Scanner for Gradle](https://docs.sonarqube.org/display/SCAN/Analyzing+with+SonarQube+Scanner+for+Gradle)
