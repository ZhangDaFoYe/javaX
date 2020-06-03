# [你在教我做事?]系列之老三层架构到DDD的实战

> 你在教我做事啊?这一波我又大了七八个,还全部大残

## 订单案例

> 为了避免整篇文章比较枯燥,这里列举一个现实生产的业务案例,暂定叫做XX买车平台

![image-20200601162006576](assets/%E4%BC%A0%E7%BB%9F%E8%80%81%E4%B8%89%E5%B1%82%E6%9E%B6%E6%9E%84%E6%BC%94%E8%BF%9B%E5%88%B0DDD%E7%9A%84%E5%AE%9E%E6%88%98/image-20200601162006576.png)

用户买车会经过一系列的动作,经过风控 => 生成订单 => 签署合同 => 付钱 => 扣除库存 => 进行配货 => 通知提车 => 交易成功

这个流程是精简过的,实际的生产流程比这个复杂点,但基本可以说清楚一个买车流程的交互了

### 传统老三层MVC架构

> 首先我们来回忆或者复习一下经典的MVC架构

#### MVC的一些概念

**模型（Model）** 用于封装与应用程序的业务逻辑相关的数据以及对数据的处理方法。“ Model ”有对数据直接访问的权力，例如对数据库的访问。“Model”不依赖“View”和“Controller”，也就是说， Model 不关心它会被如何显示或是如何被操作。但是 Model 中数据的变化一般会通过一种刷新机制被公布。为了实现这种机制，那些用于监视此 Model 的 View 必须事先在此 Model 上注册，从而，View 可以了解在数据 Model 上发生的改变。

**视图（View）**能够实现数据有目的的显示（理论上，这不是必需的）。在 View 中一般没有程序上的逻辑。为了实现 View 上的刷新功能，View 需要访问它监视的数据模型（Model），因此应该事先在被它监视的数据那里注册。

**控制器（Controller）**起到不同层面间的组织作用，用于控制应用程序的流程。它处理事件并作出响应。“事件”包括用户的行为和数据 Model 上的改变。

![image-20200601163243001](assets/%E4%BC%A0%E7%BB%9F%E8%80%81%E4%B8%89%E5%B1%82%E6%9E%B6%E6%9E%84%E6%BC%94%E8%BF%9B%E5%88%B0DDD%E7%9A%84%E5%AE%9E%E6%88%98/image-20200601163243001.png)

MODEL层映射到VIEW层,给用户展示;用户对MODEL使用更新通过CONTROLLER层去操作

那么这种MVC架构映射到我们的Java的代码是怎样的结构呢,详见下图

![image-20200601171228009](assets/%E4%BC%A0%E7%BB%9F%E8%80%81%E4%B8%89%E5%B1%82%E6%9E%B6%E6%9E%84%E6%BC%94%E8%BF%9B%E5%88%B0DDD%E7%9A%84%E5%AE%9E%E6%88%98/image-20200601171228009.png)

简单描述下上面的流程,这个算是很标准了

用户发起http请求 => 调用Controller层 => 调用Service层 => 调用Dao层或外部服务

结果返回过程: Dao层(DO)||外部服务(DTO) => Service层(BO) => Controller层(VO)

其实单看这个还是很清晰,真实的生产现状是很残酷的,主要列举了以下一些问题

- 总有人纠结Java web里面Controller层和Service层哪一层薄一点,哪一层厚一点;那么在这里我告诉你,肯定是Controller层;
- 什么DO直接暴露在Controller层,主键id封装在VO层
- 还有VO里面套DTO、DO等问题

上面列举了几种主要是跟架构相关的问题,其实如果能按照MVC的标准来构造代码,前期还是很清晰的;如果连这个都做不到(当然该生产环境的系统现状很糟糕),转向后期的DDD改造岂不是复杂度更高,聊完了问题,我们就以上述流程的创建订单接口来构思一下代码

#### 创建订单接口

套一下上面的架构流程的公式

用户发起createOrder http请求 => OrderController下面有个createOrder方法 => 调用Service里面的createOrder方法 => (调用外部服务获取相关信息或校验用户和商品信息) => 调用数据库 

##### Controller层

> 部分实现通过TODO注释说明,后面的代码也类似,后续不再说明

- 1.入参校验
- 2.入参转换
- 3.调用Service
- 4.BO出参转VO

```java
public class OrderController {
    @Resource
    private OrderService orderService;
		 /**
     * 创建订单api
     * @param createOrderParam createOrderParam
     * @return OrderVO
     */
    public Result<OrderVO> createOrder(CreateOrderParam createOrderParam) {
        // TODO 入参校验 
        // TODO 将CreateOrderParam转成CreateRequestDTO 这里直接new
        CreateRequestDTO createRequestDTO = new CreateRequestDTO();
      	// 创建订单
        OrderBO orderBO = orderService.createOrder(createRequestDTO);
        // TODO 将OrderBO转成OrderVO
        return Result.success(new OrderVO());
    }
}
```

##### Service层

- 1.参数校验
- 2.外部服务调用&check
- 3.操作数据库
- 4.整合返回结果

> ```java
> @Service
> public class OrderServiceImpl implements OrderService {
>     @Resource
>     private UserCenterAdapter userCenterAdapter;
>     @Resource
>     private GoodsCenterAdapter goodsCenterAdapter;
>     @Resource
>     private OrderDao orderDao;
> 
>     @Override
>     public OrderBO createOrder(CreateRequestDTO createRequestDTO) {
>         // TODO 参数校验
>         // TODO 调用用户中心查询用户信息 & check
>         UserResponseDTO userResponseDTO = userCenterAdapter.queryUser(new QueryUserInfoRequestDTO());
>         // TODO 调用商品中心查询商品信息 & check
>         goodsCenterAdapter.queryGoodsInfo(new QueryGoodsInfoRequestDTO());
>         // TODO 创建订单
>         orderDao.insertOrUpdate(new OrderDO());
>         // TODO 整合DO&DTO返回订单BO
>         return new OrderBO();
>     }
> }
> ```

## MVC存在的一些问题

- 其实前期的话使用这种模式快速迭代,代码也比较清晰
- 但是随着业务复杂度越来越高,你会发现你的OrderService什么方法都有,甚至有什么QueryUserInfo的方法(当然做的好的更定会另新建一个UserService去提供这个方法)
- 可能到最后你会发现ServiceImpl包揽了一切,它无所不能,它能下单,它能查询订单详情,它妈的还能查询用户信息和更新用户信息
- 随着各种各样同学的介入开发,你会发现系统越来越乱,无穷无尽的hotfix

# 拥抱DDD

## 小插曲

> 在正式聊DDD相关的东西之前,我在这顺便提一下我之前遇到的问题,也是该生产环境的一个设计问题

首先车嘛,既然有新车就会有二手车,当时我有个需求改造,要取车辆信息的费用;当时设计的时候是用了两张表,一个新车费用表,一个二手车费用表;当时有个车辆的类型叫xx二手车,你觉得这个类型的车是从新车费用表去获取还是二手车费用表获取呢?正常人都会认为是二手车费用表,事后有同事跟我说这块的逻辑取的有问题,还好我当时机智,因为我也是半道接的需求,我哪里知道有多少类型的车,我的直觉也应该取二手车费用表的字段,列下我当时写的取数逻辑

```java
    public Long getAmt() {
        if (null != orderFeeDO) {
            return orderFeeDO.getAmount();
        }
        if (null != orderSecondhandFeeDO) {
            return orderSecondhandFeeDO.getAmount();
        }
        // throw ...
    }
```

真的是坑啊,还好我是判断新车费用表是否空,不然我凉了;因为在这之前做过相关系统DDD的改造,就联想到,费用既然做为车辆信息领域的一部分,如果有个CarInfoEntity那该多好,那么后续使用的人就根本不需要care里面的数据处理了,直接拿来用了,

```java
public class CarInfoEntity {
    /**
     * 获取车辆费用
     *
     * @param newCarFeeDO    新车费用
     * @param secondCarFeeDO 二手车费用
     * @return 车辆费用
     */
    public Long getCarAmount(NewCarFeeDO newCarFeeDO, SecondCarFeeDO secondCarFeeDO) {
        if (null != newCarFeeDO) {
            return newCarFeeDO.getAmount();
        }
        if (null != secondCarFeeDO) {
            return secondCarFeeDO.getAmount();
        }
        // throw ...
    }
}
```

那么接下来,我们正式聊一下DDD

## DDD的一些核心概念

> 