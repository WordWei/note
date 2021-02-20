### 1、@Transactional不生效

> @Transactional 使用private修饰时，被调用的事务不生效

原因： 只有定义在 public 方法上的 @Transactional 才能生效。原因是，Spring 默认通过动态代理的方式实现 AOP，对目标方法进行增强，**private 方法无法代理到**，Spring 自然也无法动态增强事务处理逻辑。 

>  @Transactional 修饰的方法，必须通过代理过的类从外部调用目标方法才能生效。 

即在Service层中使用this.transactionalMethod()是无法实现事务的，需要使用经过Spring代理过的Service对象进行调用。如下为一种实现方式：

```java
// XXXService

//解决事务失效
private IEtcBillService getService(){
    return SpringUtil.getBean(this.getClass());
}
```

```java
@Component
public class SpringUtil implements ApplicationContextAware {

    private static ApplicationContext applicationContext = null;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        SpringUtil.applicationContext = applicationContext;
    }

    public static <T> T getBean(Class<T> cla) {
        return applicationContext.getBean(cla);
    }

    public static <T> T getBean(String name, Class<T> cal) {
        return applicationContext.getBean(name, cal);
    }

    public static Object getBean(String name){
        return applicationContext.getBean(name);
    }

    public static String getProperty(String key) {
        return applicationContext.getBean(Environment.class).getProperty(key);
    }
}
```

这样在XXXService中增加获取Spring代理对象的私有方法，即可在service其他方法中使用该私有方法生成的代理对象进行事务的处理。

### 2、事务即便生效也不一定能回滚

-  只有异常传播出了标记了 @Transactional 注解的方法，事务才能回滚。 
-  默认情况下，出现 RuntimeException（非受检异常）或 Error 的时候，Spring 才会回滚事务。 

>  **受检异常一般是业务异常**，或者说是类似另一种方法的返回值，出现这样的异常可能业务还能完成，所以不会主动回滚；而 Error 或 RuntimeException 代表了非预期的结果，应该回滚 

解决办法：

 **在注解中声明，期望遇到所有的 Exception 都回滚事务（来突破默认不回滚受检异常的限制）**

```java
@Transactional(rollbackFor = Exception.class)
public void createUserRight2(String name) throws IOException {
    userRepository.save(new UserEntity(name));
    otherTask();
}
```

 一个复杂的业务逻辑，其中有数据库操作、IO 操作，在 IO 操作出现问题时希望让数据库事务也回滚，以确保逻辑的一致性。 

###  3、确认事务传播配置是否符合自己的业务逻辑 

 在有些业务逻辑中，可能会包含多次数据库操作，我们不一定希望将两次操作作为一个事务来处理，  需要仔细考虑事务传播的配置 。

场景：一个用户注册的操作，会插入一个主用户到用户表，还会注册一个关联的子用户。我们希望将子用户注册的数据库操作作为一个独立事务来处理，即使失败也不会影响主流程，即不影响主用户的注册。 

Controller

```java
@GetMapping("/wrong")
public int wrong(@RequestParam("name") String name) {
    try {
        userService.createUserWrong(new User(name));
    } catch (Exception ex) {
        log.error("createUserWrong failed, reason:{}", ex.getMessage());
    }
    return 0;
}
```

UserService

```java
@Transactional
@Override
public void createUserWrong(User user) {
    createMainUser(user);
    isubUserService.createSubUserWithExceptionWrong();
}

private void createMainUser(User user) {
    this.save(user);
    log.info("createMainUser finish");
}
```

SubUserService

```java
public void createSubUserWithExceptionWrong() {
    log.info("createSubUserWithExceptionWrong start");
    throw new RuntimeException("invalid status");
}
```

执行controller,日志如下：

```java
2021-02-20 15:29:52.332  INFO 31280 --- [nio-8088-exec-1] c.t.demo.service.impl.UserServiceImpl    : createMainUser finish
    
2021-02-20 15:29:52.332  INFO 31280 --- [nio-8088-exec-1] c.t.d.service.impl.IsubUserServiceImpl   : createSubUserWithExceptionWrong start
    
2021-02-20 15:29:52.336 ERROR 31280 --- [nio-8088-exec-1] c.t.demo.controller.UserController       : createUserWrong failed, reason:invalid status
```

 因为运行时异常逃出了 @Transactional 注解标记的 createUserWrong 方法，Spring 当然会回滚事务了。如果我们希望主方法不回滚，应该把子方法抛出的异常捕获了。 

修改UserService

```java
@Transactional
@Override
public void createUserWrong(User user) {
    createMainUser(user);
    try{ 
        isubUserService.createSubUserWithExceptionWrong(); 
    } catch (Exception ex) { 
        // 虽然捕获了异常，但是因为没有开启新事务，而当前事务因为异常已经被标记为rollback了，         // 所以最终还是会回滚。 
        log.error("create sub user error:{}", ex.getMessage()); 
    }
    
}

private void createMainUser(User user) {
    this.save(user);
    log.info("createMainUser finish");
}
```

重新执行controller,日志如下：

```java
2-20 15:44:56.196  INFO 30312 --- [nio-8088-exec-1] c.t.demo.service.impl.UserServiceImpl    : createMainUser finish
    
2021-02-20 15:44:56.199  INFO 30312 --- [nio-8088-exec-1] c.t.d.service.impl.IsubUserServiceImpl   : createSubUserWithExceptionWrong start
    
2021-02-20 15:44:56.200 ERROR 30312 --- [nio-8088-exec-1] c.t.demo.service.impl.UserServiceImpl    : create sub user error:invalid status
    
2021-02-20 15:44:56.205 ERROR 30312 --- [nio-8088-exec-1] c.t.demo.controller.UserController       : createUserWrong failed, reason:Transaction rolled back because it has been marked as rollback-only
```

在controller中的日志提示createUserWrong()方法居然回滚了， 而且是**静默回滚的**。之所以说是静默，是因为 createUserWrong方法本身并没有出异常，只不过提交后发现**子方法已经把当前事务设置为了回滚，无法完成提交。**这挺反直觉的。我们之前说，出了异常事务不一定回滚，这里说的却是不出异常，事务也不一定可以提交。原因是，**主方法注册主用户的逻辑和子方法注册子用户的逻辑是同一个事务，子逻辑标记了事务需要回滚，主逻辑自然也不能提交了。** 

修复方法：

 想办法让子逻辑在独立事务中运行，也就是改一下 SubUserService 注册子用户的方法，为注解加上 **propagation = Propagation.REQUIRES_NEW** 来设置 REQUIRES_NEW 方式的事务传播策略，也就是执行到这个方法时需要开启新的事务，并挂起当前事务

SubUserService

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
@Override
public void createSubUserWithExceptionWrong() {
    log.info("createSubUserWithExceptionWrong start");
    throw new RuntimeException("invalid status");
}
```

再次执行controller

```java
2021-02-20 15:56:46.124  INFO 28768 --- [nio-8088-exec-1] c.t.demo.service.impl.UserServiceImpl    : createMainUser finish
    
2021-02-20 15:56:46.131  INFO 28768 --- [nio-8088-exec-1] c.t.d.service.impl.IsubUserServiceImpl   : createSubUserWithExceptionWrong start
    
2021-02-20 15:56:46.134 ERROR 28768 --- [nio-8088-exec-1] c.t.demo.service.impl.UserServiceImpl    : create sub user error:invalid status
```

只有子事务进行了回滚，而父事务并未执行回滚，即成功。



注：

>  Spring默认事务采用动态代理方式实现。因此只能对public进行增强（考虑到CGLib和JDKProxy兼容，protected也不支持）。在使用动态代理增强时，方法内调用也可以考虑采用AopContext.currentProxy()获取当前代理类。 

















 



