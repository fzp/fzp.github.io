---
title: An Interesting Hack of Spring Cloud Config
date: 2020-11-17
tags:
  - Spring Cloud Config
---

Spring Cloud Config是一个配置中心。简单地讲， 当你用Spring Boot开发后端应用时，可以从Spring Cloud中获取配置文件，达到统一管理配置文件的目的。

Spring Cloud Config可以选取本地或者其他存储系统存放配置文件。在开发中，我们采用的是AWS S3的云存储服务。这是官方支持的存储后端。但实际上。它的行为跟使用本地方式的配置方法有不一样的地方。

<!--more-->

## S3后端存在的问题

S3后端的不支持多配置文件。

当以本地后端启动

<http://localhost:8888/config-server/common,test/prod>

当访问上面链接时，能获取

* common.yml
* test.yml
* common-prod.yml
* test-prod.yml

当以aws s3为后端时，无法获取上述文件。它将`common,test`识别一个应用的名字。而不是以逗号分割的应用列表。

另外，

<http://localhost:8888/config-server/common/prod>

当访问上面的链接时，以本地后端启动能获取到

* common.yml
* common-prod.yml

而以AWS S3为后端时，只能获取到

* common-prod.yml
  
## 解决方案

查看源码可以找到解析参数并生成具体配置文件名字的代码在`AwsS3EnvironmentRepository::findOne`

我们当然可以fork源码修改后生成自己版本的Spring cloud config。在GitHub上可以找到PR是这么干的。

https://github.com/spring-cloud/spring-cloud-config/pull/1726

不过这样一来还要单独管理我们版本的Spring cloud config， 修改使用起来也不太方便。

既然我们使用Spring Boot, 能不能使用依赖注入的方式生成一个bean取代旧的原来的AWS S3后端？


解决S3后端问题的关键代码很简单，我们新建一个类并且override findone函数

```Kotlin
override fun findOne(
    specifiedApplication: String?,
    specifiedProfiles: String?,
    specifiedLabel: String?
): Environment? {
    return specifiedApplication?.let(StringUtils::commaDelimitedListToSet)?.map {
        super.findOne(it.trim(), "$specifiedProfiles,", specifiedLabel)
    }?.reduce { acc, i -> acc.also { it.addAll(i.propertySources) } }
}
```

`specifiedApplication`参数对应例子中的`common,test`，`specifiedProfiles`对应`prod`

代码的意思是把specifiedApplication按逗号分割提取并调用父类的findone方法。

这个解决了问题1。

注意`"$specifiedProfiles,"`, specifiedProfiles后面跟了个逗号。父类的findone后将specifiedProfiles按逗号分割成多个profile。加个逗号相当于增加了一个名字为空的profile.

这个解决了问题2。

剩下的问题就是如何注入到Spring cloud config. 我原本想直接复用原有的配置项，但不幸地生成的两个bean, 并且两个bean都起作用了。

用于配置aws s3的配置项：
```
spring.cloud.config.server.awss3.region
spring.cloud.config.server.awss3.bucket
```

所以只能用新的配置项。新配置项这样:
```
spring.cloud.config.server.myawss3.region
spring.cloud.config.server.myawss3.bucket
```

要让新配置项起作用，添加：

```Kotlin
@ConfigurationProperties("spring.cloud.config.server.myawss3")
class MyAwsS3EnvironmentProperties : AwsS3EnvironmentProperties()
```

最后添加bean:

```Kotlin
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(MyAwsS3EnvironmentProperties::class)
@Profile("myawss3")
class MyAwsS3RepositoryConfiguration {

    @Bean
    fun MyAwsS3EnvironmentRepository(...):MyAwsS3EnvironmentRepository{
      ...
    }
```

注意@Profile("myawss3")，这是Spring Boot的注解，当配置文件里：

```
spring.profiles.active=hpm-awss3
```

就会自动生成bean。
