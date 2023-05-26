#  Solution of distributed transaction by ICPSwap

## Summary
When we design a Dapp, we usually face a problem, which architecture is better, single canister mode or multiple canister mode.

Single canister mode has good stability, but its processing capacity is limited. However, most systems have a certain degree of complexity, and the design of single canister is difficult to meet the needs of large systems, so most systems will use the design of multiple canister architecture.

Similar to distributed systems, multi-canister systems may have data consistency problems. We plan to solve these problems from the following aspects:
- Use distributed locks to solve atomicity problems
- Use Saga state machine as transaction coordinator to coordinate transactions of multiple canisters
- Use message queue to achieve eventual consistency

## Atomicity

### Question
Typically, ICP network has a feature that 'One canister processes its messages one-at-a-time' and prevents race conditions, so that the request is atomic. But when we use the `await` keyword, it will allow other requests to enter, which will destroy atomicity.

```rust
var data: HashMap.HashMap<Text, Data> = HashMap.HashMap<Text, Data>(0, Text.hash, Text.equal);

public shared(msg) func test() : async Bool {
    let d : ?Data = data.get("test");
    if (d == null) {
        //do something.
        let c = actor("canister id"): Test;
        await c.test();

        data.put("test", d);
        return true;
    };
    return false;
};
```

In the real scenario, when more than one request reaches the canister side at the same time, the code of `await c.test()` releases current thread, and wakes other requests up to execute the codes before `await c.test()`, and then the judging condition of `if (d == null)` will be invalid.

### Solution
We plan to use distributed locks to solve this problem. 

Distributed locks have the following features:
- In a multi-canister environment, a lock can only be obtained by one request of a canister at a time
- Highly available acquire and release locks
- High performance acquire and release locks
- With lock failure mechanism, including automatic unlocking and deadlock prevention
- With blocking feature, it automatically waits when the lock is not obtained, and automatically unlocks when the waiting time expires

[ic-lock](https://github.com/ICPSwap-Labs/ic-lock)

## Saga

Saga is a mode often used to deal with complex business and distributed transaction in distributed environment. Each Saga consists of a series of sub-transaction (Ti), and each Ti has a corresponding compensation action Ci, which is used to undo the result caused by Ti.

![saga-demo](https://github.com/ICPSwap-Labs/consistency/blob/main/ic-saga-demo.png)

There are two common types of Saga: Choreography and Orchestration. 

Choreography-based Saga completely leaves the direction and coordination of the process to the developer of the transaction. The developer uses hard coding to realize Saga's transaction and transaction compensation. In this mode, although each sub-transaction is independent, there is no overall transaction coordinator, so the more service participants, the worse the understandability of the business process, and it is also difficult to locate problems.

Orchestration-based Saga gives the execution order of transactions to a centralized Saga orchestrator, which directs and decides the flow of business. The whole process in this mode is much clearer than Choreography-based Saga.

![saga-workflow](https://github.com/ICPSwap-Labs/consistency/blob/main/ic-saga-workflow.png)  

We plan to use a state machine as the Saga coordinator to implement the transaction choreography function, and define the business process.

![saga](https://github.com/ICPSwap-Labs/consistency/blob/main/ic-saga.png)

In the design of Saga coordinator, it is mainly divided into terminal layer, proxy layer and service layer.
1. Terminal layer   
The terminal layer is mainly for application users and process design managers to use. Application users can access the Saga coordinator directly with using the terminal program. The process design managers can design and manage Saga process through the process design page.

2. Proxy layer  
The proxy layer is mainly responsible for the construction of related data of service invocation, the management of server connection information, and the selection and scheduling of server interface.

3. Service layer  
The service layer is the core of the Saga coordinator and is used to provide services externally. Internally, it provides open API, state data manager, runtime data manager, process scheduler, retry mechanism, log management and other functions.

[ic-saga](https://github.com/ICPSwap-Labs/ic-saga)

## Eventual consistency

After a certain period of time, all data copies in the system can finally reach the consistent state that conforms to the service definition. Therefore, there is no need to ensure the strong consistency of system data in real time. Eventual consistency is a special case of weak consistency.

We can use message queues to achieve eventual consistency across multi-canister systems, and in this process, we face several problems:
- In the message sender side, the atomicity of performing local transactions and sending messages. Ensure that the message will be successfully sent when the local transaction is executed successfully
- In the message receiver side, the atomicity of performing local transactions and receiving messages. Ensure that the local transaction will be executed successfully after the message is received successfully
- Prevents repeated consumption of messages when the messages are sent repeatedly

![queue](https://github.com/ICPSwap-Labs/consistency/blob/main/ic-queue.png)

In our design, an internal queue is used to store messages, so as to ensure that messages are saved successfully. A scheduled task is used to synchronize local messages to the message queue periodically, and resend messages when sending fails.

The following components are mainly included in the design of message queue:

1. Exchanger  
The main function of the exchanger is to transfer received messages and data to the corresponding topic queue according to different topics.

2. Queue  
The queue holds the actual message data, and each topic has its own separate queue.

3. GroupManager  
GroupManager mainly maintains subscription information for consumer groups. The same group of consumers will consume the data in the queue in turn. The data which has been processed will not be consumed again by other consumers in the same group. Different groups of consumers consume data independently of each other in the queue.

4. TopicManager  
TopicManager maintains a list of pushed and subscribed topics and creates a separate queue for each topic. When a topic has been without subscribers for a long time and there is no data in the queue, the topic's queue is reclaimed by TopicManager.

5. QueueManager  
The QueueManager is responsible for keeping the data in the queue ready for consumption. When different groups subscribe to the same topic, the QueueManager ensures that data can be consumed by all consumers before removing it from the queue.

6. LoggerManager  
The LoggerManager records all operation logs in the message queue, including push and consumption data, for data recovery and troubleshooting.

[ic-message-service](https://github.com/ICPSwap-Labs/ic-message-service)

