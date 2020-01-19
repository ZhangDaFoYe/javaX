

![Man, Glasses, Hipster, Beard, Adult](assets/%E6%88%91%E6%89%80%E4%BD%BF%E7%94%A8%E7%9A%84java%E4%BB%A3%E7%A0%81%E6%8A%80%E5%B7%A7/man-597179__340.jpg)

# 前言

> 罗列工作中实际使用的一些代码技巧或者叫工具类;知识无大小,希望大家都有收获

# 实用技巧

## rpc服务出参统一化

> 什么,出参统一化有什么好说的????? 我不知道你们有没有遇到过多少五花八门的外部服务提供的返回对象,可能别人没有规范约束,我们管不了,但是从我们这里出去的,我们可以强制约束一下,不然发生新老交替,这代码还能看吗

首先出参都叫xxDTO的,阿里java开发手册提到过;再者我们是提供服务的一方,错误码code,错误信息msg,以及返回结果data都是要明确体现出来的,像下面这样

```java
public class TradeResultDTO<T> implements Serializable {
    /**
     * 默认失败编码
     */
    private static final String DEFAULT_FAIL_CODE = "500";
    private boolean success;
    private String code;
    private String msg;
    private T data;
        public static <T> TradeResultDTO<T> success(T data) {
        return base("200", null, true, data);
    }

    public static <T> TradeResultDTO<T> fail(String code, String msg) {
        return base(code, msg, false, null);
    }

    public static <T> TradeResultDTO<T> fail(String msg) {
        return base(DEFAULT_FAIL_CODE, msg, false, null);
    }

    public static <T> TradeResultDTO<T> success() {
        return base("200", null, true, null);
    }

    public static <T> TradeResultDTO<T> fail(IError iError) {
        return base(iError.getErrorCode(), iError.getErrorMsg(), false, null);
    }
}
```

统一对象返回的结构就是上面这样

接着这个我想说的是,作为服务提供方,如果这个接口提供了返回值,我拿创建订单接口举例

```java
/**
 * 创建交易单，业务系统发起
 *
 * @param req 创建入参
 * @return 返回创建信息
 */
TradeResultDTO<TradeCreateOrderResponseDTO> createOrder(TradeCreateOrderRequestDTO req)
```

比如这个TradeCreateOrderResponseDTO 返回了订单号之类的基本信息,这个接口对于具体业务场景只能产生一笔订单号,我之前遇到过对方只是提示什么的错误信息(订单已存在),是的没错,他做了幂等,但是他没有返回原值,那对应的调用方进入了死循环,可能对应的业务系统,需要返回的订单号落到自己的数据库,一直异常一直回滚重试,没有任何意义;所以作为一个负责人的服务提供方,类似这种情况,**如果你的方法有幂等,那么请一定返回存在的那个对象**;

## 异常统一化

> 统一化使用,杜绝项目出现各种各样的自定义异常

### 对外统一抛出异常

我使用的统一化有两个方面:

- 抛出的自定义异常不要五花八门,一个就够了;很多人喜欢写各种各样的异常,初衷其实没错,但是人多手杂,自定义异常可能越写越乱;

- 异常信息最好尽可能的具体化,描述出业务产生异常原因就可以了,比如入参校验的用户信息不存在之类的;或者在调用用户中心的时候,捕获了该异常,此时你只需定义调用用户中心异常就可以了

  

然后看下工作中比较推荐的:

首先,需要搞一个统一抛出异常的工具 **ExceptionUtil**(这里Exceptions是公司内部统一前后端交互的,基于这个包装一个基础util,统一整个组抛异常的入口)

```java
public class ExceptionUtil {
    public static OptimusExceptionBase fail(IError error) throws OptimusExceptionBase {
        return Exceptions.fail(errorMessage(error));
    }

    public static OptimusExceptionBase fail(IError error, String... msg) throws OptimusExceptionBase {
        return Exceptions.fail(errorMessage(error, msg));
    }

    public static OptimusExceptionBase fault(IError error) throws OptimusExceptionBase {
        return Exceptions.fault(errorMessage(error));
    }

    public static OptimusExceptionBase fault(IError error, String... msg) throws OptimusExceptionBase {
        return Exceptions.fault(errorMessage(error, msg));
    }


    private static ErrorMessage errorMessage(IError error) {
        if (error == null) {
            error = CommonErrorEnum.DEFAULT_ERROR;
        }
        return ErrorMessage.errorMessage("500", "[" + error.getErrorCode() + "]" + error.getErrorMsg());
    }

    private static ErrorMessage errorMessage(IError error, String... msg) {
        if (error == null) {
            error = CommonErrorEnum.DEFAULT_ERROR;
        }
        return ErrorMessage.errorMessage("500", "[" + error.getErrorCode() + "]" + MessageFormat.format(error.getErrorMsg(), msg));
    }
}
```

其实上面代码里也体现出来IError这个接口了,我们的错误枚举都需要实现这个异常接口,方便统一获取对应的错误码和错误信息,这里列举一下通用异常的定义

```java
public interface IError {
    String getErrorCode();

    String getErrorMsg();
}
@AllArgsConstructor
public enum CommonErrorEnum implements IError {
    /**
     *
     */
    DEFAULT_ERROR("00000000", "系统异常"),
    REQUEST_OBJECT_IS_NULL_ERROR("00000001", "入参对象为空"),
    PARAMS_CANNOT_BE_NULL_ERROR("00000002", "参数不能为空"),
    BUILD_LOCK_KEY_ERROR("00000003", "系统异常:lock key异常"),
    REPEAT_COMMIT_ERROR("00000004", "正在提交中，请稍候");

    private String code;
    private String msg;

    @Override
    public String getErrorCode() {
        return code;
    }

    @Override
    public String getErrorMsg() {
        return msg;
    }
}
```

类似上面CommonErrorEnum的方式我们可以按照具体业务定义相应的枚举,比如OrderErrorEnum、PayErrorEnum之类,不仅具有区分度而且,也能瞬速定位问题;

**所以对外抛出异常统一化就一把梭:**统一util和业务错误枚举分类

### 对内统一捕获外部异常

> 很多时候我们需要调用别人的服务然后写了一些adapter适配类,然后在里面trycatch一把;其实这时候可以利用aop好好搞一把就完事了,并且统一输出adapter层里面的日志

```java
    public Object transferException(ProceedingJoinPoint joinPoint) {
        try {
            Object result = joinPoint.proceed();
            log.info("adapter result:{}", JSON.toJSONString(result));
            return result;
        } catch (Throwable exception) {
            MethodSignature signature = (MethodSignature) joinPoint.getSignature();
            Method method = signature.getMethod();
            log.error("{}.{} throw exception", method.getDeclaringClass().getName(), method.getName(), exception);
            throw ExceptionUtils.fault(CommonErrorEnum.ADAPTER_SERVICE_ERROR);
            return null;
        }
    }
```

上面这段统一捕获了外部服务,记录异常日志,避免了每个adapter类重复捕获的问题

## 参数校验

> 用过swagger的应该了解api方法里有对应的注解属性约束是否必填项,但是如果判断不是在api入口又或者没有类似的注解,你们一般怎么做的,下面给出我自己的一种简单工具;有更好大佬的可以推荐一下

ParamCheckUtil.java

```java
@Slf4j
public class ParamCheckUtil {

    /**
     * 校验请求参数是否为空
     *
     * @param requestParams 请求入参
     * @param keys          属性值数组
     */
    public static void checkParams(Object requestParams, String... keys) {
        if (null == requestParams) {
            throw ExceptionUtil.fault(CommonErrorEnum.REQUEST_OBJECT_IS_NULL_ERROR);
        }
        StringBuilder sb = new StringBuilder();
        for (String fieldName : keys) {
            Object value = null;
            Type type = null;
            try {
                String firstLetter = fieldName.substring(0, 1).toUpperCase();
                String getter = "get" + firstLetter + fieldName.substring(1);
                Method method = requestParams.getClass().getMethod(getter);
                value = method.invoke(requestParams);
                type = method.getReturnType();
            } catch (Exception e) {
                log.error("获取属性值出错，requestParams={}, fieldName={}", requestParams, fieldName);
            } finally {
                // 判空标志 String/Collection/Map特殊处理
                boolean isEmpty =
                        (String.class == type && StringUtil.isEmpty((String) value))
                                || (Collection.class == type && CollectionUtils.isEmpty((Collection<? extends Object>) value))
                                || (Map.class == type && CollectionUtils.isEmpty((Collection<? extends Object>) value))
                                || (null == value);
                if (isEmpty) {
                    if (sb.length() != 0) {
                        sb.append(",");
                    }
                    sb.append(fieldName);
                }
            }
        }

        if (sb.length() > 0) {
            log.error(sb.toString() + CommonErrorEnum.PARAMS_CANNOT_BE_NULL_ERROR.getErrorMsg());
            throw ExceptionUtil.fault(CommonErrorEnum.PARAMS_CANNOT_BE_NULL_ERROR, sb.toString() + CommonErrorEnum.PARAMS_CANNOT_BE_NULL_ERROR.getErrorMsg());
        }
    }

    // test
    public static void main(String[] args) {
        TradeCreateOrderRequestDTO tradeCreateOrderRequestDTO = new TradeCreateOrderRequestDTO();
        tradeCreateOrderRequestDTO.setBusinessNo("");
        ParamCheckUtil.checkParams(tradeCreateOrderRequestDTO, "businessNo", "tradeType", "tradeItemDTOS");
    }

}
```

基于了上面统一异常的形式,只要参数校验出空我就抛出异常中断程序,并且告知其缺什么参数

我在业务代码需要判断字段非空的地方只需要一行就够了,就行下面这样

```java
ParamCheckUtil.checkParams(tradeCreateOrderRequestDTO, "businessNo", "tradeType", "tradeItemDTOS");
```

而不是我们常用的一堆判断,像下面这样;看到这些我人都晕了,一次两次就算了,一大段全是这种

```java
if (null == tradeCreateOrderRequestDTO) {
// 提示tradeCreateOrderRequestDTO为空
}
if (StringUtil.isEmpty(tradeCreateOrderRequestDTO.getBusinessNo())) {
// 提示businessNo为空
}
if (StringUtil.isEmpty(tradeCreateOrderRequestDTO.getTradeType())) {
// 提示tradeType为空
}
if (CollectionUtils.isEmpty(tradeCreateOrderRequestDTO.getTradeItemDTOS())) {
// 提示tradeItemDTOS列表为空
}
```

如果你是上面说的这种形式,不妨试试我提供的这种

## bean相关

### 对象的构造

> 关于对象的构造,我想提两点,构造变的对象和不变的对象

- 构造不变对象,使用builder,不提供set方法,推荐使用lombok @Builder

```java
@Builder
public class UserInfo {
    private String id;
    private String name;

    public static void main(String[] args) {
        UserInfo userInfo = UserInfo.builder().id("a").name("name").build();
    }
}
```

- 构造可变对象,推荐提供链式调用形式 使用lombok @Accessors(chain = true)注解

```java
@Data
@Accessors(chain = true)
public class CardInfo {
    private String id;
    private String name;
    public static void main(String[] args) {
        CardInfo cardInfo = new CardInfo().setId("c").setName("name");
    }
}
```

### 对象转换

> 就一把梭:lambda工具类+mapstruct进行转换

BeanConvertUtil.java 通用的对象、list、Page转换

```java
public class BeanConvertUtil {
    /**
     * 对象转换
     *
     * @param source     源对象
     * @param convertFun T -> R lambda转换表达式
     * @param <T>        输入类型
     * @param <R>        输出类型
     * @return 返回转化后输出类型的对象
     */
    public static <T, R> R convertObject(T source, Function<T, R> convertFun) {
        if (null == source) {
            return null;
        }
        return convertFun.apply(source);
    }

    /**
     * Page转换
     *
     * @param page       源对象
     * @param convertFun T -> R lambda转换表达式
     * @param <T>        输入类型
     * @param <R>        输出类型
     * @return 返回转化后输出类型的对象
     */
    public static <T, R> Page<R> convertPage(Page<T> page, Function<T, R> convertFun) {
        if (Objects.isNull(page)) {
            return new Page<>(0, 1, 10, Collections.emptyList());
        }
        List<R> pageList = convertList(page.getItems(), convertFun);
        return new Page<>(page.getTotalNumber(), page.getCurrentIndex(), page.getPageSize(), pageList);
    }

    /**
     * ListData转换
     *
     * @param inputList  数据源
     * @param convertFun T -> R lambda转换表达式
     * @param <T>        输入类型
     * @param <R>        输出类型
     * @return 输出
     */
    public static <T, R> List<R> convertList(List<T> inputList, Function<T, R> convertFun) {
        if (org.springframework.util.CollectionUtils.isEmpty(inputList)) {
            return Lists.newArrayList();
        }
        return inputList
                .stream()
                .map(convertFun)
                .collect(Collectors.toList());
    }
}

```

实战使用,在lambda方法进行转换: 先转换相同属性,再进行剩余属性赋值

```java
 public interface OrderConverter {
    OrderConverter INSTANCE = Mappers.getMapper(OrderConverter.class);
		// 入参进行相同属性转换
    TradeOrderDO createOrder2TradeOrderDO(TradeCreateOrderRequestDTO req);
}
 TradeOrderDO mainTradeOrder = BeanConvertUtil.convertObject(req, x -> {
     TradeOrderDO tod = OrderConverter.INSTANCE.createOrder2TradeOrderDO(req);
     tod.setOrderType(mainOrderType);
     tod.setOrderCode(snowflakeIdAdapterService.getId());
     tod.setOrderStatus(TradeStateEnum.ORDER_CREATED.getValue());
     tod.setDateCreate(new Date());
     tod.setDateUpdate(new Date());
     return tod;
});
```

其实对象转换也可以完全通过mapstruct提供的一些表达式进行转换,但是有时候写那个感觉不是很直观,其实都可以,我比较喜欢我这种形式,大家有建议也可以提出

## NPE解决指南

### 1.null值手动判断[强制]
嵌套取值<3 推荐 null值判断(PS:强制null写在前面,别问为什么,问就是这样写你会意识到这里要NPE)
> 学会这点 基本意识有了

```java
null!=obj&&null!=obj.getXX()
```
### 2.Optional
#### 2.1 Optional嵌套取值[强制]
> 参数>=3的取值操作
> 学会这点 基本告别NPE

这里以OrderInfo对象为例 获取purchaseType
```java
Optional<OrderInfo> optional = Optional.ofNullable(dto);
Integer purchaseType = optional.map(OrderInfo::getOrderCarDTO)
    							 .map(OrderCarDTO::getPurchaseType)
    							 .orElse(null);
```
如果对取出的值如需再次进行判断操作 参考第1点
#### 2.2 中断抛出异常[按需]
> 还是以上面的例子

```java
{
	// ...
    optional.map(OrderInfo::getOrderDTO).map(OrderDTO::getOrderBusinessType)
        	  .orElseThrow(() -> new Exception("获取cityCode失败"));
}
```
如果依赖某些值,可尽早fail-fast
### 3.对象判空[强制]
```java
Objects.equals(obj1,obj2);
```
### 4.Boolean值判断[强制]
弃用以下方式谢谢(PS:很多时候null判断你会丢的)
```java
null!=a&&a;
```
正确食用
```java
Boolean.TRUE.equals(a);
```
### 5.list正确判空姿势[强制]
```java
if (CollectionUtils.isEmpty(list)) {
    // fail fast
    // return xxObj or return;
}
List<Object> safeList = list.stream().filter(Objects::nonNull).collect(Collectors.toList());
if (CollectionUtils.isEmpty(safeList)) {
    // fail fast
    // return xxObj or return;
}
```
### 6.String正确判空姿势[强制]
```java
// 不为空
if (StringUtil.isNotEmpty(s)) {
	// ...
}
```
### 7.包装类型转换判空[强制]
特别是遍历过程中使用，需要判断是否为空。

```java
int i = 0;
list.forEach(item -> {
    if(Objects.nonNull(item.getType)){
     i += item.getType; //item.getType 返回 Integer   
    }
});
```

### 小结
> 融会贯通以上几招绝对告别NPE

# END

> 未完待续,大家如果有好的建议,希望在留言中提出