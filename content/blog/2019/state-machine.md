---
title: '【review-项目】售前 CRM 项目中 有限状态机的使用，及操作日志的实现'
date: 2019-02-16 16:58:02
updated: 2019-02-16 16:57:57
categories: 
- review
tags: 
- CRM
- FSM
- parser
- 切面
description: 使用 有限状态机 管理客户的状态变更。同时，使用 AOP 在操作状态变更的时候，进行日志记录。
---

> 使用 有限状态机 管理客户的状态变更。同时，使用 AOP 在操作状态变更的时候，进行日志记录。

<!--more-->

# 状态机
## 背景
需求：客户的状态和变更操作较多，需要方便清晰地

方案：使用 `有限状态机` 管理客户的状态。

将所有的状态转移 注册 到一处，而不是散落在一个或多个service。
也方便进行统一地进行`异常处理`，`日志记录`。

新增需求时，开发人员必须定义相应的 `状态`，`操作`，`业务方法`，并注册到状态机中。

同时，这个状态机还用来和需求方进行确认，帮助需求方梳理业务逻辑（公司没有专职产品经理）。
拆分成单个的状态和操作后，讨论问题的粒度更小，边界更清晰。

![](https://user-images.githubusercontent.com/5629307/52897079-f865e580-320a-11e9-9b4a-52cb5ac13570.png)
调研了 一些开源库后，决定造个轮子，
1. `spring-statemachine` 的功能较多，使用较为复杂。而项目要求的状态变更比较简单。
2. 可能会有定制的需求。

状态机注册参照 `spring-statemachine` 的 API 进行实现。
核心概念：`状态`，`操作`，`业务方法`。
## 使用
### 1. 定义状态
```java
public enum CustomerState {
    LEADS_INITAL(10),// 线索，未领用
    LEADS_HOLD(11),// 线索，已领用
    LEADS_LOST(12),// 线索，已流失

    TARGET_INITIAL(20),// 意向客户，未领用
    TARGET_HOLD(21),// 意向客户，已认领
    TARGET_LOST(22), // 意向客户，已流失

    FORMAL_HOLD(31), // 正式客户，已认领
    FORMAL_LOST(32), // 正式客户，已流失
}
```
### 2. 定义操作
```java
public enum Event {
    PICK,// 领用
    LOST,// 流失
    DELIVER,// 转交
    MARK_TARGET, // 更新为意向客户
    MARK_FORMAL, // 更新为正式客户
    MARK_SECONDARY_SALES; // 更新为二次销售客户
}
```
### 3. 注册 状态转移
1. 注册状态转移规则
2. 实现回调方法

spring bean 初始化 方法 执行顺序：
> Constructor > @PostConstruct > InitializingBean#afterPropertiesSet() > (xml) init-method

```java
// net.zhijia.crm.ps.service.CustomerStateMachine

// 实际保存 所有状态转移规则的地方。
private FsmBuilder<CustomerState, Event> fsm;

@PostConstruct
public void init() throws Exception {
    fsm = new FsmBuilder<>();
    fsm.configureTrans()
            ...
            /**
             * 流失
             * 意向客户：领用 => 意向客户：流失
             */
            .and()
            .source(CustomerState.TARGET_INITIAL)
            .target(CustomerState.TARGET_HOLD)
            .event(Event.PICK)
            .action(customerStateMachine::pick, errorAction)
            ;
    fsm.build();
}

@Transactional(rollbackFor = Exception.class)
public void pick(StateRequest stateRequest) {
    PsCrmCustomer psCrmCustomer = stateRequest.getPsCrmCustomer();
    CustomerState targetState = stateRequest.getTargetState();
    psCrmCustomer.setState(targetState);
    psCrmCustomer.stepPickCount();
    psCrmCustomer.setUserId(stateRequest.getTargetUserId());
    psCrmCustomer.setUpdateTime(new Date());
    Integer result = psCrmCustomerRepo.pickWithCheck(psCrmCustomer);
    Long customerId = psCrmCustomer.getId();
    String userId = psCrmCustomer.getUserId();
    followPlanService.init(customerId, userId);
}
```

### 4. 业务调用
核心方法是 `stateMachine.sendEvent(stateRequest)`

业务调用方 构造 请求体，放入必需的信息：
- 当前状态
- 要进行的操作
- 回调方法需要的参数

两个例子：
1. 转让 线索给 其他商务
根据状态机，只有`线索-已认领`和 `意向客户-已认领`两个状态才能执行 `转让`操作。
2. 转为意向客户
只有 `线索-已认领`状态，才可执行 `转为意向客户`操作。
```java
@LogOp(
    title = "转让给({targetUserId}){targetUserName}",
    withVal = true, 
    type = ActionType.DELIVER)
public void deliver(
    @CustomerId Long customerId,
    Customer customer,
    @TargetUserId @LogVar("targetUserId") String targetUserId,
    @LogVar("targetUserName") String targetUserName,
    @CusState CustomerState state,
    @LogContent String content
) {
    StateRequest stateRequest = StateRequest.builder()
            .psCrmCustomer(psCrmCustomer)
            .targetUserId(targetUserId)
            .targetUserName(targetUserName)
            .event(Event.DELIVER)
            .build();
    boolean result = stateMachine.sendEvent(stateRequest);
}

@LogOp(title = "转为意向客户", type = ActionType.MARK_TARGET)
public void markTarget(
    @CustomerId Long customerId, 
    String userId, 
    @LogContent String content) {
    PsCrmCustomer psCrmCustomer = psCrmCustomerRepo.findWithAuth(customerId, userId);
    /**
     * 更新客户状态
     */
    StateRequest stateRequest = StateRequest.builder()
            .psCrmCustomer(psCrmCustomer)
            .event(Event.MARK_TARGET)
            .build();
    boolean result = stateMachine.sendEvent(stateRequest);
}
```
## 实现
```
fsm
  ./FsmAction
  ./FsmBuilder
  ./FsmErrorAction
  ./FsmTrans
  ./FsmTransBuilder
```
### 状态机
状态机 其实是一个嵌套的 map。

第一层 map，key 是 源状态，value 是 多个转移路径的集合

第二层 map，key 是 操作，map 是 目标状态 和 回调方法。
```java
public class FsmBuilder<S, E> {
    private FsmTransBuilder<S, E> transBuilder;
    class Target<S, E> {
        private S target;
        private FsmAction<S, E> action;
        private FsmAction<S, E> errorAction;
    }

    /**
     * structure:
     * * sourceState : {
     * *                event : {  targetState,
     * *                           targetAction,
     * *                         },
     * *                ...
     * *              }
     */
    private Map<S, Map<E, Target<S, E>>> lookupMap;
}
```
状态机的 build: 

把 transList 转换成 lookupMap，方便查询。

**todo** build 的时候，直接塞入 lookupMap？
```py
loop transList as trans: 
    lookupMap.put(
        {trans.source:
            (
                trans.target,
                trans.action,
                trans.errotAction,
                )
        }
    )
```
实际代码：
```java
    public void build() {
        List<FsmTrans<S, E>> transList = transBuilder.build();
        Map<S, Map<E, Target<S, E>>> lookupMap = new HashMap<>(transList.size());

        for (FsmTrans<S, E> trans : transList) {
            S source = trans.getSource();
            Map<E, Target<S, E>> targetMap = lookupMap.get(source);
            if (targetMap == null) {
                targetMap = new HashMap<>(5);
            }

            E event = trans.getEvent();
            if (targetMap.get(event) != null) {
                String msg = MessageFormat.format("event[{0}] on source[{1}] is existed", event, source);
                throw new FsmException(msg);
            }

            S targetState = trans.getTarget();
            FsmAction<S, E> action = trans.getAction();
            FsmAction<S, E> errorAction = trans.getErrorAction();
            Target<S, E> target = new Target<>(targetState, action, errorAction);
            targetMap.put(event, target);

            lookupMap.put(source, targetMap);
        }
        this.lookupMap = lookupMap;
    }
```
### 请求体
```java
public class StateRequest {
    /**
     * 追踪调用链
     */
    private String uId;
    private Long customerId;
    // 必填 
    // 携带 sourceState
    private PsCrmCustomer psCrmCustomer;
    // 必填
    private Event event;
    /**
     * 目标 销售ID
     * 用于 转让 操作
     */
    private String targetUserId;
    private String targetUserName;

    private CustomerState targetState;
}
```
```java
// net.zhijia.crm.ps.service.CustomerStateMachine#sendEvent
public boolean sendEvent(StateRequest stateRequest) {
    CustomerState curState = stateRequest.getPsCrmCustomer().getState();
    Event event = stateRequest.getEvent();
    // todo 移出 FSM 移到这儿
    return fsm.sendEvent(curState, event, stateRequest);
}
```
TODO: 这里应该把`业务相关的异常处理`放到 状态机之外，或者以方法的形式传入 状态机。类似 `ExceptionHandler`
```java
public class FsmBuilder<S, E> {
    public boolean sendEvent(S curState, E event, StateRequest stateRequest) {
        Map<E, Target<S, E>> targetMap = lookupMap.get(curState);
        if (targetMap == null) {
            throw new FsmException("no matched targetMap,curState=" + curState);
        }
        Target<S, E> target = targetMap.get(event);
        if (target == null) {
            log.info("no matched target,curState={}, event={}", curState, event);

            // todo 移出 FSM，在外面捕获 FsmException 处理
            if (Event.PICK.equals(event)) {
                throw new FsmException("该线索已被认领");
            } else if (Event.LOST.equals(event)) {
                // 流失是 幂等操作，不报错
                return true;
            } else {
                throw new FsmException("操作失败，客户当前状态不可执行此操作");
            }
        }
        FsmAction<S, E> action = target.getAction();
        stateRequest.setTargetState((CustomerState) target.getTarget());
        try {
            action.execute(stateRequest);
            return true;
        } catch (Exception e) {
//            e.printStackTrace();
            throw e;
//            return false;
        }
//        return false;
    }

}
```

### 状态转移对象
```java
@Builder
@Data
public class FsmTrans<S, E> {
    private S source;
    private S target;
    private E event;
    private FsmAction<S, E> action;
    private FsmAction<S, E> errorAction;
}
// Action 是一个 Consumer
public interface FsmAction<S, E> {
    void execute(StateRequest context);
}
```


注册状态转移的地方，
`FsmBuilder.configureTrans()` 创建了一个 `FsmTransBuilder`

`FsmTransBuilder` 是一个 Facade，代理了 `FsmTrans.builder` 的方法：
- `and()`: 把当前的 trans 保存起来，创建一个新的 transBuilder
- `build()`: 把当前的（最后一个） trans 保存起来，返回 状态转移的集合。
```java
public class FsmTransBuilder<S, E> {

    private List<FsmTrans<S, E>> transList = new ArrayList<>();

    private FsmTrans.FsmTransBuilder<S, E> builder;

    public FsmTransBuilder<S, E> and() {
        if (builder != null) {
            FsmTrans<S, E> trans = builder.build();
            transList.add(trans);
        }
        builder = FsmTrans.builder();
        return this;
    }
    public List<FsmTrans<S, E>> build() {
        if (builder != null) {
            FsmTrans<S, E> trans = builder.build();
            transList.add(trans);
        }
        return transList;
    }
    // 代理  builder.source()
    public FsmTransBuilder<S, E> source(S state) throws Exception {
        checkBuilder();
        builder.source(state);
        return this;
    }
    // target
    // event
    // action
}
```


```java
// 实际保存 所有状态转移规则的地方。
private FsmBuilder<CustomerState, Event> fsm;

@PostConstruct
public void init() throws Exception {
    fsm = new FsmBuilder<>();
    fsm.configureTrans()
            ...
            /**
             * 流失
             * 意向客户：领用 => 意向客户：流失
             */
            .and()
            .source(CustomerState.TARGET_INITIAL)
            .target(CustomerState.TARGET_HOLD)
            .event(Event.PICK)
            .action(customerStateMachine::pick, errorAction)
            ;
    fsm.build();
}
```

```
FSM := []Trans
Trans := [](source, event, target, action)

source := State
target := State
event := Event
action := Function，Consumer

State := enum
Event := enum

S = source
T = (target,action)
E = event
Trans = (S-E-T)
---------------------
       T1        T2
---------------------
S1     E1        E1
---------------------
S2     E2        E3
---------------------
```

# 操作日志
业务需求：记录跟进线索的操作日志。

方案：
在以下方法上，加上 `@LogOp` 注解
- 在所有涉及到线索状态变更的地方，也就是所有调用 `sendEvent` 的地方。
- 添加跟进记录 操作。

![](https://user-images.githubusercontent.com/5629307/52897080-f8fe7c00-320a-11e9-82c4-de8734091532.png)
![](https://user-images.githubusercontent.com/5629307/52897081-f8fe7c00-320a-11e9-822f-30ebc3d9e498.png)
## 使用
```java
// usage
  @LogOp(title = "转移给({targetUserId}){targetUserName}", 
  withVal = true, 
  type = ActionType.DELIVER)
    public void deliver(
            @CustomerId Long customerId,
            PsCrmCustomer psCrmCustomer,
            @TargetUserId @LogVar("targetUserId") String targetUserId,
            @LogVar("targetUserName") String targetUserName,
            @CusState CustomerState state,
            @LogContent String content
    );
```
## 实现
目录结构：
```js
.
├── LogTemplate.java
├── LogToken.java
├── TokenType.java
├── annotation
│   ├── CusState.java // 日志记录字段，当前状态
│   ├── CustomerId.java // 日志记录字段
│   ├── LogContent.java // 日志记录字段，用于跟进记录.详细信息
│   ├── LogOp.java
│   ├── LogUserId.java   // 日志记录字段，当不使用当前登录用户作为 createUser 时，传递这个值，比如 每天凌晨系统流失 长时间未跟进的客户时，传入管理员 ID。
│   ├── LogVar.java     // 模板参数
│   └── TargetUserId.java // 日志记录字段，用于 转移客户时，目标商务 ID
└── aop
    ├── LogAspect.java
    └── LogParser.java
```

日志记录 结构：
```Java
public class ActionLog {
    private Long customerId;
    private ActionType actionType;
    private String title;
    private String content;
    ...
}
```

### 切点
LogOp
```java
@Inherited
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LogOp {
    String title() default "no-title";
    // 是否带参
    boolean withVal() default false;
    // 操作类型，和 fsm.event 差不多，为以后的统计用
    ActionType type() default ActionType.OTHERS;
}
```

### 切面
如果 不带参的话，直接保存 title。
如果 带参的话，获取对应的模板，传入参数，进行 `render`。
```java
// net.zhijia.common.log.aop.LogAspect#saveActionLog
 boolean withVal = logOp.withVal();
 if (!withVal) {
     builder.title(logOp.title());
 } else {
     String methodRef = method.toString();
     // 按照方法签名 匹配 template
     LogTemplate template = findTemplate(logOp, methodRef);
     // log var map
     Map<String, Object> varMap = parseLogVariable(method, args);

     String title = template.render(varMap);
     builder.title(title);
 }
```


### 日志模板
#### 缓存
模板使用 map 做缓存，以 方法签名作为 key。
```java
/**
 * key: method.toString()
 * value:
 */
private Map<String, LogTemplate> logTempCache = new HashMap<>();

private LogTemplate findTemplate(LogOp logOp, String methodRef) {
    LogTemplate template = logTempCache.get(methodRef);
    if (template == null) {
        log.debug("parse LogOp,methodRef={}", methodRef);
        template = LogParser.parse(logOp.title());
        logTempCache.put(methodRef, template);
    }
    return template;
}
```
#### 解析
实现了一个简单的自顶向下解析器。
只有两种 token：text 和 val。
val 从 方法的 `@LogVal` 参数中获取。

```java
public class LogParser {
    /**
     * log: token*
     * token: text | val
     * text: word
     * val: { word }
     * word: (^[{}])+
     *
     * @param logStr
     * @return
     */
    public static LogTemplate parse(String logStr) {
        LogTemplate temp = new LogTemplate();

        char[] chars = logStr.toCharArray();
        StringBuilder lookup = new StringBuilder();
        for (char c : chars) {
            switch (c) {
                case '{':
                    temp.consume(TokenType.TEXT, lookup);
                    lookup = new StringBuilder();
                    break;
                case '}':
                    temp.consume(TokenType.VAL, lookup);
                    lookup = new StringBuilder();
                    break;
                default:
                    lookup.append(c);
                    break;
            }
        }
        temp.consume(TokenType.TEXT, lookup);
        return temp;
    }
}
// token
public class LogToken {
    private TokenType type;
    private String key;
    private String value;
}
// tokenType
public enum TokenType {
    /**
     * 普通文本
     */
    TEXT,
    /**
     * 变量
     */
    VAL
}
```
## 其他
后续又增加了
- 操作前状态
- 每个操作都要带有备注（也就是跟进内容）
- 每次操作，都要修改对应客户的意向度，并记录到操作日志中
- targetUserId 也要入 操作日志 （比如转让操作，之前是嵌入文本的，改成单独字段保存是统计的需要，统计报表需要每个商务每日转让的进出情况）

针对 方法的 注解参数 增加的情况，
增加了 `findArgByAnnotation` 方法，
以及，增加了很多代理方法，用于转换 前台请求，对象查询 以及 操作日志方法所需要的细粒度参数的 映射。
这是一个现实倒逼的方法，4+N 个注解参数，已经不太方便了。

# 总结
## 线程安全问题分析
### 状态机 注册
安全初始化后，状态机是 `事实不可变` 的，不需要考虑线程安全问题。
只提供了初始化接口，没有提供修改接口。

### 操作日志模板 缓存
使用了普通的 hashMap 存储。
因为是幂等操作，即使重复解析模板也不影响业务，不需要考虑线程安全问题。

# log
2018-07-12
init 
- 组内分享

2019-02-15
- 图片挂到 github-wiki
- 整理补充，发布。


# refer
- [spring-statemachine 实例-打车服务-曹祖鹏](https://github.com/JoeCao/qbike/blob/master/order/src/main/java/club/newtech/qbike/order/domain/service/OrderStateMachineBuilder.java)
- [spring-statemachine](https://github.com/spring-projects/spring-statemachine)