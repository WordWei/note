### @Valid：

 @Valid注解用于校验，所属包为：javax.validation.Valid。 

 首先需要在实体类的相应字段上添加用于充当校验条件的注解，如：@Min,如下代码（age属于Girl类中的属性）： 

> ```java
> @Min(value = 18,message = "未成年禁止入内")  
> private Integer age; 
> ```

 其次在controller层的方法的要校验的参数上添加@Valid注解，并且需要传入BindingResult对象，用于获取校验失败情况下的反馈信息，如下代码： 

```java
@PostMapping("/girls")  
public Girl addGirl(@Valid Girl girl, BindingResult bindingResult) {  
    if(bindingResult.hasErrors()){    					                 	           System.out.println(bindingResult.getFieldError().getDefaultMessage()); 
        return null;  
    }  
    return girlResposity.save(girl);  
}
```

 注: 通常不在这里处理异常, 由统一的exceptioin全局异常处理 

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    
    @ResponseBody
    @ExceptionHandler(value = MethodArgumentNotValidException.class)
    public JsonResult violationException(MethodArgumentNotValidException exception) {
        // 不带任何参数访问接口,会抛出 BindException
        // 因此，我们只需捕获这个异常，并返回我们设置的 message 即可
        String message = exception.getBindingResult().getAllErrors().get(0).getDefaultMessage();
        return JsonResult.fail(message);
    }

}
```

### @Validated：

 @Valid是javax.validation里的。 

 @Validated是@Valid 的一次封装，是Spring提供的校验机制使用。 

### 两者的区别

在检验Controller的入参是否符合规范时，使用@Validated或者@Valid在基本验证功能上没有太多区别。但是在**分组**、**注解地方**、**嵌套验证**等功能上两个有所不同：

#### 分组

@Valid 作为标准JSR-303规范，还没有吸收分组的功能。 

@Validated Spring's JSR-303规范，是标准JSR-303的一个变种, 提供了一个分组功能，可以在入参验证时，根据不同的分组采用不同的验证机制。  

#### 注解地方

 @Valid：可以用在方法、构造函数、方法参数和成员属性（字段）上 

 @Validated：可以用在类型、方法和方法参数上。但是不能用在成员属性（字段）上 

 两者是否能用于成员属性（字段）上直接影响能否提供嵌套验证的功能。 

#### 嵌套验证

 在比较两者嵌套验证时，先说明下什么叫做嵌套验证。比如我们现在有个实体叫做Item： 

```java
public class Item {

    @NotNull(message = "id不能为空")
    @Min(value = 1, message = "id必须为正整数")
    private Long id;

    @NotNull(message = "props不能为空")
    @Size(min = 1, message = "至少要有一个属性")
    private List<Prop> props;
}
```

 Item带有很多属性，属性里面有属性id，属性值id，属性名和属性值，如下所示： 

```java
public class Prop {

    @NotNull(message = "pid不能为空")
    @Min(value = 1, message = "pid必须为正整数")
    private Long pid;

    @NotNull(message = "vid不能为空")
    @Min(value = 1, message = "vid必须为正整数")
    private Long vid;

    @NotBlank(message = "pidName不能为空")
    private String pidName;

    @NotBlank(message = "vidName不能为空")
    private String vidName;
}
```

 属性这个实体也有自己的验证机制，比如属性和属性值id不能为空，属性名和属性值不能为空等。 

 现在我们有个ItemController接受一个Item的入参，想要对Item进行验证，如下所示： 

```java
@RestController
public class ItemController {

    @RequestMapping("/item/add")
    public void addItem(@Validated Item item, BindingResult bindingResult) {
        doSomething();
    }
}
```

 在上图中，如果Item实体的props属性不额外加注释，只有@NotNull和@Size，无论入参采用@Validated还是@Valid验证，Spring Validation框架只会对Item的id和props做非空和数量验证，不会对props字段里的Prop实体进行字段验证，也就是**@Validated**和**@Valid**加在方法参数前，都不会自动对参数进行嵌套验证。也就是说如果传的List<Prop>中有Prop的pid为空或者是负数，入参验证不会检测出来。 

 为了能够进行嵌套验证，必须手动在Item实体的props字段上明确指出这个字段里面的实体也要进行验证。由于@Validated不能用在成员属性（字段）上，但是@Valid能加在成员属性（字段）上，而且@Valid类注解上也说明了它支持嵌套验证功能，那么我们能够推断出：@Valid加在方法参数时并不能够自动进行嵌套验证，而是用在需要嵌套验证类的相应字段上，来配合方法参数上@Validated或@Valid来进行嵌套验证。 

 我们修改Item类如下所示： 

```java
public class Item {

    @NotNull(message = "id不能为空")
    @Min(value = 1, message = "id必须为正整数")
    private Long id;

    @Valid // 嵌套验证必须用@Valid
    @NotNull(message = "props不能为空")
    @Size(min = 1, message = "props至少要有一个自定义属性")
    private List<Prop> props;
}
```

 然后我们在ItemController的addItem函数上再使用@Validated或者@Valid，就能对Item的入参进行嵌套验证。此时Item里面的props如果含有Prop的相应字段为空的情况，Spring Validation框架就会检测出来，bindingResult就会记录相应的错误。 

### 总结

@Validated：用在方法入参上无法单独提供嵌套验证功能。**不能用在成员属性（字段）上**，也无法提示框架进行嵌套验证。能配合嵌套验证注解@Valid进行嵌套验证。

@Valid：用在方法入参上无法单独提供嵌套验证功能。**能够用在成员属性（字段）上**，提示验证框架进行嵌套验证。能配合嵌套验证注解@Valid进行嵌套验证。