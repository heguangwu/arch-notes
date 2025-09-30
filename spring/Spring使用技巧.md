<!-- omit in toc -->
# `Spring` 使用技巧

大多数代码来自[`RuoYi`](https://ruoyi.vip/)。

- [`PreAuthorize` 方法级权限控制](#preauthorize-方法级权限控制)
  - [开启注解支持](#开启注解支持)
  - [基本用法](#基本用法)
- [`Aspect`切面](#aspect切面)
- [自定义校验注解](#自定义校验注解)
- [`InitBinder` 请求参数绑定及校验](#initbinder-请求参数绑定及校验)
  - [参数绑定](#参数绑定)
  - [校验](#校验)


## `PreAuthorize` 方法级权限控制

### 开启注解支持

要使用`PreAuthorize`首先就需要在配置类中开启安全注解支持：

```java
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;

@Configuration
@EnableMethodSecurity(prePostEnabled = true, securedEnabled = true)
public class SecurityConfig {
    // 其它spring安全配置
}
```

### 基本用法

```java
@RestController
public class SystemController {

    //使用自定义检查器
    @PreAuthorize("@ps.hasPermission('system:user:list')")
    @GetMapping("/users")
    public TableDataInfo list() {
        //web处理业务逻辑
    }

    //使用spring自带，检查角色名是否为 ROLE_ADMIN
    @PreAuthorize("hasRole(ADMIN)")
    @PostMapping("/users")
    public TableDataInfo addUser(User user) {
        //web处理业务逻辑
    }
}

//自定义检查器
@Service("ps")
public class PermissionService {

    public boolean hasPermission(String permission) {
        //获取登录用户
        LoginUser loginUser = (LoginUser)SecurityContextHolder.getContext().getAuthentication().getPrincipal();
        //判断用户权限是否包含输入权限
        return loginUser.getPermissions().contains(permission);
    }
}
```

## `Aspect`切面

以日志切面为例：

```java
@Target({ ElementType.PARAMETER, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Log
{
    /**
     * 模块
     */
    public String title() default "";

    /**
     * 功能
     */
    public BusinessType businessType() default BusinessType.OTHER;
}

@Aspect
@Component
public class LogAspect {
    /**
     * 处理请求前执行
     */
    @Before(value = "@annotation(controllerLog)")
    public void doBefore(JoinPoint joinPoint, Log controllerLog)
    {
        TIME_THREADLOCAL.set(System.currentTimeMillis());
    }

    /**
     * 处理完请求后执行
     *
     * @param joinPoint 切点
     */
    @AfterReturning(pointcut = "@annotation(controllerLog)", returning = "jsonResult")
    public void doAfterReturning(JoinPoint joinPoint, Log controllerLog, Object jsonResult)
    {
        handleLog(joinPoint, controllerLog, null, jsonResult);
    }

    /**
     * 拦截异常操作
     * 
     * @param joinPoint 切点
     * @param e 异常
     */
    @AfterThrowing(value = "@annotation(controllerLog)", throwing = "e")
    public void doAfterThrowing(JoinPoint joinPoint, Log controllerLog, Exception e)
    {
        handleLog(joinPoint, controllerLog, e, null);
    }
}
```

在其它函数中使用`Log`注解：

```java
@Log(title = "参数管理", businessType = BusinessType.EXPORT)
@PostMapping("/export")
public void export(HttpServletResponse response, SysConfig config)
{
    List<SysConfig> list = configService.selectConfigList(config);
    ExcelUtil<SysConfig> util = new ExcelUtil<SysConfig>(SysConfig.class);
    util.exportExcel(response, list, "参数数据");
}
```

## 自定义校验注解

下面以手机号码校验为例：

```java
// 自定义注解
@Target({FIELD, PARAMETER})
@Retention(RUNTIME)
@Constraint(validatedBy = {PhoneValidator.class}) // 关联验证器
public @interface Phone {
    String message() default "手机号格式无效";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

验证器接口`javax.validation.ConstraintValidator`是 `Java Bean Validation` 规范中用于自定义约束验证逻辑的核心接口。

```java
import javax.validation.ConstraintValidatorContext;

public class PhoneValidator implements ConstraintValidator<Phone, String> {
    private static final String PHONE_REGEX = "^1[3-9]\\d{9}$";

    @Override
    public void initialize(Phone constraintAnnotation) {
        // 可通过 constraintAnnotation 获取注解中的参数（如自定义正则）
    }

    @Override
    public boolean isValid(String phone, ConstraintValidatorContext context) {
        // 验证逻辑：非空且符合手机号正则
        if (phone == null) {
            return false;
        }
        return phone.matches(PHONE_REGEX);
    }
}
```

使用的时候比较简单，直接使用注解即可：

```java
public class User {
    @Phone
    private String phone;
}
```

## `InitBinder` 请求参数绑定及校验

### 参数绑定

以日期为例：

```java
@RestController
@RequestMapping("/users")
public class UserController {

    // 初始化 WebDataBinder，配置日期绑定规则
    @InitBinder
    public void initBinder(WebDataBinder binder) {
        // 1. 绑定 java.util.Date 类型：将 "yyyy-MM-dd" 格式的字符串转为 Date
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
        dateFormat.setLenient(false); // 严格解析（避免非法日期如 2023-13-01 被转换）
        binder.registerCustomEditor(Date.class, new CustomDateEditor(dateFormat, true));

        // 2. 绑定 java.time.LocalDate 类型（Java 8+）
        binder.registerCustomEditor(LocalDate.class, new PropertyEditorSupport() {
            @Override
            public void setAsText(String text) throws IllegalArgumentException {
                if (text != null && !text.isEmpty()) {
                    setValue(LocalDate.parse(text, DateTimeFormatter.ofPattern("yyyy-MM-dd")));
                }
            }
        });
    }

    // 测试：接收包含日期参数的 User 对象
    @PostMapping("/add")
    public String addUser(User user) {
        // User 类包含 birthday(Date 类型) 和 registerDate(LocalDate 类型)
        System.out.println("生日：" + user.getBirthday());
        System.out.println("注册日期：" + user.getRegisterDate());
        return "用户添加成功";
    }
}

// User 实体类
class User {
    private String name;
    private Date birthday;
    private LocalDate registerDate;
    // getter/setter
}
```

### 校验

```java
@RestController
public class OrderController {

    // 自定义订单验证器
    private final OrderValidator orderValidator;

    @Autowired
    public OrderController(OrderValidator orderValidator) {
        this.orderValidator = orderValidator;
    }

    @InitBinder
    public void initBinder(WebDataBinder binder) {
        // 为 Order 类型注册自定义验证器
        binder.addValidators(orderValidator);
    }

    // 提交订单：@Valid 触发验证（包括自定义验证器）
    @PostMapping("/orders")
    public String submitOrder(@Valid Order order, BindingResult result) {
        if (result.hasErrors()) {
            return "订单验证失败：" + result.getAllErrors();
        }
        return "订单提交成功";
    }
}

// 自定义订单验证器
@Component
class OrderValidator implements Validator {
    @Override
    public boolean supports(Class<?> clazz) {
        return Order.class.isAssignableFrom(clazz); // 只支持 Order 类型
    }

    @Override
    public void validate(Object target, Errors errors) {
        Order order = (Order) target;
        // 自定义验证逻辑：订单金额不能为负
        if (order.getAmount() < 0) {
            errors.rejectValue("amount", "amount.negative", "订单金额不能为负数");
        }
    }
}
```
