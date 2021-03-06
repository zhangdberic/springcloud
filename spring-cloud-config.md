# spring cloud config

## 服务器端(安装和配置)

### pom.xml

```xml
	<!-- 相关依赖包 -->
	<dependencies>
		<!-- spring cloud config server -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-config-server</artifactId>
		</dependency>
		<!-- spring cloud bus -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-bus-amqp</artifactId>
		</dependency>		
		<!-- spring boot security  -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-security</artifactId>
		</dependency>	
		<!-- spring boot actuator -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>			
	</dependencies>
```

spring-cloud-config-server：config server核心包。

spring-cloud-starter-bus-amqp：基于rabbitmq实现的属性刷新，服务器端publish，客户端listener，有刷新改变则publish消息。

spring-boot-starter-security：实现config server访问的安全认证，使用用户名和密码才能访问配置服务器。

spring-boot-starter-actuator：spring boot actuator，提供/actuator/xxx工具集，例如：实现属性刷新(/actuator/bus-refresh)。

如果你不需要考虑安全，不需要使用属性刷新，则只需要spring-cloud-config-server包就可以提供config server服务。

### yaml

#### bootstrap.yml

```yaml
spring:
  profiles:
     # 默认设置为开发环境
    active: dev
  application:
    name: dy-config # 应用名称
  cloud:
    config:
      server:
        encrypt:
          enabled: false  # 配置属性客户端解密
# 开发环境        
---
spring:
  profiles: dev
encrypt:
  key: 12345678 # 配置文件加密秘钥
# 测试环境        
---
spring:
  profiles: test
encrypt:
  key: 12345678 # 配置文件加密秘钥  
```

这里使用[---]分隔符加spring.profiles属性来区别不同环境的配置，其将当前环境来使用不同的配置属性。

#### application.yml

```yaml
# 默认的配置文件
server:
  port: 9000
  # tomcat 字符集
  tomcat: 
    uri-encoding: UTF-8   
  # tomcat 字符集设置
spring:        
  http: 
    encoding: 
      charset: UTF-8
      enabled: true
      force: true
# actuator       
management:
  endpoint:
    health:
      show-details: always           
  endpoints:
    web:
      exposure:
        include:
        - "*"
```

注意：actuator的配置，其配置了健康检查，并且暴露了所有的actuator端点，尽管这里暴露了所有的actuator端点，但actuator内端点的访问控制可以通过spring security来控制，例如，配置/actuator/xxx公开访问，/actuator/yyy只能登录后访问，一般都是配置一个httpBasic认证(用户名和密码)访问。

#### application-dev.yml

```yaml
# 开发环境配置 
spring:
  # 开启安全认证
  security:
    user:
      name: dy-config
      password: 12345678
  cloud:
    config:
      server:
        git:
          # Spring Cloud Config配置中心使用gitlab的话，要在仓库后面加后缀.git，而GitHub不需要
          uri: http://192.168.5.32/zhangdb/config-repo.git
          # 搜索属性文件路径,可以是正则表达式,默认只搜索根目录下的文件,配置为/**搜索所有子目录下的文件
          search-paths: /**
          # 因为github的账户和密码不能泄露,因此需要在启动脚本中加入--spring.cloud.config.server.git.username=xxxx --spring.cloud.config.server.git.password=xxxx 
          username: uuuuu
          password: xxxxxxx
        encrypt:
          enabled: false # 直接返回密文，而并非解密后的原文(需要客户端解密)
  # 属性刷新使用队列          
  rabbitmq:
    host: 192.168.5.76
    port: 5672
    username: zhangdb
    password: 12345678 
```

### @EnableConfigServer

在main方法加入@EnableConfigServer源注释，用于加载config server相关的spring bean。

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
	
	public static void main(String[] args) {
		SpringApplication.run(ConfigServerApplication.class, args);
	}
	
	@EnableWebSecurity
	class WebSecurityConfig extends WebSecurityConfigurerAdapter {
		@Override
		protected void configure(HttpSecurity httpSecurity) throws Exception {
			// Spring Security 默认开启了http页面认证登陆，需要修改为http basic模式
			httpSecurity.authorizeRequests().anyRequest().authenticated().and().httpBasic().and().csrf()
			.ignoringAntMatchers("/actuator/**","/encrypt","/decrypt");
			
		}
	}

}
```

## 客户端

### pom.xml

```xml
		<!-- spring cloud config client -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-config</artifactId>
		</dependency>
		<!-- spring boot actuator -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
		<!-- spring cloud bus -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-bus-amqp</artifactId>
		</dependency>	
```

spring-cloud-starter-config：config客户端核心依赖包；

spring-cloud-starter-bus-amqp：属性刷新使用；

spring-boot-starter-actuator：实现属性刷新(/actuator/bus-refresh)。

### yaml

#### bootstrap.yml

在/src/main/resources目录下，创建bootstrap.yml，内容包括：应用名、config客户端属性配置、config解密key属性配置，除了这些属性外，应用的其它所有属性全部都放在gitlab上集中管理。

#### gitlab创建属性文件

在gitlab上创建：

config-repo仓库

​    应用名(目录)

​       应用名.yml(文件)

​       应用名-dev.yml(文件)   // 开发环境配置文件

​       应用名-prod.yml(文件)   // 生产环境配置文件

例如：dy-eureka项目    

​	dy-eureka(目录)

​		dy-eureka.yml(文件)

​		dy-eureka-dev.yml(文件)

​		dy-eureka-prod.yml(文件)

这里使用dy-eureka为例，一般在config-repo仓库下创建dy-eureka目录，然后在下面按照文件名规则创建：

dy-eureka.yml（公共属性文件）、dy-eureka-dev.yml（开发环境属性文件）、dy-eureka-prod.yml（生产环境属性文件）；

#### 合并属性和覆盖

当你通过config server访问dy-eureka-dev.yml文件（例如：http://192.168.5.76:9000/dy-eureka-dev.yml），config server会自动会先读取dy-eureka.yml（公共属性文件），然后再读取dy-eureka-dev.yml（开发环境属性文件），然后把两个属性文件内容合并返回给请求调用者，如果两个文件有重复的属性，则使用dev文件属性覆盖公共属性。

注意：bootstrap.yml内的属性无法被applicaton.yml内的属性覆盖。

#### 变量

yaml文件内任何一个属性都可以作为变量，都可以使用${xxx}使用变量，你也可以专门定义某个属性为变量，变量的替换是由config server来完成的，例如：

```yaml
server:
  port: 8761
check:
  url: http://localhost:${server.port}
```

#### 完整例子(dy-eurkeka)

##### bootstrap.yml

在项目本地/src/main/resources目录下创建bootstrap.yml文件，如下：

```yaml
spring:
  application:
    name: dy-eureka
  profiles:
    active: dev
# 开发环境        
---
encrypt:
  key: 12345678
spring:
  profiles: dev
  cloud:
    config:
      uri: http://192.168.5.76:9000
      profile: ${spring.profiles}  # 指定从config server配置的git上拉取的文件(例如:dy-eureka-dev.yml)
      username: dy-config   # config server的basic认证的user
      password: 12345678 # config server的basic认证的password
# 测试环境
---
encrypt:
  key: 12345678
spring:
  profiles: test
  cloud:
    config:
      uri: http://192.168.5.76:9000
      profile: ${spring.profiles}  # 指定从config server配置的git上拉取的文件(例如:dy-eureka-test.yml)
      username: dy-config   # config server的basic认证的user
      password: 12345678 # config server的basic认证的password      
# 学习环境
---
encrypt:
  key: 12345678
spring:
  profiles: study
  cloud:
    config:
      uri: http://10.60.33.18:9000
      profile: ${spring.profiles}  # 指定从config server配置的git上拉取的文件(例如:dy-eureka-study.yml)
      username: dy-config   # config server的basic认证的user
      password: xxxxxx # config server的basic认证的password
# 生产环境(eureka1)
---
encrypt:
  key: 12345678
spring:
  profiles: proc_eureka1
  cloud:
    config:
      uri: http://10.60.32.198:9000
      profile: ${spring.profiles}  # 指定从config server配置的git上拉取的文件(例如:dy-eureka-proc_eureka1.yml)
      username: dy-config   # config server的basic认证的user
      password: xxxxxx # config server的basic认证的password 
# 生产环境(eureka2)
---
encrypt:
  key: 12345678
spring:
  profiles: proc_eureka2
  cloud:
    config:
      uri: http://10.60.32.198:9000
      profile: ${spring.profiles}  # 指定从config server配置的git上拉取的文件(例如:dy-eureka-proc_eureka2.yml)
      username: dy-config   # config server的basic认证的user
      password: xxxxxx # config server的basic认证的password                            
```

##### dy-eureka.yml(公共属性)

在gitlab上创建config-repo/dy-eureka/dy-eureka.yml文件

```yaml
server:
  port: 8761
  # tomcat 字符集
  tomcat: 
    uri-encoding: UTF-8  
spring:
  cloud:
    config:
      # 允许使用java -Dxxx=yyy,来覆盖远程属性，例如:java -Dserver.port=8071
      overrideSystemProperties: false
  # tomcat 字符集设置      
  http: 
    encoding: 
      charset: UTF-8
      enabled: true
      force: true
```

##### dy-eureka-dev.yml(开发环境属性)

在gitlab上创建config-repo/dy-eureka/dy-eureka-dev.yml文件

```yaml
# 开发环境配置 
spring:
  # 开启安全认证       
  security:
    user:
      name: dy-eureka
      password: 12345678
  # 和spring-cloud-starter-bus-amqp配合,用于属性刷新
  rabbitmq:
    host: 192.168.5.76
    port: 5672
    username: zhangdb
    password: 12345678
eureka:
  server:
    # 关闭自我保护模式
    enable-self-preservation: false
  instance: 
    hostname: 192.168.5.76
  client:
    # 开发环境关闭获取注册信息
    fetch-registry: false
    # 开发环境不注册到自己
    register-with-eureka: false
    service-url:
      # eureka注册中心位置
      defaultZone: http://${spring.security.user.name}:${spring.security.user.password}@${eureka.instance.hostname}:${server.port}/eureka/
```

##### 请求返回内容

请求URL：http://192.168.5.76:9000/dy-eureka-dev.yml

```yaml
eureka:
  client:
    fetch-registry: false
    register-with-eureka: false
    service-url:
      defaultZone: http://dy-eureka:12345678@192.168.5.76:8761/eureka/
  instance:
    hostname: 192.168.5.76
  server:
    enable-self-preservation: false
server:
  port: 8761
  tomcat:
    uri-encoding: UTF-8
spring:
  cloud:
    config:
      overrideSystemProperties: false
  http:
    encoding:
      charset: UTF-8
      enabled: true
      force: true
  rabbitmq:
    host: 192.168.5.76
    password: 12345678
    port: 5672
    username: zhangdb
  security:
    user:
      name: dy-eureka
      password: 12345678
```

### 发送刷新(bus-refresh)请求

例如：刷新sgw项目，发送请求到spring cloud config，并在/actuator/bus-refresh/项目名，来说刷新指定项目的配置。

```
curl -u dy-config:12345678 -X POST http://192.168.5.76:9000/actuator/bus-refresh/sgw
```



### 属性加密

#### 1.spring cloud config服务器端

**bootstrap.ym**l加入encrypt.key属性，例如：

```yaml
# 学习环境        
---
spring:
  profiles: study
encrypt:
  key: xxxxx  # 加密key
  cloud:
    config:
      server:
        encrypt:
          enabled: false # 由Config client自行解密，无法通过浏览器直接访问config server来获取解密后属性值
```

**WebSecurityConfig**配置csrf忽略/encrypt和/descrypt的

```java
	@EnableWebSecurity
	class WebSecurityConfig extends WebSecurityConfigurerAdapter {
		@Override
		protected void configure(HttpSecurity httpSecurity) throws Exception {
			// Spring Security 默认开启了http页面认证登陆，需要修改为http basic模式
			httpSecurity.authorizeRequests().anyRequest().authenticated().and().httpBasic().and().csrf()
			.ignoringAntMatchers("/actuator/**","/encrypt","/decrypt");
			
		}
	}
```

#### 2.获取加密值

发送请求计算属性加密值

```
curl -u dy-config:Study-401 http://10.60.33.18:9000/encrypt -d mysecret
```

例如：加密后的值

5754c1c69c2ee67fc31314965750c9444c4c8d53b860726affc7afa465fae8de

#### 3.spring cloud config客户端

**bootstrap.ym**l加入encrypt.key属性，并且spring cloud config的服务器和客户端的encrypt.key属性值必须相同，例如：

```yaml
# 学习环境        
---
spring:
  profiles: study
encrypt:
  key: xxxxx 
```

修改yaml属性为加密值，注意：点引号'{cipher}前缀

```yaml
spring: 
  datasource:
    password: '{cipher}f86ff2a9e2cee0cb5a5ff1c1060862dcdb64afe3287b8256cb27cd0c440887f6'
```

#### 解密

```
curl -u dy-config:123456 http://10.60.33.18:9000/decrypt -d f86ff2a9e2cee0cb5a5ff1c1060862dcdb64afe3287b8256cb27cd0c440887f6
```

