
## 前言


对于开发人员来说，对用户输入的参数或者系统参数做校验，是日常工作之一。


很多小伙伴在写接口的时候，可能都会碰到一个问题：**参数校验应该怎么写？**


比如，开发一个用户注册接口，需要校验以下条件：


* 用户名不能为空，长度在 3 到 20 个字符之间；
* 密码不能为空，长度至少为 8 个字符；
* 年龄必须是正整数，不能超过 120；
* 邮箱必须符合标准格式。


乍一看，这种校验逻辑看起来很简单嘛，直接写几个 `if` 就完事了。


但真的这么简单吗？


接下来我们就从传统的参数校验入手，看看问题出在哪，然后再聊聊 **Spring Boot 中如何优雅地实现参数校验**，希望对你会有所帮助。


## 一、传统参数校验的问题


很多人可能会直接在 Controller 里手写校验逻辑，比如下面这个代码：



```
@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping("/register")
    public ResponseEntity register(@RequestBody Map request) {
        String username = (String) request.get("username");
        if (username == null || username.length() < 3 || username.length() > 20) {
            return ResponseEntity.badRequest().body("用户名不能为空，且长度必须在3到20之间");
        }

        String password = (String) request.get("password");
        if (password == null || password.length() < 8) {
            return ResponseEntity.badRequest().body("密码不能为空，且长度至少为8个字符");
        }

        Integer age = (Integer) request.get("age");
        if (age == null || age <= 0 || age > 120) {
            return ResponseEntity.badRequest().body("年龄必须是正整数，且不能超过120");
        }

        return ResponseEntity.ok("注册成功！");
    }
}

```

这段代码乍一看没什么问题，但如果仔细分析，会发现一堆隐患：


1. **代码冗余**：校验逻辑散落在 Controller 里，写起来麻烦，后期维护更是灾难。
2. **重复劳动**：类似的校验逻辑可能会出现在多个接口里，导致代码重复度极高。
3. **用户体验差**：返回的错误信息不统一、不规范，前端开发还得猜用户输入到底哪儿错了。
4. **扩展性差**：万一某天需要加新的校验规则，你可能要到处改代码。


所以，这种手写参数校验的方式，在简单场景下勉强能用，但如果业务变复杂，问题会越来越多。


那么问题来了，那有没有更优雅的方式来处理这些问题呢？


答：当然是有的。


## 二、Spring Boot 的参数校验机制


在 Spring Boot 中，我们可以使用 **Hibernate Validator**（Bean Validation 的参考实现）来实现参数校验。


它的核心思路是：**把校验逻辑从业务代码里抽离出来，用注解的方式声明校验规则**。


接下来我们一步步来看怎么实现。


### 1\. 使用注解进行参数校验


首先，定义一个用于接收用户注册参数的 DTO 对象：



```
@Data
public class UserRegistrationRequest {

    @NotNull(message = "用户名不能为空")
    @Size(min = 3, max = 20, message = "用户名长度必须在3到20之间")
    private String username;

    @NotNull(message = "密码不能为空")
    @Size(min = 8, message = "密码长度至少为8个字符")
    private String password;

    @NotNull(message = "年龄不能为空")
    @Min(value = 1, message = "年龄必须是正整数")
    @Max(value = 120, message = "年龄不能超过120")
    private Integer age;

    @Email(message = "邮箱格式不正确")
    private String email;
}

```

这里我们用了几个常见的校验注解：


* `@NotNull`：字段不能为空；
* `@Size`：限制字符串长度；
* `@Min` 和 `@Max`：限制数值范围；
* `@Email`：校验邮箱格式。


这些注解由 Hibernate Validator 提供，基本涵盖了日常开发中的大部分校验需求。


然后，在 Controller 中这样写：



```
@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping("/register")
    public ResponseEntity register(@Valid @RequestBody UserRegistrationRequest request) {
        return ResponseEntity.ok("注册成功！");
    }
}

```

注意这里的 `@Valid` 注解，它的作用是告诉 Spring：**对请求参数进行校验**。


### 2\. 统一处理校验错误


如果前端传的参数不合法，Spring 会抛出一个 `MethodArgumentNotValidException` 异常。默认情况下，这个异常返回的信息不太友好，可能是这样的：



```
{
  "timestamp": "2024-01-01T12:00:00.000+00:00",
  "status": 400,
  "error": "Bad Request",
  "message": "Validation failed for object='userRegistrationRequest'. Error count: 2",
  "path": "/api/users/register"
}

```

为了提升用户体验，我们可以用全局异常处理器来统一格式化错误信息：



```
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity> handleValidationException(MethodArgumentNotValidException ex) {
        Map errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error -> {
            errors.put(error.getField(), error.getDefaultMessage());
        });
        return ResponseEntity.badRequest().body(errors);
    }
}

```

现在，当参数校验失败时，返回的错误信息会变成这样：



```
{
  "username": "用户名长度必须在3到20之间",
  "password": "密码不能为空"
}

```

清晰又直观，用户一看就明白自己错在哪儿了。


最近建了一些工作内推群，收集了不少工作岗位，加我微信：su\_san\_java，备注：博客园\+所在城市，即可进群。


## 三、应对复杂场景的高级技巧


### 1\. 分组校验


有些场景下，不同的接口对参数的校验规则是不一样的，比如：


* 注册接口要求 `username` 和 `password` 是必填项；
* 更新接口只需要校验 `email` 和 `age`。


这种情况下，可以用 **分组校验** 来解决。


#### 定义校验分组



```
public interface RegisterGroup {}
public interface UpdateGroup {}

```

#### 在字段上指定分组



```
public class UserRequest {

    @NotNull(groups = RegisterGroup.class, message = "用户名不能为空")
    @Size(min = 3, max = 20, groups = RegisterGroup.class, message = "用户名长度必须在3到20之间")
    private String username;

    @NotNull(groups = RegisterGroup.class, message = "密码不能为空")
    private String password;

    @Email(groups = UpdateGroup.class, message = "邮箱格式不正确")
    private String email;

    @Min(value = 1, groups = UpdateGroup.class, message = "年龄必须是正整数")
    private Integer age;
}

```

#### 在 Controller 中指定分组



```
@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping("/register")
    public ResponseEntity register(@Validated(RegisterGroup.class) @RequestBody UserRequest request) {
        return ResponseEntity.ok("注册成功！");
    }

    @PutMapping("/update")
    public ResponseEntity update(@Validated(UpdateGroup.class) @RequestBody UserRequest request) {
        return ResponseEntity.ok("更新成功！");
    }
}

```

### 2\. 自定义校验注解


如果 Hibernate Validator 提供的注解不能满足需求，还可以自定义校验注解。例如，校验手机号格式。


#### 定义注解



```
@Documented
@Constraint(validatedBy = PhoneValidator.class)
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidPhone {
    String message() default "手机号格式不正确";
    Class[] groups() default {};
    Class <span class="hljs-keyword"extends Payload>[] payload() default {};
}

```

#### 实现校验逻辑



```
public class PhoneValidator implements ConstraintValidator {

    private static final String PHONE_REGEX = "^1[3-9]\\d{9}$";

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        return value != null && value.matches(PHONE_REGEX);
    }
}

```

#### 使用自定义注解



```
public class UserRequest {
    @ValidPhone
    private String phone;
}

```

## 四、总结


优雅的参数校验不仅能提高代码的可维护性，还能显著提升用户体验。


在 Spring Boot 中，通过使用 Hibernate Validator 提供的注解，配合分组校验、自定义校验和统一异常处理。


我们可以轻松实现简洁、高效、可扩展的参数校验机制。


优雅的参数校验的秘籍是：


1. 注解优先：能用注解解决的校验，就不要手写逻辑代码。
2. 分离校验逻辑：参数校验应该集中在 DTO 层，避免散落在业务代码中。
3. 全局统一异常处理：确保错误信息规范化、友好化。
4. 合理使用分组校验：根据接口需求灵活调整校验规则。
5. 覆盖边界条件：通过单元测试验证校验逻辑，确保没有漏网之鱼。


如果看了这篇文章有些收获，记得给我点赞和转发喔。


## 最后说一句(求关注，别白嫖我)


如果这篇文章对您有所帮助，或者有所启发的话，帮忙关注一下我的同名公众号：苏三说技术，您的支持是我坚持写作最大的动力。
求一键三连：点赞、转发、在看。
关注公众号：【苏三说技术】，在公众号中回复：进大厂，可以免费获取我最近整理的10万字的面试宝典，好多小伙伴靠这个宝典拿到了多家大厂的offer。


 本博客参考[樱花宇宙官网](https://yzygzn.com)。转载请注明出处！
