## 个人网站搭建系列: 部署到云端
### 小插曲
本系列文章，持续更新，关注浩的个人网站<http://linhao.cfapps.io/>

### 基础准备
#### 开发环境及工具
* Jdk 1.7及以上
* Maven 3.2及以上 [下载地址](http://maven.apache.org/download.cgi)
* WampServer (使用其MySQL服务) [下载地址](http://www.wampserver.com/en/)
* cf Command Line Interface (CLI) [下载地址](https://console.run.pivotal.io/tools)

#### 参考站点
* 本文源码 <https://github.com/imlinhao/website-pws.git>
* Pivotal Web Services <http://run.pivotal.io/>
* Getting Started Deploying Spring Apps <http://docs.run.pivotal.io/buildpacks/java/gsg-spring.html>

#### 目标与方法描述
本文重点阐述如何将开发的Spring应用部署到云端，数据库的使用为MySQL，本地和云端各配置一个，通过配置文件来迅速切换本地与云端的数据库使用。

### 部署到PWS
#### PWS准备

1. **账号申请:** 登陆 PWS <http://run.pivotal.io/>，直接`SIGN UP FOR FREE`，申请账号。
1. **下载安装cf CLI:** 下载地址 <https://console.run.pivotal.io/tools>
1. **下载案例源码:** 此处我们使用[个人网站搭建系列: Spring MVC, JPA, MySQL](http://blog.csdn.net/jksl007/article/details/44600517)里使用的源码 <https://github.com/imlinhao/website-mvc-jpa-mysql.git>

#### 适配项目到PWS
此处修改基于项目`website-mvc-jpa-mysql`，修改`pom.xml`里面的项目名称配置，修改为：

	<groupId>org.test</groupId>
	<artifactId>website-pws</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

在项目根目录添加`manifest.yml`，其内容为：

	---
	applications:
	  - name: your_app_id
	    buildpack: java_buildpack
	    path: target/website-pws-0.0.1-SNAPSHOT.jar
	    services:
	      - mysql0

**注意:** applications的name，此处为`your_app_id`需要换成你自己所想要使用的名字，这个名字必须是别人没有使用过的。

修改之后，便可使用`mvn clean package`进行项目编译，编译完之后，会在`target`目录下生产一个`website-pws-0.0.1-SNAPSHOT.jar`文件。**注意：**website-mvc-jpa-mysql项目需要连接本地MySQL，所以需要在本地安装MySQL。

#### 发布到PWS

1. **登陆cf CLI:** 执行`cf login -a https://api.run.pivotal.io`，使用之前申请的PWS账号进行登陆
1. **PWS开启mysql服务:** `cf create-service cleardb spark mysql0`，该命令启动了一个mysql服务，名字为`mysql0`
1. **推送应用:** `cf push`，该命令会读取`manifest.yml`中的配置，将`website-pws-0.0.1-SNAPSHOT.jar`推送到云端执行

部署成功之后，便可以直接访问自己的应用了，如：<http://imlinhao.cfapps.io/>

### 中文、中文
如果是想在个人网站上撰写英文博客，那么以上配置便已足够。为了能让自己写中文博客，该项目还需要进一步改造。

使用命令 `cf service mysql0`可以显示之前建立的mysql0服务实例的信息，复制命令行输出的Dashboard的链接，然后在浏览器中通过该链接，查看数据库的相关信息；然后执行`cf unbind-service your_app_id mysql0`，将该服务与应用解除绑定。

删除`manifest.yml`中的`services`部分，删除之后的格式文件内容如下：

	---
	applications:
	  - name: your_app_id
	    buildpack: java_buildpack
	    path: target/website-pws-0.0.1-SNAPSHOT.jar

然后删除`src/main/resources/application.properties`，建立文件`src/main/resources/application.yml`。Spring读取配置文件时，首先会从`application.properties`中读取，如果不存在的话会从`application.yml`中读取，具体信息可以参考[Spring Boot Reference Guide](http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#howto-use-yaml-for-external-properties)。

	spring:
	  profiles:
	    active: cf
	---
	spring:
	  profiles: local
	  datasource:
	    driverClassName: com.mysql.jdbc.Driver
	    url: jdbc:mysql://localhost/your_db_name?characterEncoding=utf8
	    username: root
	    password: 123456
	---
	spring:
	  profiles: cf
	  datasource:
	    driverClassName: com.mysql.jdbc.Driver
	    url: jdbc:mysql://<hostname>:3306/<database name>?characterEncoding=utf8&autoReconnect=true
	    username: <username>
	    password: <password>
	    max-active: 2

其中，`hostname`,`database name`,`username`,`password`都可以从，原先`cf service mysql0`给出链接对应页面的Endpoint Information中获取到。

在这个`application.yml`中我配置了两个`spring profiles`：`local`和`cf`，根据自己应用的部署方式本地或者云端，可以设置`active`为`local`或者`cf`。对于云端部署，需要注意的是`url`中添加了`characterEncoding=utf8&autoReconnect=true`，是为了配置MySQL Driver使用的编码方式，以及解决云端使用MySQL的易掉线问题。而`max-active`设置为2，主要是因为，云端免费MySQL的使用，限制连接数最大为4，所以在此配置一个比4小的数字。
