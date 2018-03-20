---
title: mybatis+springboot配置多数据源
date: 2018-03-20 22:48:22
categories:
- mybatis
tags:
- mybatis
- springboot
---

---
# 遇到的问题
---

##  无法使用类型别名的包扫描

按照文档中的说明，只需要在sprinboot的配置文件中添加：`mybatis.type-aliases-package=com.example.domain.model`即可，但是自己的项目中的配置就不是不起作用。

### 问题分析

官方文档中说的配置是针对mybatis自己的SqlSessionFactory初始化的时候自动根据spring配置文件中的配置去进行包扫描，但是由于自己使用多数据源，没有使用mybatis自带的数据源配置，自己的数据源以及SqlSessionFactory初始化的时候，并没有指定mybatis的配置文件，所以肯定是没有效果的。

- 数据源配置

```
@Configuration
@MapperScan(basePackages = DataSourceCounterConfig.PACKAGE, sqlSessionFactoryRef = "counterSqlSessionFactory")
public class DataSourceCounterConfig {
	private static final Logger MLOG = LoggerFactory.getLogger(DataSourceCounterConfig.class);
	// 精确到 counter 目录，以便跟其他数据源隔离
	protected static final String PACKAGE = "com.talkingdata.campaign.channel.dao.counter";
	private static final String MAPPER_LOCATION = "classpath:mapper/counter/*.xml";
	private static final String MYBATIS_CONFIG_LOCATION = "classpath:mybatis-config.xml";

	@Bean(name = "counterDataSource")
    @Primary
    public DataSource counterDataSource() {
        ...
    }
 
    @Bean(name = "counterTransactionManager")
    @Primary
    public DataSourceTransactionManager counterTransactionManager() {
        return new DataSourceTransactionManager(counterDataSource());
    }
 
    @Bean(name = "counterSqlSessionFactory")
    @Primary
    public SqlSessionFactory counterSqlSessionFactory(@Qualifier("counterDataSource") DataSource counterDataSource)
            throws Exception {
        final SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();
        sessionFactory.setDataSource(counterDataSource);
        sessionFactory.setMapperLocations(new PathMatchingResourcePatternResolver()
                .getResources(DataSourceCounterConfig.MAPPER_LOCATION));
		/** 下面一句代码的作用是给mybatis的SqlSessionFactoryBean设置配置文件，在配置文件中配置包扫描即可。 */
		sessionFactory.setConfigLocation(new PathMatchingResourcePatternResolver()
				.getResource(DataSourceCounterConfig.MYBATIS_CONFIG_LOCATION));
        return sessionFactory.getObject();
    }
}
```

- mybatis配置文件`mybatis-config.xml`

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <typeAliases>
        <package name="com.talkingdata.campaign.channel.bean.metadata"/>
        <package name="com.talkingdata.campaign.channel.bean.count"/>
    </typeAliases>
</configuration>
```

## mybatis自带数据源的禁用

自然用的都是自己配置的数据源，那么初始化的时候就不需要加载mybatis自带的数据库配置了，代码中对应`MybatisAutoConfiguration`这个bean，而且自带bean初始化的时候会注入数据源对应的bean，也就是上面的代码中的`counterDataSource`，但是由于程序中配置了两个datasource，所以在通过类型注入的时候，就存在datasource的两个bean，无法成功注入，程序报错。

可以在服务启动的时候排除`MybatisAutoConfiguration`的初始化。

```
@SpringBootApplication(exclude = {
        MybatisAutoConfiguration.class
})
```

如果使用的springcloud，那么需要将`@SpringCloudApplication`注解替换成其内部调用的注解，然后通过springboot的注解排除`MybatisAutoConfiguration`：

```
@EnableDiscoveryClient
@EnableCircuitBreaker
@SpringBootApplication(exclude = {
        DataSourceAutoConfiguration.class,
        MybatisAutoConfiguration.class
})
```

## mybatis的配置文件

springboot支持一些在`application.properties`的配置，如上面提到的`mybatis.type-aliases-package=com.example.domain.model`。

更详细的可以在配置了配置文件路径之后`sessionFactory.setConfigLocation()`，在`mybatis-config.xml`文件中进行配置。
