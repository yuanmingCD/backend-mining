基于spring-boot2.0 构建快速开发的ssh框架



spring boot 2.0
https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready-endpoints-enabling-endpoints

包括以下内容

### 

### 异常处理与结果封装
异常处理
使用ExceptionHandler进行统一的异常处理
@ExceptionHandle注解

结果封装
之前也在一直尝试将结果进行统一封装
个人认为将不同的状态进行统一处理即可，如果统一封装会很复杂，需要在controller中进行统一封装

 ### 文档

swagger

### SSL connection

### 各层职能

dao 与数据库的数据交互
https://www.ibm.com/developerworks/cn/opensource/os-cn-spring-jpa/index.html
https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.auditing
spring data jpa 
一种约定规范的实现

Repository
CrudRepository
​	<S extends T> S save(S entity);
​	<S extends T> Iterable<S> saveAll(Iterable<S> entities);
​	Optional<T> findById(ID id);
​	boolean existsById(ID id);
​	Iterable<T> findAll();
​	Iterable<T> findAllById(Iterable<ID> ids);
​	long count();
​	void deleteById(ID id);
​	void delete(T entity);
​	void deleteAll(Iterable<? extends T> entities);
​	void deleteAll();
PagingAndSortingRepository
​     Iterable<T> findAll(Sort sort);
​	Page<T> findAll(Pageable pageable);
JpaRepository
​    

	List<T> findAll();
	List<T> findAll(Sort sort);
	List<T> findAllById(Iterable<ID> ids);
	
	<S extends T> List<S> saveAll(Iterable<S> entities);
	
	void flush();
	<S extends T> S saveAndFlush(S entity);
	void deleteInBatch(Iterable<T> entities);
	void deleteAllInBatch();
	
	T getOne(ID id);
	@Override
	<S extends T> List<S> findAll(Example<S> example);
	@Override
	<S extends T> List<S> findAll(Example<S> example, Sort sort);

service 业务逻辑
controller 文档以及对外接口

分工明确，不要跨层级的调用，个人觉得即便dao层有方法可以直接使用，在service添加一个对应方法中转一下，看起来会清晰很多


18516736066

### 缓存
spring boot 支持很多缓存
通过spring.cache.type 指定



使用方法
@Cacheable	触发缓存发布，方法执行前会先检查缓存中有没有数据，如果有就直接返回，没有就执行方法，返回结果并将结果缓存起来。(triggers cache population)
@CacheEvict	触发缓存删除(triggers cache eviction)
@CachePut	执行方法并将返回结果更新缓存(updates the cache without interfering with the method execution)
@Caching	构建多个缓存操作应用在一个方法上(regroups multiple cache operations to be applied on a method)
@CacheConfig	共享一些类级别的缓存相关配置(shares some common cache-related settings at class-level)





### 日志

spring-boot默认支持logbck
https://www.concretepage.com/spring-boot/spring-boot-logging-example

日志配置在对应环境的yml文件中进行修改

关于日志配置

接入elk





### 指标监控

1.x 和 2.x 区别

通用指标
https://aboullaite.me/actuator-in-spring-boot-2-0/

自定义指标

跨域支持

指标持久化
influxdb
https://jasper-zhang1.gitbooks.io/influxdb/content/Introduction/installation.html

异步支持



### 多环境支持
通常会分为dev，test，prod等环境，针对不同的环境有不同的配置文件


协议支持rpc http2 ssl

### 测试
https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html#boot-features-testing-spring-boot-applications-testing-autoconfigured-tests

### api标准
关于rest标准，已经有一些讨论



### 版本管理



### 鉴权

### 限流

单机限流，采用guaua



### 负载均衡和水平扩展

### 治理



### 网关

### 降级

### 



部署


运维


