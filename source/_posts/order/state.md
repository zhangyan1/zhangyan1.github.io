---
title: 订单之状态机
date: 2019-09-22 
categories: 设计
tags: [订单]
keywords: 订单,状态机,源码
---

### 意义

- 订单系统各种订单类型的流转，触发事件、执行事件如果不利用状态机代码会变得更加复杂且难以维护
- 通过状态机,能够把状态流转的概念抽象出来,更符合java编程思想的规范
- 状态机能够反映时序的布进控制

<!--more--> 

### 基础概念

- 网络上关于状态机的定义有非常多的理论解释,在这里笔者仅仅按照自己的理解尽可能的简要描述
- 例如订单支付 订单由待支付状态(现态),执行了支付(事件),订单进行了状态更新(动作)最终变为已支付的状态(次态)就是一个完整的状态机流程

### 简要设计

	简要回顾上一篇文章定义的几个初始状态
 ![state](/images/orderImage/state.png)

- 图中绿色代表订单的终态,不同类型的订单系统终态可能不一致,例如商城类订单已退款就可能不能再继续操作。商旅类订单退款之后还能继续再往下走,这些场景都需要根据实际业务场景来设计订单的终态部分。
- 其中每一步的事件流向也需要系统设计者因地制宜,笔者建议结合订单子状态对一些模糊状态进行区分,例如订单取消分为主动取消和被动取消,订单退款分为部分退款和全额退款等等

### 代码设计
- 秉着不重复造轮子的原则,笔者之前订单系统主要借鉴了hadoop yarn状态机的设计,并进行了一点小小的优化和封装
- yarn状态机多弧过渡部分因为没有用到直接省略(感兴趣的同学可以自己去查看源码,笔者认为多弧过渡主要是次态的可选择性,但是反而增加了不确定性所以没有采用)接下来的部分主要是源码解析

- 状态机主要类图如下

- hadoop底层代码
```java

//此方法在addTransition方法中调用 主要是构建链表
private StateMachineFactory
      (StateMachineFactory<OPERAND, STATE, EVENTTYPE, EVENT> that,
       ApplicableTransition<OPERAND, STATE, EVENTTYPE, EVENT> t) {
    this.defaultInitialState = that.defaultInitialState;
    this.transitionsListNode 
        = new TransitionsListNode(t, that.transitionsListNode);
    this.optimized = false;
    this.stateMachineTable = null;
}
//installTopology 中调用进行初始化过程 主要逻辑封装在makeStateMachineTable()方法中
private StateMachineFactory
      (StateMachineFactory<OPERAND, STATE, EVENTTYPE, EVENT> that,
       boolean optimized) {
    this.defaultInitialState = that.defaultInitialState;
    this.transitionsListNode = that.transitionsListNode;
    this.optimized = optimized;
    if (optimized) {
      makeStateMachineTable();
    } else {
      stateMachineTable = null;
    }
}
  
//
private void makeStateMachineTable() {

//创建堆栈 ApplicableTransition将此前链表的各个ApplicableTransition压入栈中
    Stack<ApplicableTransition<OPERAND, STATE, EVENTTYPE, EVENT>> stack =
      new Stack<ApplicableTransition<OPERAND, STATE, EVENTTYPE, EVENT>>();

    Map<STATE, Map<EVENTTYPE, Transition<OPERAND, STATE, EVENTTYPE, EVENT>>>
      prototype = new HashMap<STATE, Map<EVENTTYPE, Transition<OPERAND, STATE, EVENTTYPE, EVENT>>>();

    prototype.put(defaultInitialState, null);//默认状态

    // 构建拓扑表 每个状态都有一个对应的
    stateMachineTable
       = new EnumMap<STATE, Map<EVENTTYPE,
                           Transition<OPERAND, STATE, EVENTTYPE, EVENT>>>(prototype);

    for (TransitionsListNode cursor = transitionsListNode;
         cursor != null;
         cursor = cursor.next) {
      stack.push(cursor.transition);
    }
 //apply方法就是完整构建拓扑表过程 此处简要描述 在拓扑表中找到当前订单状态的动作映射map,然后把动作放入此map中
    while (!stack.isEmpty()) {
      stack.pop().apply(this);
     
    }
  }
```

- 应用层面代码
  ```java
//创建一个初始状态机
private static final StateMachineFactory<OrderRequest, OrderStatusEnum, OrderEvent.OrderEventEnum, OrderEvent> stateMachineFactory = new StateMachineFactory<OrderRequest, OrderStatusEnum, OrderEvent.OrderEventEnum, OrderEvent>(OrderStatusEnum.INVALID)
//调用私有构造方法 添加单弧事件去transitionsListNode
.addTransition(OrderStatusEnum.INVALID, OrderStatusEnum.SUBMIT_ORDER, OrderEvent.OrderEventEnum.INIT_SUCCESS, new OrderInitTransition()) 
//构建拓扑表
.installTopology();
  ```

- 执行过程代码

  ```
  //将自定义request参数传入状态机,根据传入status为默认状态返回一个状态机。
  //如果拓扑表为构建会重新执行一遍构建过程
  orderStateMachine = OrderStateMachine.getStateMachine(request, OrderStatusEnum.getById(request.getOrderStatus()));
  
  //状态转移过程 返回次态
  orderStateMachine.doTransition(orderEvent.getOrderEvent(), orderEvent)
  
  //具体doTranstion方法
  private STATE doTransition
             (OPERAND operand, STATE oldState, EVENTTYPE eventType, EVENT event)
        throws InvalidStateTransitionException {
      //根据订单状态找到之前构建的map
      Map<EVENTTYPE, Transition<OPERAND, STATE, EVENTTYPE, EVENT>> transitionMap
        = stateMachineTable.get(oldState);
      if (transitionMap != null) {
      //根据事件找到具体动作
        Transition<OPERAND, STATE, EVENTTYPE, EVENT> transition
            = transitionMap.get(eventType);
        if (transition != null) {
        	//执行动作
          return transition.doTransition(operand, oldState, event, eventType);
        }
      }
      throw new InvalidStateTransitionException(oldState, eventType);
    }
  
  ```



- 主要核心代码大概就是上面这些,笔者主要是抛弃了多弧过渡,订单系统中可以优化单弧过渡执行方法时直接将次态带入自定义的transition 这样可以将同类型的多个transition 合并