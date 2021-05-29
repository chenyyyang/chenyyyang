# TODO：整理DDD相关的、问答形式的面试题
# 前言：
看了  [阿里殷浩写的DDD相关的分享](https://mp.weixin.qq.com/s/SjU1DbsXcBD-2DJt9z65zg)  
受益匪浅，他的文章使DDD不再虚无缥缈，再结合平时写代码时候看到的代码坏味道，总结下DDD代码规范，归根到底DDD还是实践了软件设计的软件设计七大原则

 软件设计七大原则
 1、 开闭原则  
 2、 依赖倒置原则  
 3、 单一职责原则  
 4、 接口隔离原则  
 5、 迪米特法则  
 6、 里氏替换原则  
 7、 合成复用原则  
  
基本就是这样的  
**首先是interface层的封装**，我理解也是openAPI层，将http，mq，rpc等不同协议执行方法的入口封装再这一层，这一层也多数是由框架来实现，我们可能只要传入函数，或者使用框架提供的注解。  
**接着是Application层**ApplicationService应用服务：最核心的类，负责业务流程的编排，但本身不负责任何业务逻辑。这一层输入是CQE，返回是DTO或者直接throw RuntimeException，用@Valid注解校验参数。AppService通常不做任何决策（Precondition除外），仅仅是把所有决策交给DomainService或Entity，把跟外部交互的交给Infrastructure接口，如Repository或防腐层。 
```

@Service
public class CheckoutServiceImpl implements CheckoutService {

    @Resource
    private ItemFacade itemFacade;
    @Resource
    private InventoryFacade inventoryFacade;
    
    @Override
    public OrderDTO checkout(@Valid CheckoutCommand cmd) {
        ItemDTO item = itemFacade.getItem(cmd.getItemId());
        if (item == null) {
            throw new IllegalArgumentException("Item not found");
        }

        boolean withholdSuccess = inventoryFacade.withhold(cmd.getItemId(), cmd.getQuantity());
        if (!withholdSuccess) {
            throw new IllegalArgumentException("Inventory not enough");
        }

        // ...
    }
}
```
**DomainService层** 通常有分支逻辑的，都代表一些业务判断，应该将逻辑封装到DomainService或者Entity里。
```
@Data
public class Order {

    private Long itemUnitPrice;
    private Integer count;

    // 把原来一个在ApplicationService的计算迁移到Entity里
    public Long getTotalCost() {
        return itemUnitPrice * count;
    }
}
```
**Anti-Corruption Layer防腐层** 把rpc调用其他服务service的api再包一层facade。
```
@Service
public class InventoryFacadeImpl implements InventoryFacade {

    @Resource
    private ExternalInventoryService externalInventoryService;

    @Override
    public boolean withhold(Long itemId, Integer quantity) {
        return externalInventoryService.withhold(itemId, quantity);
    }
}
```
Repository可以认为是一种特殊的ACL，屏蔽了具体数据操作的细节，即使底层数据库结构变更，数据库类型变更，或者加入其他的持久化方式，Repository的接口保持稳定，ApplicationService就能保持不变。

### 同步调用还是事件驱动，怎么选择呢
从调用链路来看：  
Orchestration：是从一个服务主动调用另一个服务，所以是Command-Driven指令驱动的。  
Choreography：是每个服务被动的被外部事件触发，所以是Event-Driven事件驱动的。  

无论O&C如何设计，Application层都“无感知”，因为ApplicationService天生就可以处理Command、Query和Event，至于这些对象怎么来，是Interface层的决策。

### 总结

Interface层：  
职责：主要负责承接网络协议的转化、Session管理等。  
接口数量：避免所谓的统一API，不必人为限制接口类的数量，每个/每类业务对应一套接口即可，接口参数应该符合业务需求，避免大而全的入参。  
接口出参：统一返回Result。  
异常处理：应该捕捉所有异常，避免异常信息的泄漏。可以通过AOP统一处理，避免代码里有大量重复代码。

Application层：  
入参：具像化Command、Query、Event对象作为ApplicationService的入参，唯一可以的例外是单ID查询的场景。  
CQE的语意化：CQE对象有语意，不同用例之间语意不同，即使参数一样也要避免复用。  
入参校验：基础校验通过Bean Validation api解决。Spring Validation自带Validation的AOP，也可以自己写AOP。  
出参：统一返回DTO，而不是Entity或DO。  
DTO转化：用DTO Assembler负责Entity/VO到DTO的转化。 
异常处理：不统一捕捉异常，可以随意抛异常。

部分Infra层：  
用ACL防腐层将外部依赖转化为内部代码，隔离外部的影响。

DDD这个一开始学可能感觉有点虚，随着手里负责项目的逻辑越来越多且混乱，代码要重构的时候就觉得还挺有用，DDD让代码逻辑更清楚，代码可读性更高。

