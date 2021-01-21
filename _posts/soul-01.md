---
title: soul学习——01环境搭建
date: 2021/1/14 22:00:10
categories:
- soul
tags:
- soul
- 网关
keywords:
- soul , 网关, 插件, 高性能, 可配置
---
# 前言

网关作为微服务架构中唯一的对外出入口，它的重要性不言而喻。按照经验，网关可为分为流量网关和业务网关。
<!-- more -->
流量网关的主要功能有：

1. 全局性流控
2. 日志统计
3. 防止SQL注入
4. 防止Web攻击
5. 屏蔽工具扫描
6. 黑白IP名单
7. 证书加解密处理

业务网关的主要功能有：

1. 服务级别流控
2. 服务降级与熔断
3. 路由，负载均衡与灰度策略
4. 服务过滤、聚合与发现
5. 权限验证与用户等级策略
6. 业务规则与参数校验
7. 多级缓存策略

可见网关涉及的功能非常的多，不同公司不同场景测重的方向也不同。在这种情况下大家都更倾向于找一个备选功能很齐全，可能根据自己的需求自由配置功能的网关。

有没有这样的网关呢？经过我的调研发现，国内有款开源的网关——soul满足这样的需求。soul网关有大量可插拔的功能插件来满足不同的使用场景，同时支持动态配置，使各种策略能迅速生效。最重要的是soul使用webFLux响应式的编程框架保证了服务的速度。

下面我将用一些列文章和大家一起学习soul网关，今天主要介绍测试环境的搭建。

# 环境概述

为了更好的学习soul网关，首先是深入了解soul的现有功能，那么就需要搭建测试环境。本次搭建的测试环境主要有：soul-admin、soul网关服务、eureka注册中心以及测试服务。其中soul-admin的主要功能是管理网关配置。soul网关服务是核心服务，主要实现网关的核心逻辑。eureka注册中心会记录微服务体系中启动的业务。测试服务实现了简单的GET/POST接口便于测试网关功能。

# eureka注册中心

eureka注册中心很好配置，步骤如下：

1. 创建一个maven项目，pom.xml如下配置

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
       <parent>
           <artifactId>soul-test</artifactId>
           <groupId>club.projectgaia.soul</groupId>
           <version>0.0.1-SNAPSHOT</version>
       </parent>
       <modelVersion>4.0.0</modelVersion>
   
       <artifactId>registry</artifactId>
       <dependencies>
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
           </dependency>
       </dependencies>
       <build>
           <plugins>
               <plugin>
                   <groupId>org.springframework.boot</groupId>
                   <artifactId>spring-boot-maven-plugin</artifactId>
               </plugin>
               <plugin>
                   <artifactId>maven-antrun-plugin</artifactId>
               </plugin>
           </plugins>
       </build>
   </project>
   ```

2. 在resources中添加applicatin.yml，并作如下配置

   ```yaml
   spring:
     application:
       name: registry
   server:
     port: 8079
   
   eureka:
     instance:
       prefer-ip-address: true
     client:
       register-with-eureka: false
       fetch-registry: false
       service-url:
         defaultZone: http://127.0.0.1:8079/eureka/
     server:
       enable-self-preservation: false
       eviction-interval-timer-in-ms: 5000
   
   ```

3. 编写启动类RegistryApplication.java，内容如下

   ```java
   @SpringBootApplication
   @EnableEurekaServer
   public class RegistryApplication {
       
       public static void main(String[] args) {
           SpringApplication.run(RegistryApplication.class, args);
       }
   }
   ```

整个项目结构如下图

![图1](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/soul-01-01.png)

# soul-admin

Soul-admin是soul的管理后台，准备直接使用soul源码启动。主要步骤如下：

1. fork soul的自主分支

2. soul项目拉到本地

3. 设置upstream分支为soul的官方 主分支

4. 切一个viewCode分支防止错误提交

5. 修改soul-admin/resources/application-local.yml文件，主要是配置正确的可访问的mysql地址

6. 启动，访问http://localhost:9095  使用admin 123456 就可以登录soul的管理后台了

   ![图1](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/soul-01-02.png)

   可以看出来soul的管理后台是做了国际化的，同时左侧菜单有"插件列表"和"系统设置"2个栏目，从插件列表可以发现，soul已经为我们实现了许多插件。

# soul网关服务

和管理后台服务一样，网关服务也从源码中直接启动。

1. 修改soul-bootstrap项目的pom文件

   1. 由于我们使用的是springCloud体系，所以注释掉 http proxy相关内容，打开springCloud plugin相关内容，如下	

      ```xml
              <!--if you use http proxy start this-->
              <!-- ... -->
              <!--if you use http proxy end this-->
      
      				<!--soul alibaba dubbo plugin start-->
              <!-- ... -->
              <!--soul alibaba dubbo plugin end-->	
      
      				<!--soul  apache dubbo plugin start-->
              <!-- ... -->
              <!--soul alibaba dubbo plugin end-->	
      
              <!--soul springCloud plugin start-->
              <dependency>
                  <groupId>org.dromara</groupId>
                  <artifactId>soul-spring-boot-starter-plugin-springcloud</artifactId>
                  <version>${project.version}</version>
              </dependency>
      
              <dependency>
                  <groupId>org.springframework.cloud</groupId>
                  <artifactId>spring-cloud-commons</artifactId>
                  <version>2.2.0.RELEASE</version>
              </dependency>
              <dependency>
                  <groupId>org.springframework.cloud</groupId>
                  <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
                  <version>2.2.0.RELEASE</version>
              </dependency>
              <!--soul springCloud plugin start end-->
      ```

      我们可以发现，除了支持spring-cloud体系，soul还支持无框架，apache-dubbo体系，alibaba-dubbo体系。

   2. 由于我们注册中心使用的是eureka，所以注释掉其他注册中心，打开eureka相关配置，如下

      ```xml
              <!-- Dubbo zookeeper registry dependency start -->
              <!-- ... -->
              <!-- Dubbo zookeeper registry dependency end -->
      
      				<!-- springCloud if you config register center is nacos please dependency this-->
              <!-- ... -->
      
              <!-- springCloud if you config register center is eureka please dependency end-->
              <dependency>
                  <groupId>org.springframework.cloud</groupId>
                  <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
                  <version>2.2.0.RELEASE</version>
              </dependency>
      ```

      从配置中我们可以看出来，使用spring-cloud体系还能使用nacos作为配置中心，使用dubbo体系可以用zookeeper作为配置中心。

   3. 修改application-local.yml相关配置，主要是打开eureka的配置如下：

      ```yml
      # Licensed to the Apache Software Foundation (ASF) under one or more
      # contributor license agreements.  See the NOTICE file distributed with
      # this work for additional information regarding copyright ownership.
      # The ASF licenses this file to You under the Apache License, Version 2.0
      # (the "License"); you may not use this file except in compliance with
      # the License.  You may obtain a copy of the License at
      #
      #     http://www.apache.org/licenses/LICENSE-2.0
      #
      # Unless required by applicable law or agreed to in writing, software
      # distributed under the License is distributed on an "AS IS" BASIS,
      # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
      # See the License for the specific language governing permissions and
      # limitations under the License.
      
      server:
        port: 9195
        address: 0.0.0.0
      
      spring:
         main:
           allow-bean-definition-overriding: true
         application:
          name: soul-bootstrap
      #   cloud:
      #    nacos:
      #       discovery:
      #          server-addr: 127.0.0.1:8848
      management:
        health:
          defaults:
            enabled: false
      soul :
          file:
            enabled: true
          corss:
            enabled: true
          dubbo :
            parameter: multi
          sync:
              websocket :
                   urls: ws://localhost:9095/websocket
      #        zookeeper:
      #             url: localhost:2181
      #             sessionTimeout: 5000
      #             connectionTimeout: 2000
      #        http:
      #             url : http://localhost:9095
      #        nacos:
      #              url: localhost:8848
      #              namespace: 1c10d748-af86-43b9-8265-75f487d20c6c
      #              acm:
      #                enabled: false
      #                endpoint: acm.aliyun.com
      #                namespace:
      #                accessKey:
      #                secretKey:
      eureka:
        client:
          service-url:
            defaultZone: http://127.0.0.1:8079/eureka
          instance:
            prefer-ip-address: true
      logging:
          level:
              root: info
              org.springframework.boot: info
              org.apache.ibatis: info
              org.dromara.soul.bonuspoint: info
              org.dromara.soul.lottery: info
              org.dromara.soul: info
      ```

3. 启动项目，在soul和soul-admin都可以看见相关配置

# 自定义服务

编写一个spring-web项目用于测试，步骤如下：

1. 创建spring-web项目

2. pom文件中引入eureka-clien和soul-spring-boot-client，如下

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
       <parent>
           <artifactId>soul-test</artifactId>
           <groupId>club.projectgaia.soul</groupId>
           <version>0.0.1-SNAPSHOT</version>
       </parent>
       <modelVersion>4.0.0</modelVersion>
   
       <artifactId>demo</artifactId>
   
       <dependencies>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-web</artifactId>
           </dependency>
   
   
           <!--soul springCloud plugin end-->
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
           </dependency>
   
           <!-- 引入 Soul 针对 Spring Cloud 的集成的依赖 -->
           <dependency>
               <groupId>org.dromara</groupId>
               <artifactId>soul-spring-boot-starter-client-springcloud</artifactId>
               <version>2.2.0</version>
           </dependency>
   
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-configuration-processor</artifactId>
               <optional>true</optional>
           </dependency>
       </dependencies>
   
   </project>
   ```

3. 修改application.yml文件，增加eureka配置以及soul配置，如下

   ```yml
   server:
     port: 8082
   eureka:
     client:
       service-url:
         defaultZone: http://127.0.0.1:8079/eureka
   spring:
     application:
       name: test
   
   soul:
     # Soul 针对 SpringMVC 的配置项，对应 SoulSpringCloudConfig 配置类
     springcloud:
       # Soul Admin 地址
       admin-url: http://127.0.0.1:9095
       context-path: /test-api 
       # 设置在 Soul 网关的路由前缀，例如说 /order、/product 等等。
       # 后续，网关会根据该 context-path 来进行路由
   ```

4. 编写简单业务逻辑，核心要点是要在controller层的方法上添加@SoulSpringCloudClient注解，代码如下

   ```java
   /**
    * @author Phoenix Luo
    * @version 2020/11/28
    **/
   @RestController
   @RequestMapping("/demo")
   @Slf4j
   public class DemoController {
       
       @GetMapping("/test")
       @SoulSpringCloudClient(path = "/demo/test")
       public String hello() {
           
           return "{\"name\":\"hello\"}";
       }
   }
   ```

   可以发现我们的服务实现了 GET /demo/test 返回{"name":"hello"}的功能。

5. 启动服务，会看见日志` http client register success :{} {"appName":"test","context":"/test-api","path":"/test-api/demo/test","pathDesc":"","rpcType":"springCloud","ruleName":"/test-api/demo/test","enabled":true}`
6. 登录soul-admin
   1. 在  系统管理----> 元数据管理 中我们会发现我们的服务已经自动注册到了soul中
   2. 在  系统管理----> 插件管理中 我们设置spring-cloud插件为开启状态
   3. 在 插件管理 ----> spring cloud 中我们会发现，已经帮我们配置好了一个选择器以及一个选择器规则
7. 访问 http://localhost:9195/test-api/demo/test  既可以通过网关访问到我们实际的自定义业务服务。