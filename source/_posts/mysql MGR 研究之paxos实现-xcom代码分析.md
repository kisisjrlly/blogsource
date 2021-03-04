---
title: Mysql Mgr模块研究之孟子协议实现：xcom模块
categories:
- distributed system
tags: 
- system
- distribute
- fault tolerance
---

xcom是MGR中实现一致性协议的核心模块，借鉴Mencius算法的思想实现了一套类paxos协议。xcom首先通过GCS接受到MGR封装的事务消息，然后后通过paxos协议在各个节点上达成一致，最终发送再发送给上层MGR。因此MGR中与xcom相关的模块主要包括与GCS的交互接口，xcom的核心模块paxos实现。在xcom中，首先实现了一套协程机制，xcom所有过程都使用这套协程机制。因此，本文主要介绍一下与上层MGR的交互过程，paxos协议的实现，协程机制的实现，最后看一下paxos各个过程如何通过这套协程实现。



## 1. 与上层MGR的交互

MGR中的事务以Paxos请求的方式发送给xcom，Paxos通过两阶段协议（propose、accept）或者三阶段的(prepare、propose、accept)方式使各节点达成一致后返回给MGR在进行后续处理。


在Gcs_xcom_communication::send_message()接口中会将消息类型设置为Gcs_internal_message_header::CT_USER_DATA，交由 Gcs_xcom_proxy_impl::xcom_client_send_data()发送。

xcom_client_send_data 将事务消息放入m_xcom_input_queue(无锁MPSC队列)中，然后通过与xcom的socket连接，通知xcom模块有消息进入队列。

```
bool Gcs_xcom_proxy_impl::xcom_client_send_data(unsigned long long len,
                                                char *data) {
  /* We own data. */
  
  ....
  
  if (len <= std::numeric_limits<unsigned int>::max()) {
    assert(len > 0);
    app_data_ptr msg = new_app_data();
    /* Takes ownership of data. */
    msg = init_app_msg(msg, data, static_cast<uint32_t>(len));
    successful = xcom_input_try_push(msg){
        
        m_xcom_input_queue.push(data);  // 将消息放入MGR向xcom发送消息的缓冲队列
        if (pushed) successful = ::xcom_input_signal(); // 通过tcp通知xcom模块缓冲队列中存入了数据
        return successful;
    }
    
    if (!successful) {
      MYSQL_GCS_LOG_DEBUG("xcom_client_send_data: Failed to push into XCom.");
    }
  }
  ....
}
```
在xcom中，通过本地协程local_server,等待socket请求的到来，然后从m_xcom_input_queue队列中读取消息，调用dispatch_op进行处理，对于op为client_msg的消息，dispatch_op会进一步调用handle_client_msg()插入到prop_input_queue请求channel的末尾。每个MGR节点的Xcom有一个proposer_task，会获取prop_input_queue头部的请求，然后进入paxos的流程。

local_server的主要过程如下：
```
int local_server(task_arg arg) {
    .....
     while (!xcom_shutdown) {
        /* 等待客户端信号，以便可以从m_xcom_input_queue中读取数据. */
        TASK_CALL(task_read(&ep->rfd, ep->buf, 1024, &ep->nr_read));
        ep->request = xcom_try_pop_from_input_cb();
        while (ep->request != NULL) {
            ....
            dispatch_op(NULL, ep->request_pax_msg, &ep->internal_reply_queue);
            ....
        }
    
    }
}
```

xcom中的executor_task会按序获取已经完成Paxos处理流程的事务请求，调用execute_msg()执行该请求，对于app_type类型的请求，会调用deliver_to_app()，该函数最终调用了在MGR初始化时注册的xcom_data_receiver处理函数cb_xcom_receive_data()，发送到上层客户端(GCS)。

下面主要介绍一下事务消息在xcom模块中的处理过程，即paxos的实现。

## 2. paxos核心协议流程(ing)

### Mencius协议回顾


#### basic paxos

![basic paxos](https://img-blog.csdnimg.cn/2020122914594029.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4MjU4OTAz,size_16,color_FFFFFF,t_70)

- multi paxos：集群中设置leader节点，可以跳过prepare阶段(变成两阶段)；leader故障时走basic paxos 协议的过程(三阶段)。
#### 多领导者

![image](https://img-blog.csdnimg.cn/20201102195424666.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4MjU4OTAz,size_16,color_FFFFFF,t_70#pic_center)

#### simple paxos

- coordinator(leader)：在自己负责的日志序列中对应位置，可以提议客户端请求和no-op操作
- 非 coordinator：只能提议no-op

![image](https://img-blog.csdnimg.cn/20201029203916286.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4MjU4OTAz,size_16,color_FFFFFF,t_70#pic_center)

#### 优化
- 将no-op消息piggyback到accept阶段
- 将no-op消息piggyback到未来的propose阶段

 ![image](https://img-blog.csdnimg.cn/20201029203955265.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4MjU4OTAz,size_16,color_FFFFFF,t_70#pic_center)

目前xcom模块已经实现了Mencius算法中的paxos和simple paxos部分，但由于Mencius算法中优化算法依赖于异步的FIFO通信机制(Asynchronous FIFO communication channel)来保证正确性，因此xcom中并没有实现。


### 2.1 主要数据结构
在xcom模块中比较重要的数据结构如下：

struct site_def 描述了一个有时效性的MGR/Paxos集群配置,每个节点维护了一个唯一的site_def对象
```
// 描述了一个有时效性的MGR/Paxos集群配置,每个节点维护了一个site_def对象
struct site_def {
  synode_no start;    /* Config is active from this message number */
  synode_no boot_key; /* The message number of the original unified_boot */
  node_no nodeno;     /* Node number of this node */
  node_list nodes;    /* Set of nodes in this config */
  server *servers[NSERVERS]; /* Connections to other nodes */
  detector_state detected;   /* Time of last incoming message for each node */
  node_no global_node_count; /* Number of live nodes in global_node_set */
  node_set global_node_set;  /* The global view */
  node_set local_node_set;   /* The local view */
  int detector_updated;      /* Has detector state been updated? */
  xcom_proto x_proto;
  synode_no delivered_msg[NSERVERS];
  double install_time;
  xcom_event_horizon event_horizon;
};
```
主要内容包括集群节点生效的开始消息编号，本节点的编号，在当前视图配置下节点的编号，当前视图中节点的集合，与本节点保持连接的节点，每个节点最新的消息发送时间，全局视图中节点的数量及集合，本节点视图中节点的数量及集合，与正常的paxos协议执行相关的字段包括servers和delivered_msg。前者维护了本节点到集群中其他节点的连接，后者表现各个节点的消息完成状态。

经过paxos处理的消息都有一个消息号synode_no。nodeno是节点在Paxos集群中的序号，该序号是Paxos消息synode_no的组成部分，synode_no的另一部分为消息序号，结合在一起synode_no表现为{X, N}，其中X为消息序号，N为节点序号。

```
struct synode_no {
	uint32_t group_id; // 用于判断新加入的节点是否是同一个组，MGR节点间交互通过xcom进行，而MGR规定的不同组不能构成集群的要求，只能在xcom中实现
	uint64_t msgno;
	node_no node;
};

int synode_gt(synode_no x, synode_no y) {
  assert(x.group_id == 0 || y.group_id == 0 || x.group_id == y.group_id);
  return (x.msgno > y.msgno) || (x.msgno == y.msgno && x.node > y.node);
}
```
![image](https://img-blog.csdnimg.cn/20201230114950546.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4MjU4OTAz,size_16,color_FFFFFF,t_70)

在进行prepare、propose、accept和learn等Paxos操作时，需要依赖site_def中的server对象发送消息。

```
/* Server definition */
struct server {
  int garbage;
  int refcnt;
  char *srv;                 /* Server name */
  xcom_port port;            /* Port */
  connection_descriptor con; /* Descriptor for open connection,本节点到指定节点的长连接 */
  double active;             /* Last activity */
  double detected;           /* Last incoming */
  channel outgoing;          /* Outbound messages */
  task_env *sender;          /* The sender task */
  task_env *reply_handler;   /* The reply task */
  srv_buf out_buf;
  int invalid;
  int number_of_pings_received; /* Number of pings received from this server */
  double last_ping_received;    /* Last received ping timestamp */
};
```

跟消息发送相关的字段主要有：srv是server指向节点的名称，con维护了本节点到server指向节点的长连接，outgoing队列用于缓存本节点需发送到server指向节点的消息，sender是在server初始化时注册的sender_task协程，用来负责从outgoing队列中读取消息发送到server指向节点。在Paxos正常运行期间，server对象是集群中节点间消息发送的媒介，通过con建立2个节点间的联系，发送端即为sender字段对应的sender_task协程，接收端就是server对应节点上的acceptor_learner_task协程。需要注意的是，本节点也会为自己创建一个server对象，此时sener字段即为local_sender_task。

server维护了Paxos集群各节点间的通信管道，pax_msg就是管道中传输的消息。其定义如下：
```
struct pax_msg{
  node_no to;             /* To node */
  node_no from;           /* From node */
  uint32_t group_id; /* Unique ID shared by our group */
  synode_no max_synode; /* Gossip about the max real synode */
  start_t start_type; /* Boot or recovery? DEPRECATED */
  ballot reply_to;    /* Reply to which ballot */
  ballot proposal;    /* Proposal number */
  pax_op op;          /* Opcode: prepare, propose, learn, etc */
  synode_no synode;   /* The message number */
  pax_msg_type msg_type; /* normal, noop, or multi_noop */
  bit_set *receivers;
  app_data *a;      /* Payload */
  snapshot *snap;	/* Not used */
  gcs_snapshot *gcs_snap; /* gcs_snapshot if op == gcs_snapshot_op */
  client_reply_code cli_err;
  bool force_delivery; /* Deliver this message even if we do not have majority */
  int32_t refcnt;
  synode_no delivered_msg;
  xcom_event_horizon event_horizon;  
  synode_app_data_array requested_synode_app_data;
 }
```


from和to字段标识了消息的源节点和目标节点，to字段可能为null，表示发往所有节点；op标识消息的操作类型，比如是prepare还是propose消息；synode表示该消息的序号，类型为synode_no，消息组成详见site_def定义；msg_type表示消息类型，是普通消息还是不携带数据的noop消息等；receivers表示有多少节点已经接收到了该消息；a是消息数据，比如对于事务消息，该字段就包含了一系列的log_event。reply_to和proposal是2个ballot，ballot由一个节点序号nodeno和计数器cnt组成，表示某次投票号，显然proposal字段表示本次消息提案的投票号，reply_to表示本次消息是对reply_to对应的提案编号的回复。同个pax_msg的proposal和reply_to字段对应的ballot可能不同，举个例子，A节点向B节点发送prepare消息，B节点收到后，发现本节点已经接受了对应synode的消息，那么B节点在回复A消息时，会将消息中的propsal字段设置为本节点的提案编号并替换掉消息体。ballot是Paxos算法强相关的字段，对于某个确定的synode消息，在prepare阶段，一个节点只能继续处理比其之前收到的所有提案编号更大的提案，在acceptor阶段，一个节点只能处理不小于其之前收到的所有提案编号的提案。

在节点上，pax_msg保存在pax_machine中，在Paxos算法中，pax_machine就是一个Paxos实例，即Mencius算法中的instance，对应一条Paxos日志，一系列有序的Paxos日志组成了Paxos状态机。每个节点维护了一个最小5万个pax_machine对象的缓存（paxos cache），有专门的缓存管理协程cache_manager_task负责维持缓冲区的大小在合理范围内。如果节点上所有pax_machine的缓存大小超过了阈值，就会开始清理无用的pax_machine。淘汰算法采用lru机制，不能清理还未走完Paxos协议的pax_machine。pax_machine定义如下：

```
/* Definition of a Paxos instance */
struct pax_machine {
  linkage hash_link; // 在hash表桶中的位置
  stack_machine *stack_link; //指向hash_stack中的stack_machine
  lru_machine *lru; // 指向当前节点的paxos cache中的LRU缓存
  synode_no synode;
  double last_modified; /* Start time */
  linkage rv; /*  系统中的睡眠协程队列，本字段主要用于将协程挂到这个队列上，和一个协程执行完成后，唤醒睡眠的协程 */

  struct {
    ballot bal;            /* The current ballot we are working on */
    bit_set *prep_nodeset; /* Nodes which have answered my prepare */
    ballot sent_prop;      // 记录响应本节点的消息中号码最大的消息
    bit_set *prop_nodeset; /* Nodes which have answered my propose */
    pax_msg *msg;          /* The value we are trying to push */
    ballot sent_learn;
  } proposer;

  struct {
    ballot promise; /* Promise to not accept any proposals less than this */
    pax_msg *msg;   /* The value we have accepted */
  } acceptor;

  struct {
    pax_msg *msg; /* The value we have learned */
  } learner;
  int lock; /* 本条pax machine 对应的锁, 其实是一个标志位，标记本条pax machine对象是否正在被使用，比如处于使用状态的pax machine 不能被cache管理协程回收*/
  pax_op op;
  int force_delivery;
  int enforcer;
};

struct ballot {
	int32_t cnt;
	node_no node;
};
```

前三个字段与缓存相关。synode标识了该pax_machine处理的消息序号。在MGR中，一个消息/提案是以异步的方式走完Paxos协议，发起投票的proposer_task会在rv字段上等待并周期性被唤醒，直到该提案完成。op表示消息操作类型。proposer、acceptor和learner分别保存了消息序号为synode的提案从本节点视角看到的不同阶段的状态。proposer字段是提案的发起者维护的字段，accetor和learner不会操作proposer字段。MGR中的Paxos实现支持2阶段(multi-paxos)和3阶段(basic paxos)协议，相比2阶段，3阶段协议多了一个prepare的过程。MGR对于正常的事务消息采用2阶段的方式。所以proposer字段包括了prepare和propose 2个阶段，其中bal表示目前的提案编号，prep_nodeset和prop_nodeset分别表示已经回复prepare和propose消息的节点集；sent_prop和sent_learn分别在收到大多数节点回复prepare和propose消息时设置，设置为bal；msg字段在proposer_task进行pax_machine初始化时被置为等待投票的客户端消息，如果有其他节点回复的prepare消息中携带的proposal大于proposer节点的ballot，那么msg会被替换为回复消息对应的pax_msg对象，即投票的value发生改变。acceptor字段由作为提案接受者的节点维护，保存了本节点至今已收到的消息编号为synode的最大ballot，msg字段为对应ballot携带的pax_msg。learner字段表示编号为synode的消息最终被学习的消息体/value。显然，该字段不为空表示对应synode已经完成了Paxos协议。





### 2.2 paxos协议：Mencius算法实现

#### 2.2.1 basic paxos协议过程

正常事务处理时的流程也即paxos协议的流程，paxos协议主要分为三个阶段：prepare，propose，accept。

![basic paxos协议流程](https://img-blog.csdnimg.cn/20201229113215824.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4MjU4OTAz,size_16,color_FFFFFF,t_70)

**proposer_task**

prepare和propose会通过proposer_task协程发起，当xcom模块接受到上层客户端发送来的事务消息时，会被放入prop_input_queue队列中，proposer_task会循环的从队列中获取消息，并对普通的事务消息进行batch操作。而对于视图或者配置变更消息，则不能进行batch。当事务被batch以后，会进行消息的propose操作，典型的paxos协议需要三个阶段，而具有leader的paxos协议只需要两阶段。具有leader节点的paxos协议在正常消息提议时只需要走两阶段，但是当被检测leader故障等情况下需要走三阶段过程。在proposer_task中，会不停的尝试提议消息，直到消息在整个集群中达成一致。

由于paxos整个过程从proposer_task开始，proposer_task中还负责待提议消息号赋值和对应pax machine的获取。
主要逻辑如下：

```
static int proposer_task(task_arg arg) {
    while (!xcom_shutdown){
        // 从prop_input_queue队列中取出消息，存入client_msg中
        CHANNEL_GET(&prop_input_queue, &ep->client_msg, msg_link); // ep为当前协程的上下文特征，即栈数据
        
        // 对消息进行batch
        while (AUTOBATCH && ep->size <= MAX_BATCH_SIZE &&
                ep->nr_batched_app_data <= MAX_BATCH_APP_DATA &&
                !link_empty(&prop_input_queue.data)){
                
                msg_link *tmp;
                app_data_ptr atmp;
                CHANNEL_GET(&prop_input_queue, &tmp, msg_link);              
                atmp = tmp->p->a;
                // 对于视图或者配置变更信息，不可以被batch
                if (is_config(atmp->body.c_t) || is_view(atmp->body.c_t) ||
                    ep->nr_batched_app_data > MAX_BATCH_APP_DATA ||
                    ep->size > MAX_BATCH_SIZE) {
                    channel_put_front(&prop_input_queue, &tmp->l);
                    break;
                }
                ep->client_msg->p->a = atmp;
                                  
        }
        
        ep->msgno = current_message;  // 初始化消息号，current_message代表了本节点下一个可用的消息号，每当一个消息被提交到上层MGR时都会更新current_message
        
        // 直到要发送的消息在paxos中达成一致，否则不停的propose
        for (;;) { 
            // 从状态机pax machine缓存获取一个实例，用于发送GCS上层发过来的客户端消息，最长等待60s
            TASK_CALL(wait_for_cache(&ep->p, ep->msgno, 60));
            
            当自身配置为三阶段提交或者需要force_delivery(如发生配置变更)或者其他节点在等待本节点时超时所以提出了no-op，并且本节点已经accept了，那么就需要三阶段提交
            if (threephase || ep->p->force_delivery || ep->p->acceptor.promise.cnt) {
                push_msg_3p(ep->site, ep->p, ep->prepare_msg, ep->msgno, normal);
            } else {
                push_msg_2p(ep->site, ep->p);
            }
            
            // Try to get a value accepted
             while (!finished(ep->p)) {
                
                // 周期性的睡眠等待和醒来
                TIMED_TASK_WAIT(&ep->p->rv, ep->delay = wakeup_delay(ep->delay))；
                // 判断该pax_machine是否已经走完learn阶段或者如果在本次propose过程中已经得到了一个被提议了一个值，break；
                if (finished(ep->p)) break;
                push_msg_3p(ep->site, ep->p, ep->prepare_msg, ep->msgno, normal); // 否则，继续走三阶段阶段
            }

            // 检测本次提议最终获得的值是不是本节点想propose的值
            if (match_my_msg(ep->p->learner.msg, ep->client_msg->p)) {
                break;
            } else{ // 如果不是，进行重试，继续尝试提议本次想要提议的消息
                GOTO(retry_new);
            }
            
        }
        
    }
}
```
**prepare：**

在三阶段过程中，首先进行prepare，根据paxos协议，prepare准备提议一个消息，并从整个集群中得知已经被accept的最大提议号的消息。在prepare阶段只带有本次的提议号，不带有消息。主要过如下：
```
static void push_msg_3p(site_def const *site, pax_machine *p, pax_msg *msg,
                        synode_no msgno, pax_msg_type msg_type) {
  if (wait_forced_config) {
    force_pax_machine(p, 1);
  }

  assert(msgno.msgno != 0);
  
  BIT_ZERO(p->proposer.prep_nodeset); // 清空已经回复本节点发出的prepare消息的所有节点
  p->proposer.bal.node = get_nodeno(site); // 获取本节点的node号
  {
    int maxcnt = MAX(p->proposer.bal.cnt, p->acceptor.promise.cnt);
    p->proposer.bal.cnt = ++maxcnt; // 找到一个已知的最大的提议号
  }
  msg->synode = msgno; // 设置本次propose message的消息号为当前pax machine的消息号
  msg->proposal = p->proposer.bal; // 提议号
  msg->msg_type = msg_type; 
  msg->force_delivery = p->force_delivery;
  
  assert(p->proposer.msg);
  send_to_acceptors(p, "prepare_msg"){
      ....
      // 向当前视图中的所有节点发送prepare消息
      for (i = 0; i < max; i++) {
      if (test_func(s, i)) { // 调用all函数，总是return 1
        retval = _send_server_msg(s, i, p){
            msg_link *link = msg_link_new(p, to);
            // 最终消息被放入像server发送的outgoing消息队列中
            channel_put(&s->outgoing, &link->l);
        }
      }
    }
    ....
  }
}
```
当另外的节点收到prepare请求时，首先通过这个请求的类型字段prepare_op在dispatch_op中进行对应的处理：
```
    // 处理prepare 请求
    pax_msg *reply = handle_simple_prepare(p, pm, pm->synode);
    // 响应prepare请求，通过reply
    if (reply != NULL) SEND_REPLY;
```
根据paxos协议，当节点收到其他节点发送的prepare消息时，首先判断接受的消息的提议号如果不小于当前的提议号，则响应本节点已经接受的最大提议号的消息值。并做出以后不再接受比接受的prepare提议号小的承诺。在xcom中，处理prepare消息的详过程如下：
```
pax_msg *handle_simple_prepare(pax_machine *p, pax_msg *pm, synode_no synode) {
  pax_msg *reply = NULL;
  
  // 如果本节点已经学到了一个值
  if (finished(p)) { 
    reply = create_learn_msg_for_ignorant_node(p, pm, synode); // 响应已经学到的值
  } else {
    int greater =
        gt_ballot(pm->proposal,
                  p->acceptor.promise); // 判断收的提议号本节点做出承诺的提议号
    if (greater || noop_match(p, pm)) { // 如果满足承诺条件或者消息类型为noop，noop消息可以直接忽略
      p->last_modified = task_now();
      if (greater) {
        p->acceptor.promise = pm->proposal; // 承诺不再接受更小的提议
      }
      reply = create_ack_prepare_msg(p, pm, synode);// 创建响应prepare的消息
    }
  }
  return reply;
}
```
其中判断ballot大小的函数为：
```
int gt_ballot(ballot x, ballot y) {
  return x.cnt > y.cnt || (x.cnt == y.cnt && x.node > y.node);
}
```
可以看出通过比较cnt和节点的node号来确定

根绝paxos协议，对于响应prepare的消息 reply，会带有本节点已经accept的消息值，如果还没accept，则直接响应空消息ack_prepare_empty_op。响应prepare请求的消息通过以下方式创建：
```
reply = create_ack_prepare_msg(p, pm, synode);
static pax_msg *create_ack_prepare_msg(pax_machine *p, pax_msg *pm,
                                       synode_no synode) {
  CREATE_REPLY(pm);
  reply->synode = synode;
  if (accepted(p)) { // 如果已经接受一个值，那么回复消息中带有这个提议
    reply->proposal = p->acceptor.msg->proposal;
    reply->msg_type = p->acceptor.msg->msg_type;
    reply->op = ack_prepare_op; // 把消息类型设置为ack_prepare_op
    safe_app_data_copy(&reply, p->acceptor.msg->a);
  } else { // 还没有接受任何值，直接返回空
    reply->op = ack_prepare_empty_op;
  }
  return reply;
}
```

当发起prepare的节点收到其他节点对于prepare请求的ack_prepare_op响应时，如果响应的提议号比本节点的提议号大，则进行响应消息的替换，并检测是否可以进入propose过程：
```
static void handle_ack_prepare(site_def const *site, pax_machine *p,
                               pax_msg *m) {

    bool_t can_propose = FALSE;
    if (m->op == ack_prepare_op &&
        gt_ballot(m->proposal, p->proposer.msg->proposal)) { // 如果响应的prepare请求是ack_prepare_op(代表非空)并且提议号比自身提议号大
      replace_pax_msg(&p->proposer.msg, m); // 需要将自身的提议内容设置为响应的提议内容
      assert(p->proposer.msg);
    }
    if (gt_ballot(m->reply_to, p->proposer.sent_prop)) { // 如果比自身已经收到的所有响应的提议号都大，才进行check_propose过程
      can_propose = check_propose(site, p);
    }
    return can_propose;
}
```
如果收到的ack_prepare_op的个数大于集群中节点数目的一大半，则可以进入propose过程，通过以下方式检测是否prepare成功：
```
bool_t check_propose(site_def const *site, pax_machine *p) {
 
  {
    bool_t can_propose = FALSE;
    // 判断是否大多数节点都已经回复本次prepare
    if (prep_majority(site, p)) {
      // 进行一些propose阶段的初始化工作
      p->proposer.msg->proposal = p->proposer.bal;
      BIT_ZERO(p->proposer.prop_nodeset); // 清空记录收到节点响应的集合
      p->proposer.msg->synode = p->synode; // 更新消息号
      init_propose_msg(p->proposer.msg); // 将消息初始化为accept_op
      p->proposer.sent_prop = p->proposer.bal; // 更新收到所有响应里最大的提议号
      can_propose = TRUE;
    }
    return can_propose;
  }
}
```
**propose:**

按照与prepare相同的过程，进入propose阶段后会向所有的节点发送propose请求。当其他节点收到其他节点的accept_op消息时，根据paxos协议，如果propose消息的提议号大于等于当前节点做出的承诺号，则接受这个消息。过程如下：
```
static void handle_accept(site_def const *site, pax_machine *p,
                          linkage *reply_queue, pax_msg *m) {
  {
    pax_msg *reply = handle_simple_accept(p, m, m->synode){
        pax_msg *reply = NULL;
        if (finished(p)) { // 已经learned到了一个值
            reply = create_learn_msg_for_ignorant_node(p, m, synode);
            } else if (!gt_ballot(p->acceptor.promise,
                        m->proposal) || 
             noop_match(p, m)) { // 如果propose的提议号大于等于本节点承诺不再接受的提议号 或者 pax_msg是noop，或者本节点pax_machine对象的状态是已经accept了一个noop消息
            p->last_modified = task_now();
            replace_pax_msg(&p->acceptor.msg, m);
            reply->op = ack_accept_op; // 设置消息类型为ack_accept_op;
            reply->synode = synode; // 设置消息号
        }
        return reply;
    };
    // 响应propose请求
    if (reply != NULL) SEND_REPLY;
  }
}
```
当发出accept_op的节点收到ack_accpt_op的消息时，会检查能否进入learn阶段。做以下的处理：                                                             
```
static void handle_ack_accept(site_def const *site, pax_machine *p,
                              pax_msg *m) {
                              
    pax_msg *learn_msg = handle_simple_ack_accept(site, p, m){
        pax_msg *learn_msg = NULL;
        if (get_nodeno(site) != VOID_NODE_NO && m->from != VOID_NODE_NO &&
            eq_ballot(p->proposer.bal, m->reply_to)) { // 如果是发送本节点的ack_accept_op消息
            BIT_SET(m->from, p->proposer.prop_nodeset);
            if (gt_ballot(m->proposal, p->proposer.sent_learn)) { // 同ack_prepare_op操作，同样为了拒绝比当前已经接受的accept消息号小的请求
                learn_msg = check_learn(site, p);  // 检查是否可以进入learn阶段
            }
        }
        return learn_msg;
    };
    if (learn_msg != NULL) {  // 如果learn_msgx消息不为空
      if (learn_msg->op == tiny_learn_op) { // 如果设置的learn 消息的方式为tiny_learn_msg，
        send_tiny_learn_msg(site, learn_msg);
      } else {
        /* purecov: begin deadcode */
        assert(learn_msg->op == learn_op);
        send_learn_msg(site, learn_msg);
        /* purecov: end */
      }
    }
}
```

**learn：**

当接受到一半以上的成功accept的响应时，可以进入learn阶段。需要做以下的检查：
```
static pax_msg *check_learn(site_def const *site, pax_machine *p) {
  
pax_msg *learn_msg = NULL;
if (get_nodeno(site) != VOID_NODE_NO && prop_majority(site, p)) {

  if (no_duplicate_payload) { // no_duplicate_payload 被设置为1
    learn_msg = create_tiny_learn_msg(p, p->proposer.msg);
  } else {
    /* purecov: begin deadcode */
    init_learn_msg(p->proposer.msg);
    learn_msg = p->proposer.msg;
    /* purecov: end */
  }
  p->proposer.sent_learn = p->proposer.bal;
}
return learn_msg;
}
```
当进入learn阶段以后，根据收到的消息是tiny_learn_op和learn_op，做出对应的不同的处理，在更新本节点pax_machine对象时，tiny_learn_op是将对象的acceptor.msg赋learner.msg，而learn_op时是将从其他节点收到的pax_msg赋予本节点pax_machine对象的acceptor.msg和learner.msg。对于前者，如果当前节点已经accet值并且接收到tiny_learn_op消息对应的learn值等于自身accpet值，则直接进入进入do_learn()过程，此时一条消息走完了整个paxos协议，在所有节点上达成一致；如果上述两个条件都不满足，则需要调用send_learn向其他节点学习需要最终的值，主要过程如下：
```
void handle_tiny_learn(site_def const *site, pax_machine *pm, pax_msg *p) {
  if (pm->acceptor.msg) {
    if (eq_ballot(pm->acceptor.msg->proposal, p->proposal)) { // 如果是当前已经接收的提议
      pm->acceptor.msg->op = learn_op;
      update_max_synode(p); // 更新已知的最大消息号
      handle_learn(site, pm, pm->acceptor.msg);
    } else { // 如果接收的learn消息不是本节点已经accept的消息，则向其他节点学习
      send_read(p->synode);
    }
  } else { // 如果本节点还没有接收值，则向其他节点学习
    send_read(p->synode);
  }
}
```
在handle_learn函数中,主要消息的替换和把消息当前pax_machine设置为被选定，此时，消息在paxos中的整个流程结束。此时还会激活sweeper_task协程进行pax_machine 缓冲区的清理和进行Mencius协议的simple
paxos 过程。handle_learn主要过程如下：
```
void handle_learn(site_def const *site, pax_machine *p, pax_msg *m) {
  if (!finished(p)) { / 避免重复学习新的值
    activate_sweeper(); // 
    do_learn(site, p, m){
        if (m->a) m->a->chosen = TRUE; // 首先将消息设置为被选定状态
        // 在do_learn 中，将自身作为acceptor和learner角色时的值替换为最终被决定的消息m
        replace_pax_msg(&p->acceptor.msg, m);
        replace_pax_msg(&p->learner.msg, m);
    }
    // 处理一些特殊消息
    if (m->a && m->a->body.c_t == unified_boot_type) {
      XCOM_FSM(x_fsm_net_boot, void_arg(m->a));
    }
    /* See if someone is forcing a new config */
    if (m->force_delivery && m->a) {
      /* Immediately install this new config */
      switch (m->a->body.c_t) {
        case add_node_type:
          ....
          break;
        /* purecov: end */
        case remove_node_type:
          ....
          break;
        /* purecov: end */
        case force_config_type:
          .....
          break;
        default:
          break;
      }
    }
  }

  task_wakeup(&p->rv);
}
```

在handle_tiny_learn中，如果本节点acceptor.msg不存在或者acceptor.msg->proposal不等于tiny_learn_op消息携带的投票器，则表示没有参与之前的accept流程或者当前accept值不是经过大多数同意的值，这两种情况需要调用send_read处理：
```
static void send_read(synode_no find) {
  // 找到一个属于当前视图的节点
  site_def const *site = find_site_def(find);

  /* See if node number matches ours */
  if (site) {
    //  如果找到的节点不是本节点
    if (find.node != get_nodeno(site)) {
      pax_msg *pm = pax_msg_new(find, site); // 创建一个pax_msg消息
      ref_msg(pm); // 对本条pm消息引用计数
      create_read(site, pm); // 将消息类型设置为read_op类型
      
      // 如果本节点还没有分配节点号(why?)
      if (get_nodeno(site) == VOID_NODE_NO)
        send_to_others(site, pm, "send_read"); // 向所有的其它节点发送read_op消息
      else
        send_to_someone(site, pm, "send_read"); // 通过round-bin的方式挑选一个live节点发送read_op

      unref_msg(&pm); // 解引用，如果引用计数为0，析构本条消息
    } else { // 如果找到的节点是本节点自身
      pax_msg *pm = pax_msg_new(find, site);
      ref_msg(pm);
      create_read(site, pm);
      send_to_others(site, pm, "send_read");
      unref_msg(&pm);
    }
  }
}
```

**提交消息到上层客户端**

当一条消息被learn之后，表明已经在大多数节点上达成一致，此时便可以进入paxos协议的提交阶段，在xcom模块中，即将达成一致的消息返回给MGR的GCS模块。提交阶段主要分为两个过程：x_fetch和x_execute，前者是获取之前已经被do_learn()设置为被选定状态的消息，后者用于将这些消息发送给上层GCS模块。主要通过executor_task()协程进行：
```
static int executor_task(task_arg arg MY_ATTRIBUTE((unused))) {

  if (executed_msg.msgno == 0) executed_msg.msgno = 1;
  delivered_msg = executed_msg;
  ep->xc.state = x_fetch;
  executor_site = find_site_def_rw(executed_msg);

  while (!xcom_shutdown && ep->xc.state != 0) {
    // 进入请求获取阶段
    if (ep->xc.state == x_fetch) { 
        TASK_CALL(get_xcom_message(&ep->xc.p, executed_msg, FIND_MAX)); // 调用get_xcom_message获取已经被选定但是还未被execute_task感知的消息
        x_fetch(&ep->xc){
            if (x_check_exit(xc)) {
                xc->state = x_terminate;
            } else {
                SET_EXECUTED_MSG(incr_synode(executed_msg)); // 增加executed_msg号，用于表示已经走完Paxos全流程但还未被executor_task()处理的最小的synode请求。同时递增全局的下一个可用的current_message,用于proposer_task中客户端消息号的赋值
                if (x_check_execute_inform(xc)) {
                    xc->state = x_execute; // 将状态设置为x_execute
                }
        }
      }
    } else { // 当协程是execute状态，处于请求执行阶段
      ep->xc.state(&ep->xc)； // 调用x_execute协程
    }
  }
}
```

对于 x_execute 协程，在执行之前会首先判断是否满足执行条件，delivery_limit为视图变更阶段退出节点可以执行的消息号的上限，只有当前消息号小于delivery_limit时才可以执行，详见集群成员变更过程：
```
static void x_execute(execute_context *xc) {
  site_def const *x_site = find_site_def(delivered_msg);

  
  xc->p = get_cache(delivered_msg);
  
  if (xc->p->learner.msg->msg_type != no_op) {
    /* 只有小于delivery_limit时，才可以执行 */
    if (xc->exit_flag == 0 || synode_lt(delivered_msg, xc->delivery_limit)) {
     
      last_delivered_msg = delivered_msg;
      execute_msg(find_site_def_rw(delivered_msg), xc->p, xc->p->learner.msg);
    }
  }
  /* delivered_msg 等于当前配置的start号，表示此配置刚开始执行，旧配置已经结束，可以进行垃圾回收机制 */
  if (synode_eq(delivered_msg, x_site->start)) {
    garbage_collect_servers();
  }

  /* 检测是否可以退出和增加delivered_msg */
  x_check_increment_execute(xc);
}


void execute_msg(site_def *site, pax_machine *pma, pax_msg *p) {
  
  last_delivered_msg = delivered_msg;
  execute_msg(find_site_def_rw(delivered_msg), xc->p, xc->p->learner.msg){
    app_data_ptr a = p->a;
    // 对于不同的消息类型，做不同的处理
    switch (a->body.c_t) {
      // 配置消息
      case unified_boot_type:
      case force_config_type:
        deliver_config(a);
      case add_node_type:
      case remove_node_type:
        break;
      case app_type:
        // 如果是客户端类型消息，将会把消息发送到客户端，最终会执行MGR上层注册到Xcom的回调处理函数xcom_receive_data进行处理。
        deliver_to_app(pma, a, delivery_ok);
        break;
      // 全局视图消息
      case view_msg:
        deliver_global_view_msg(site, p->synode);
        break;
      default:
        break;
  }
}
```


#### 2.2.1 simple paxos协议过程

在Mencius算法中，所谓simple paxos即在设置leader角色的paxos协议过程中，规定了：

- 只有leader可以正常的提议值，且当leader节点提议no-op值时，不用经过两阶段。
- 非leader节点只能提议no-op值，且必须经过三阶段

对于第一点要求，由于非leader节点只能提议no-op值，因此leader提议no-op时不经过两阶段过程也不会造成不一致的情况；对于第二点要求，虽然非leader节点只可以提议非no-op值，但最后被decide的值也不一定是no-op，因为此时可能leader节点提议的正常事务消息已经被部分节点accept，这里的三阶段可以利用paxos原理，保证不会发生不一致的现象。

何时提议no-op值？

在Mencius算法中，提议no-op的场景如下：

- 1. instance序列中，当大于本节点负责的instance已经被选定value时，所有本节点负责的instance且索引小于这个已经被决定的instance都被提议no-op
- 2. 非leader节点怀疑leader节点故障，希望提议no-op填充故障leader节点负责的instance
![image](https://img-blog.csdnimg.cn/20201030111556671.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4MjU4OTAz,size_16,color_FFFFFF,t_70#pic_center)

对于第一点，在xcom模块中，主要通过sweeper_task 协程负责触发提议no-op的过程，首先会寻找本节点上已经执行的消息号executed_msg，然后从此消息号开始检测是否需要提议no-op消息。首先需要判断需要检测的消息号是不是小于距今已知的最大消息号(如果大于，那么代表以后本节点可能会提议此消息号，还不需要进行skip操作)，根据此消息号从paxos cache中新建或者查找一个已经存在的pax machine，即instance，如果此instane满足一下条件:没有处于被占用的状态，没有accept任何值，自身没有提议任何值，没有选定任何值，则可以进行skip操作。

sweeper不停的增加消息号，检测每个消息号对应的instance状态，对于满足条件的进行skip操作。主要过程如下：

```
static int sweeper_task(task_arg arg MY_ATTRIBUTE((unused))) {
  
  ep->find = get_sweep_start(); // 初始化为本节点的已经被执行的消息号
  
  while (!xcom_shutdown) { // 不停处于检测状态
    ep->find.group_id =
        executed_msg.group_id; /* In case group id has changed */
    {
      while (synode_lt(ep->find, max_synode) && !too_far(ep->find)) {
        /* pax_machine * pm = hash_get(ep->find); */
        pax_machine *pm = 0;
        if (ep->find.node == VOID_NODE_NO) {
          if (synode_gt(executed_msg, ep->find)) {
            ep->find = get_sweep_start();
          }
          
        }
        pm = get_cache(ep->find); // 寻找或者新建一个对应消息号的instance
        if (pm && !pm->force_delivery) { /* We want full 3 phase Paxos for
                                            forced messages */
          
          if (!is_busy_machine(pm) && pm->acceptor.promise.cnt == 0 &&
              !pm->acceptor.msg && !finished(pm)) { //instance未被使用, 没有在对应instance上做出承诺,没有接受别的节点建议的消息或者自身未建议消息,instance没有学习到任何值
            pm->op = skip_op;
            
            
            skip_msg(pax_msg_new(ep->find, find_site_def(ep->find))) // 新建一条skip 消息进行 skip操作{
                prepare(p, skip_op);
                p->msg_type = no_op;
                return send_to_all(p, "skip_msg"); // 根据Mencius算法，对于skip消息协调者可以直接入learn阶段，而不用三阶段或者两阶段
            };
            printf("can skipping, %lld,   %lld\n",(long long)ep->find.msgno, (long long)ep->find.node);
            fflush(stdout);
           
          }
        }
        ep->find = incr_msgno(ep->find); // 递增检测所有的消息号
        printf("next to detect skip, %lld,  %lld\n",(long long)ep->find.msgno, (long long)ep->find.node);
        fflush(stdout);
      }
    }

}
```

对于第二点，发生在节点提交到上层MGR时，在basic paxos中的executor_task协程中，在x_fetch，也即获取可以提交给上层MGR的消息时，如果对于一个消息号对应的instance在本节点上还未被决定，首先本节点将会尝试从其他节点learn对应的消息，如果尝试无果，将会通过三阶段(从prepare过程开始)尝试skip这条消息。对应代码如下：

```
int get_xcom_message(pax_machine **p, synode_no msgno, int n) {
 
  ....
  *p = force_get_cache(msgno); // 获取消息号对应的paxs machine,即instance
 
  dump_debug_exec_state();
  while (!finished(*p)) { // 如果此instance还未达成一致，不停重试
    ....
    find_value(ep->site, &ep->wait, n); // 从其他节点学习或者尝试skip这条instance
    *p = get_cache(msgno);
    ...
  }
}
```
在 find_value中，当前的节点会在4次以内获取指定msgno消息的尝试。第一次和第二次会尝试从其他节点learn对应的消息。而从第3次开始，本节点将通过检测集群中还活动的节点，如果本节点可以成为leader，会通过三阶段尝试跳过这条消息，否则还是从其他节点读取。如果前三次尝试仍然失败，从第4次开始，集群中所有节点都会进行三阶段过程。
```
static void find_value(site_def const *site, unsigned int *wait, int n) {

  switch (*wait) {
    case 0: 
    case 1: // 第一次和第二次都会尝试从其他节点读取
      read_missing_values(n);
      (*wait)++;
      break;
    case 2: // 第三次：如果本节点是集群中节点号最小的节点，则由本节点负责提议no op，接下来进入push_msg_3p过程
      if (iamthegreatest(site))
        propose_missing_values(n);
      else
        read_missing_values(n);
      (*wait)++;
      break;
    case 3:
      propose_missing_values(n);
      break;
    default:
      break;
  }
}
```

### 2.2 故障检测

MGR中故障响应机制主要包括故障探测，移除故障节点和自动重新加入，其中xcom模块主要负责故障探测。每个节点都有一个server对象，其中与故障检测相关的字段如下：
```
/* Server definition */
struct server {
    ...
  double detected;           /* Last incoming */
};

struct site_def {
  ....
  detector_state detected;   /* 每个节点的最后一条传入消息的时间 */
};

typedef double detector_state[NSERVERS];


```

xcom在每次成功的发送数据到某节点或者从某个节点接收到数据时都会记录当前时间(detected time，即server字段的detected)。意味着这个节点在当前时间还活着。

与故障检测相关的协程主要为alive_task()和detector_task()，

alive_task负责：

- 在节点空闲了0.5秒后，alive_task()会发送i_am_alive_op消息给其他节点。空闲是指0.5秒内没有发送任何消息出去。
```
if (server_active(site, get_nodeno(site)) < sec - 0.5) 
```

- 当某个节点4秒(#define MAX_SILENT 4)内没有任何通信时(代码中称作may_be_dead)，发送are_you_alive_op消息给这个节点，去探查该节点是否还活着。

detector_task()主要任务是：

每1秒做一轮检查。检查节点是否还活着。判断的标准是：

```
#define DETECTOR_LIVE_TIMEOUT 5.0

detected_time > task_now() - DETECTOR_LIVE_TIMEOUT
```

即5秒内没有和这个节点通信。如果此节点的detected time在5秒之内，则为活着，否则认为该节点可能已经死去。

当detector_task()发现任何节点的状态发生了变化：从活着变成可能死了；从可能死了变成活着

都会通过local view回调函数xcom_receive_local_view来处理状态变化。状态的变化由GCS模块处理。

### 2.3 集群成员变更
xcom集群成员变更算法参考了两篇论文，一篇是lamport论文 Reconfiguring a State Machine 中的方法。论文解读如下：

文档：paper Reconfiguring a State Machine

链接：http://note.youdao.com/noteshare?id=fe63ab4a6e146e67927cca351bc2e806&sub=56A28ECB00134D44A989A080C9E0984E

另一篇是：

The SMART way to migrate replicated stateful services. In Proc. EuroSys’06, ACM SIGOPS/EuroSys European Conference on Computer Systems 

集群成员变更分为加入节点和删除节点的过程，一致性协议中集群成员变更的方法主要分为两类，一类是两阶段方式，一类是raft 论文中提到的joint consencus方式。在xcom中采用了上述论文中的Ra算法，其中a为alpha个延迟，即集群成员变更的消息被提议以后会经过alpha延迟后才生效，这可以允许alpha个消息的并发提交。经过alpah个消息后，每个节点按照新的配置进行集群成员变更的操作。在xcom中，alpha对应的变量为**event_horizon**。

当有新的节点加入集群时，不会发生数据丢失的问题。但是当有节点退出时，处理过程比较复杂。下面通过一个例子介绍一下具体的节点退出过程：


  考虑三个配置C1和C2，C3。C1有两个节点，A和B。C2只有节点B。C3为空。
  消息编号为N的配置将在（至少）alpha消息延迟后激活，其中alpha 是event horizon大小。

  所以，C1.start=C1+alpha，C2.start=C2+alpha。
  从C1中删除的A在新的配置C2（在本例中为B）中的大多数节点从配置C1中学习了所有消息(所有小于C2.start的消息)之前，不能退出。
  如何知道大部分C2已经学习到了这些信息？

  假设E表示尚未由被选定（并执行）的第一条消息，则提议者将不会尝试提出编号大于等于E+alpha的消息，
  并且所有消息编号大于等于E+alpha的传入消息都将被忽略。
  E由executor_task递增，因此所有小于E的消息都是已知的。这意味着当E+alpha的值可以被学习到时，直到E（包括E）的所有消息也都已经被学习到，
  尽管并非所有消息E+1..E+alpha-1都需要被learn。

  **这就要求退出的节点（A）需要等待，直到它知道C2.start+alpha的值，
  因为到那时它知道C2中的大多数节点已经准备好执行C2.start，
  这反过来意味着C2中的大多数节点都知道来自配置C1的所有值。**
  注意，离开C1的节点应该传递给应用程序的最后一条消息是C2.start-1，这是C1的最后一条消息。

  退出的节点如何从下一个配置中获取被选定的值？有两种方法都被使用。
  首先，尝试退出的节点可以简单地请求消息。get_xcom_message（）将对所有消息小于等于max_synode执行此操作，但可能需要一些时间。
  其次，C2的节点可以将消息C2.start..C2.start+alpha发送给被移除的节点（C1中的节点，而不是C2中的节点）。
  inform_removed()函数负责执行此操作。通过跟踪包含要退出的节点的最旧配置来处理配置足够接近C0<C1<=C0+alpha的情况。

  上述方式将处理离开C1的节点的情况。如果处理离开C2的节点呢？C3为空，因此离开C2的B不能等待来自C3的消息。
  但由于C3是空的，所以不需要等待。它可以在执行完C3.start‐1（C2的最后一条消息）后立即退出。
  如果C3.start-1<C2.start+alpha呢？如果C2和C3接近，就会发生这种情况：
  B将在A有机会学习C2.start+alpha之前退出，这将使A永远都被hang住。
  显然，需要施加一个额外的约束，即C3.start必须大于C2.start+alpha。这由空配置的特殊测试来处理。

  除了上述方法之外，还有中比较完美的解决方案尚未被实现即raft论文中的共同一致，因为它需要对一致性协议的过程进行更多的修改。
  按照共同一致算法，在xcom中，如果我们要求消息C2..C2.start‐1的大部分来自C1中的节点和C2中的节点，
  那么不在C2中的节点可以在执行消息C2.start-1后退出，因为我们知道C2的大多数节点也同意这些消息，
  因此它们不再依赖于不在C2中的节点。即使C2是空的，这也是有效的。
  请注意，要求C1和C2的多数与要求C1+C2的多数不同，这意味着提议者逻辑需要考虑来自两组不同接受者的应答者。
  
当集群成员变更的配置消息传来时，首先会进行配置的更改，如果配置更改成功，会判断本节点是属于删除的节点还是新添加的节点还是旧配置中存在，新配置中还会继续保留的节点，其中后两种情况的处理过程相同；下面看一下节点加入或者退出时具体的执行过程：

```
static void x_fetch(execute_context *xc) {
  /* 在准备好执行来自新配置定义的消息之前，不要传递消息。此时需要保证大多数节点已经从旧配置集群中学习到了所有消息。 */

  app_data *app = xc->p->learner.msg->a;
  if (app && is_config(app->body.c_t) &&
      synode_gt(executed_msg, get_site_def()->boot_key)) /* 判断是否是配置变更请求，当节点退出时，消息类型为remove_node_type */
  {
    site_def *site = 0;
    bool_t reconfiguration_successful =
        handle_config(app, (xc->p->learner.msg->force_delivery != 0));
    if (reconfiguration_successful) {
      /*如果重新配置失败，则不会产生任何影响。只有当重新配置生效时，下面的过程才有意义 */
      set_last_received_config(executed_msg);
      garbage_collect_site_defs(delivered_msg);
      site = get_site_def_rw();
      if (site == 0) {
        xc->state = x_terminate;
        return;
      }
     
      if (xc->exit_flag == 0) {
        /* We have not yet set the exit trigger */
        setup_exit_handling(xc, site); // 设置退出逻辑，包括设置退出的延迟消息号等
      }
    }
  } 
  /* 检查是否可以退出和增加 executed_msg */
  x_check_increment_fetch(xc);
}
```
对于setup_exit_handling 函数，主要过程如下：
```
static void setup_exit_handling(execute_context *xc, site_def *site) {
  synode_no delay_until;
  if (is_member(site)) { // 如果本节点属于新的配置，那么代表本节点不是新添加进来的节点，也不是被删除的节点，可能是新被加入的节点，也可能是已经存在的节点
    delay_until = compute_delay(site->start, site->event_horizon); // 根据event_horizon 计算退出的延迟，计算方式为start.msgno += event_horizon
  } else { 
    /* 看看本节点离开时下一个配置是否会是空，即上述例子中的C3配置。如果新的site为空，应该在从配置传递完最后一条消息后退出。 */

    /* 在下一个配置开始生效后不能传递任何消息 */
    xc->delivery_limit = site->start;

    /* 
    如果本节点不是新配置的成员，应该在看到超过当前节点结束消息号的足够多的消息后退出，对应于上述例子中的c2+start。这样可以确保下一个配置的大多数节点都学习到了属于当前配置的所有消息。
     */
    xc->exit_synode = compute_delay(site->start, site->event_horizon);
    if (is_empty_site(site)) {
      /* 如果新配置为空，增加start以允许节点在start之前终止。这就好像在exit_synode之后有一个非空的组，有效地允许当前组的大多数成员在exit_synode之前的所有消息上达成一致。
       */
      site->start = compute_delay(
          compute_delay(site->start, site->event_horizon), site->event_horizon);
    }
    if (!synode_lt(xc->exit_synode, max_synode)) {
      /* 需要来自下一个配置的消息，所以相应地设置max_synode。 */
      set_max_synode(incr_synode(xc->exit_synode));
    }
    /* 设置在哪里切换到执行并通知删除的节点 */
    delay_until = xc->exit_synode;

    /* 代表本节点是被删除的节点且将要退出 */
    xc->exit_flag = 1;
  }


  if (synode_gt(delay_until, max_synode))
    set_max_synode(incr_msgno(delay_until)); // 更新m最大消息号
  fifo_insert(delay_until);
  (xc->inform_index)++;

}
```
下面检测此时是否可以退出:
```

static void x_check_increment_fetch(execute_context *xc) {
  if (x_check_exit(xc)) { // 表明本节点需要退出
    xc->state = x_terminate;
  } else { // 本节点还属于新的配置中，不需要退出
    SET_EXECUTED_MSG(incr_synode(executed_msg));
    if (x_check_execute_inform(xc)) { // inform_removed 函数，将消息推送到将要离开的节点
      xc->state = x_execute; 然后便可以继续执行
    }
  }
}
// 当本节点是被删除的节点且满足删除条件时，可以退出；满足删除的条件为本节点delivered_msg消息号没有超过传递限制且本节点executed_msg消息号已经等于exit_synode消息号了，那么此时也可以保证新配置中节点已经学习到了旧配置中所有的消息
static int x_check_exit(execute_context *xc) {
  return (xc->exit_flag && !synode_lt(executed_msg, xc->exit_synode) &&
          !synode_lt(delivered_msg, xc->delivery_limit));
}
```

## 3. 协程模块task
xcom首先实现了一套协程库：
```
/** \file
        Rudimentary task system in portable C, based on Tom Duff's switch-based
   coroutine trick
        and a stack of environment structs. (continuations?)
        Nonblocking IO and event handling need to be rewritten for each new OS.
*/
```
可以看出xcom中采用了[基于"Duff-switch"的协程实现](https://www.chiark.greenend.org.uk/~sgtatham/coroutines.html)，所谓"Duff-switch",是指主要通过c语言中的switch语句与 循环语句的嵌套实现函数退出后循环语句的继续执行，下面展示一种协程的无堆栈实现：
```
int function(void) {
    static int i, state = 0;
    switch (state) {
        case 0: 
        for (i = 0; i < 10; i++) {
            state = 1; 
            return i;
            case 1:;
        }
    }
} 
```

该函数的作用是第i次调用时返回i，最多10次，其核心部分为switch语句以及return前后两句，通过设置不同的state来保证下一次调用时从上次退出的地方继续执行。

因此在每次调用return时设置的不同的state，利用switch的条件跳转可以跳转到函数指定的位置继续执行。可以利用__LINE__宏来设置state，下面是简单利用__LINE__ 实现协程举例：
```
#include <stdio.h>

#define TASK_BEGIN static int state = 0; switch (state) { case 0:
#define TASK_YIELD(x) do { state = __LINE__; return x; case __LINE__:; } while (0)
#define TASK_END }



void f1() {
    TASK_BEGIN;
    puts("1");
    puts("2");
    TASK_YIELD();
    puts("3");
    TASK_END;
}

void f2() {
    TASK_BEGIN;
    puts("x");
    TASK_YIELD();
    puts("y");
    puts("z");
    TASK_END;
}

int main (void) {
    f1();
    f2();
    f1();
    f2();
    return 0;
} 
```
```
void f1(){
    static int state = 0;
    switch (state)
    { case 0:
        puts("1");
        puts("2");
        do{ 
            state = __LINE__;
            return;
    case __LINE__:;
            } while (0);
        puts("3");
    };
}
```
上述代码运行后会依次输出1,2,x,3,y,z。可以看出各个函数执行是一个交互的过程，并能保证函数退出后继续从上次返回的位置执行。
main()中演示的过程便是协程切换的过程。

因为switch内部不能任意定义变量，所以需要在TASK_BEGIN之前定义所需变量，在xcom模块中，每个协程开始之前都会定义和初始化与协程运行和切换相关的上下文。xcom中协程机制的实现和如上的实现方式相同，但是添加了堆栈，更为复杂。xcom中所有的协程(task)也通过类似于TASK_BEGIN, TASK_YEILD, TASK_END以及其他更为完善的机制进行协程的开始，睡眠，抢占和跳转。下面将主要介绍xcom的协程机制：

协程的数据结构如下：
```
struct task_env {
  linkage l;    /* Used for runnable tasks and wait queues */
  linkage all;  /* Links all tasks */
  int heap_pos; /* Index in time priority queue, necessary for efficient removal
                 */
  terminate_enum
      terminate;        /* Set this and activate task to make it terminate */
  int refcnt;           /* Number of references to task */
  int taskret;          /* Return value from task function */
  task_func func;       /* The task function */
  task_arg arg;         /* Argument passed to the task */
  const char *name;     /* The task name */
  TaskAlign *where;     /* High water mark in heap */
  TaskAlign *stack_top; /* The stack top */
  TaskAlign *sp;        /* The current stack pointer */
  double time;          /* Time when the task should be activated */
  TaskAlign buf[TASK_POOL_ELEMS]; /* Heap and stack */
  int debug;
  int waitfd;
  int interrupt; /* Set if timeout while waiting */
};
```
前两个个字段用于所有协程底层数据结构的阻止方式；refcnt用于表示一个协程被引用的次数，协程只被创建一次，但是可以被引用多次，当引用次数为0时，便可以删除；func字段是一个函数指针，指向协程代表的函数，taskret表示协程的返回值，用于判断协程被挂起还是真的退出，name表示协程代表的函数的名；缓冲区buf用于协程的堆栈，where代表堆的最高水位，初始指向buf[0]的位置，stack_top字段指向栈顶，初始指向buf[TASK_POOL_ELEMS -1 ]位置，每分配一次协程栈便减一，sp用于表示某个协程在栈中的位置。

另外比较重要的数据结构包括task queue：
```
/* Priority queue of task_env */
struct task_queue {
  int curn;
  task_env *x[MAXTASKS + 1];
};
typedef struct task_queue task_queue;
```
task_queue 定义的全局变量为task_time_q，用来保存休眠的协程，处于task_time_q的协程不能立刻执行，需要等待一段时间。

此外由全局活动(等待执行)协程列表tasks，是协程创建(task_new())后构成的双向循环列表，处于执行状态和等待执行的协程都处于列表中



### 协程创建
主要用于状态初始化，绑定对应的函数，加入到活动协程列表tasks中去。
```
task_env *task_new(task_func func, task_arg arg, const char *name, int debug) {
  task_env *t;
  if (link_empty(&free_tasks))
    t = (task_env *)malloc(sizeof(task_env));
  else
    t = container_of(link_extract_first(&free_tasks), task_env, l);
  IFDBG(D_NONE, FN; PTREXP(t); STREXP(name); NDBG(active_tasks, d););
  task_init(t);
  t->func = func;
  t->arg = arg;
  t->name = name;
  t->debug = debug;
  t->waitfd = -1;
  t->interrupt = 0;
  activate(t){
      link_into(&t->l, &tasks);  //tasks为可执行协程双向循环列表，tasks为列表的头结点， t加入到tasks的前面
  }
  task_ref(t);
  active_tasks++;
  return t;
}
```

### 协程开始
协程开始的过程如下：
```
#define TASK_BEGIN                                            \
 
  switch (stack->sp->state) {                                 \
    case 0:                                                   \
      pushp(stack, TASK_ALLOC(stack, struct env));            \
      ep = _ep;                                               \
      assert(ep);                                             \
      TERM_CHECK;

```
由于协程的运行机制为单线程模式，因此某一时刻只能有一个协程在运行，因此xcom设置一个全局变量stack，作为当前正在运行的协程栈，stack->sp->state为协程运行的位置，即代码的第几行，初始时设置为0，因此在协程被创建(task_new)第一次执行时，会进入switch语句的case 0 分支，首先会分为此协程分配栈帧，并且将该协程的运行环境上下文ep与全局变量_ep(stack->sp->ptr)绑定。因此在协程被挂起之前,运行过程一直处于switch 语句的case 0 分支内。

### 协程挂起(YIELD)
当协程被挂起后会暂停执行，调度过程会切换到下一个活动的协程执行，协程从运行状态转化为挂起状态的过程如下所示：
```
#define TASK_YIELD                     \
  {                                    \
    TASK_DEBUG("TASK_YIELD");          \
    stack->sp->state = __LINE__;       \
    return 1;                          \
    case __LINE__:                     \
      TASK_DEBUG("RETURN FROM YIELD"); \
      ep = _ep;                        \
      assert(ep);                      \
      TERM_CHECK;                      \
  }
```
注意，在case __LINE__ 语句之前的所有代码段都处于switch 的case 0 分支内，当调用 TASK_YIELD时，只需要保存当前协程的运行位置，即stack->sp->state = __LINE__， 如果下次协程被重新调度执行，那又会进行TASK_BEGIN过程内，此时会命中switch过程的 case __LINE__ 位置，这样便可以接着上次挂起的位置继续执行，通过 ep = _ep 语句恢复一下运行上下文。

当一个协程被TASK_YIELD之后依然会处于活动协程队列tasks中，如果活动队列中没有其他的活动事务，可以又被执行；还有一种延迟执行的挂起方式，即首先将协程从活动队列中移除，按照延时设置的时间，放入task_time_q中相对位置（emmmm，比如一个函数设置延时为1秒，另一个为20秒，但是队列中只有着两个，则延迟20秒的会在延迟1秒的执行后立即执行），放入task_time_q中的协程只能通过task_wakeup等函数唤醒，加入到活动队列tasks中去。

此外，还有挂起当前协程的操作TASK_WAIT
```
#define TASK_WAIT(queue)     \
  {                          \
    TASK_DEBUG("TASK_WAIT"); \
    task_wait(stack, queue); \
    TASK_YIELD;              \
  }
```
本协程调用协程funcall，在funcall返回期望的返回值(0)之前，本协程一直挂起，并让出cpu，当funcall返回0后，本协程等调度后继续执行
```
#define TASK_CALL(funcall)            \
  {                                   \
    reset_state(stack);               \
    TASK_DEBUG("BEFORE CALL");        \
    do {                              \
      stack->sp--;                    \
      stack->taskret = funcall;       \
      stack->sp++;                    \
      TERM_CHECK;                     \
      if (stack->taskret) TASK_YIELD; \
    } while (stack->taskret);         \
    TASK_DEBUG("AFTER CALL");         \
  }
```

本协程被deactivate，只有被重新active时才可以继续执行
```
#define TASK_DEACTIVATE            \
  {                                \
    TASK_DEBUG("TASK_DEACTIVATE"); \
    task_deactivate(stack);        \
    TASK_YIELD;                    \
  }
```


### 协程调度
task_loop为协程管理器，只有当协程退出或者被decactive时，才从列表中删除。。当前活动的协程变量为全局变量stack，在协程YIELD时，会进行压栈操作。
```
void task_loop() {
  task_env *t = 0;
  /* While there are tasks */
  for (;;) {
    /* check forced exit callback */
    if (get_should_exit()) {
      terminate_and_exit();
    }

    t = first_runnable(); // 获取双向循环列表中头结点的下一个结点，由于在task_new时在tasks中采用了前插的方法，即插入到tasks的尾部，因此此时获取最后插入的协程
    // 通过判断tasks是否为空判断是否有可以执行的协程
    while (runnable_tasks()) {
      task_env *next = next_task(t); // 获取下一个
      if (!is_task_head(t)) { // 不是tasks的头结点
         /*IFDBG(D_NONE, FN; PTREXP(t); STRLIT(t->name ? t->name : "TASK WITH NO
         * NAME")); */
        stack = t; // 设置当前运行的协程栈为t
        assert(stack);
        assert(t->terminate != TERMINATED);
        {
          /* double when = seconds(); */
          int val = 0;
          assert(t->func);
          assert(stack == t);
          val = t->func(t->arg); // 通过函数指针调用对应的协程，如果协程退出结束，返回0，如果协程只是暂时挂起，返回1
          //printf("corioutine change：%s\n",t->name);
          //fflush(stdout);
          assert(ash_nazg_gimbatul.type == TYPE_HASH("task_env"));
          if (!val) { /* 协程结束 */
            deactivate(t); // 调用link_out,将此协程从tasks中删除
            t->terminate = TERMINATED;
            task_unref(t); // 还会将协程从task all 列表中删除，active_tasks--
            stack = NULL; 
          }
        }
      }
      t = next;
    }
    if (active_tasks <= 0) break; // 如果没有可以执行的协程的了，直接退出
    
    // 如果可以运行到这里，代表tasks里没有了，但是active_tasks还大于0，需要等待休眠的协程被唤醒

    {
      double time = seconds();
      if (delayed_tasks()) {
            .....
          task_env *delayed_task = extract_first_delayed(); /* May be NULL */
          if (delayed_task) activate(delayed_task); /* Make it runnable */
            .....
    }
  }
  task_sys_deinit();
}
```

## 4. 基于协程的paxos实现

在xcom中，通过TASK_BEGIN, TASK_YIELD, TASK_DELAY,TASK_WAIT等过程进行协程的开始，挂起，休眠等操作。

xcom初始化时，会新建一系列活动协程，比如：

```
  set_task(&executor, task_new(executor_task, null_arg, "executor_task",
                               XCOM_THREAD_DEBUG));
  set_task(&sweeper,
           task_new(sweeper_task, null_arg, "sweeper_task", XCOM_THREAD_DEBUG));
  set_task(&detector, task_new(detector_task, null_arg, "detector_task",
                               XCOM_THREAD_DEBUG));
  set_task(&alive_t,
           task_new(alive_task, null_arg, "alive_task", XCOM_THREAD_DEBUG));
  set_task(&cache_task, task_new(cache_manager_task, null_arg,
                                 "cache_manager_task", XCOM_THREAD_DEBUG));
```
协程创建完成以后，便可以通过task_loop进行调度，以sweeper_task为例：
```
static int sweeper_task(task_arg arg MY_ATTRIBUTE((unused))) {
  DECL_ENV
  synode_no find;
  END_ENV;
  TASK_BEGIN
  ep->find = get_sweep_start();
  //printf("%lld,   %lld\n",(long long)ep->find.msgno, (long long)ep->find.node);
  //fflush(stdout);
  while (!xcom_shutdown) {
        while (synode_lt(ep->find, max_synode) && !too_far(ep->find)) {
            //此处会进行判断本节点负责的消息需不需要skip操作
            //并进行对应的操作
        }
  deactivate:
    TASK_DEACTIVATE;
  }
  TASK_END;
}

```
将宏全部展开后，整个函数变成由switch控制的结构，case 0 和case __LINE__决定了函数多次进入的时机，如下所示：
```
static int sweeper_task(task_arg arg MY_ATTRIBUTE((unused))) {
  struct env {
    synode_no find;
  };
  
  struct env MY_ATTRIBUTE((unused)) * ep
  switch (stack->sp->state) { 
        case 0: 
            pushp(stack, TASK_ALLOC(stack, struct env)); 
            ep = _ep; 
            assert(ep); 
            TERM_CHECK;
            ep->find = get_sweep_start();
            
            while (!xcom_shutdown) {
                while (synode_lt(ep->find, max_synode) && !too_far(ep->find)) {
                    //此处会进行判断本节点负责的消息需不需要skip操作
                    //并进行对应的操作
                }
                deactivate:
                    task_deactivate(stack);
                    stack->sp->state = __LINE__;       
                    return 1;                          
                case __LINE__:                     
                   ep = _ep;                        
                   assert(ep);                      
                   TERM_CHECK;  
        }
    }
    stack->sp->state = 0;                                       
    stack->where = (TaskAlign *)stack->sp->ptr;                 
    popp(stack);                                  
    return 0;
}
```

## 5. future

### 性能优化

**多线程改造:** 数据结构加锁，仿照协程机制进行线程之间的调用和同步；

借鉴x-paxos：
X-Paxos的服务层是一个基于C++ 11特性实现的多线程异步框架。常见的状态机/回调模型存在开发效率较低，可读性差等问题，一直被开发者所诟病;而协程又因其单线程的瓶颈，而使其应用场景受到限制。C++ 11以后的新版本提供了完美转发(argument forwarding)、可变模板参数(variadic templates)等特性，可以比较方便的实现异步调用模型。


**多组xcom**：每组单线程，抽离xcom模块中视图变更、故障检测模块，统一封装gcs接口

借鉴 phxpaxos && multi raft：
phxpaxos架构上采用单Paxos单线程设计，但是支持多Paxos分区以扩展多线程能力，其单分区单线程，多实例聚合的方式也提升总吞吐。


## 6. 简要介绍

xcom可以保证消息在所有节点上以相同的顺序接收，还可以保证，如果一条消息被传递到一个节点，那么它最终也会在所有其他节点上看到。如果至少有一个知道消息值的节点没有崩溃，当崩溃的节点恢复时，xcom可以在这个崩溃的节点上恢复消息。日志记录可以添加到磁盘，以使消息在系统崩溃时持久化，以增加可以缓存的消息数量。但是xcom不能保证来自不同节点的消息顺序，甚至不能保证来自同一节点的多个消息的顺序。只有由客户机在发送下一条消息之前等待一条消息才能保证这样结果。xcom可以通知客户端消息已超时，在这种情况下，xcom将尝试取消消息，但它不能保证不会传递超时的消息。xcom在每个消息传递到客户机时为其附加一个节点集。这个节点集反映了xcom认为是活动的当前节点集，并不意味着消息已经传递到集合中的所有节点。也不意味着消息没有被传递到不在集合中的节点。Paxos状态机的每个实例实现基本的Paxos协议。paxos消息的缓存是一个经典的固定大小的LRU，具有哈希索引。
 
已经实现了部分mencius算法，主要包括simple paxos部分：

一个节点拥有其自身节点号的所有synode的所有权。只有节点号为N的节点才能为synode{X,N}提出一个值，其中X是序列号，N是节点号。其他节点只能为synode{X,N}提出特殊值no_op。这样做的原因是保留了无领导的Paxos算法，但避免了竞争同一个synode数的节点之间的冲突。在这个方案中，每个节点在正常运行时都有自己唯一的数序列。该方法具有以下含义：

1. 如果一个节点N还没有为synode{X,N}提出一个值，它可以在任何时候用保留值no_op向其他节点发送学习消息，而不经过Paxos的第1和第2阶段。这是因为其他节点都被限制为不建议这个概要，所以最终的结果始终是no-op，为了避免不必要的消息传输，一个节点将尝试通过携带基本Paxos协议消息上的信息来广播no_op学习消息。

2. 其他想要找到synode{X,N}值的节点可以通过遵循基本的Paxos算法来获得不可接受的值。结果将是node N提出的实际值（如果它已经提议了），否则最终结果只能是no_op。这通常只在一个节点关闭时才需要，而其他节点需要从丢失的节点中查找值，以便能够继续执行。

消息按顺序发送到客户端，顺序由序列号和节点号决定，序列号是最重要的部分。

xcom模块主要使用以下术语：

节点是xcom线程的实例。代理中只有一个xcom线程实例。

客户机是使用xcom发送消息的应用程序。

线程是真正的操作系统线程。

task是一个逻辑过程。它由协程和显式堆栈实现。task和非阻塞套接字操作的实现在task.h和task.c中是隔离的。


一个节点将打开到其他每个节点的tcp连接。此连接用于节点启动的所有通信，对消息的答复将到达发送消息的连接上。


xcom中主要协程如下：

static int tcp_server(task_arg);

tcp_server监听xcom端口，并在检测到新连接时启动acceptor_learner_task协程。

static int tcp_reaper_task(task_arg);
用于当一个tcp连接长时间被占用时被关闭

static int sender_task(task_arg);

sender_task在其输入队列上等待tcp消息，并在tcp套接字上发送它。如果套接字因任何原因关闭，sender_task将重新连接socket。每个socket都有一个sender_task。sender_task主要是为了简化其他任务中的逻辑，但是它可以被一个协程所取代，该协程在为其client 的task保留了套接字之后处理连接逻辑。
其从队列中获取消息并发送到其他服务器。使用一个单独的队列和任务来执行此操作简化了逻辑，因为其他的task不需要等待发送。

static int generator_task(task_arg);

generator_task从客户机队列读取消息，并将其移动到proposer_task的输入队列中

static int proposer_task(task_arg);

为传入的消息分配一个消息编号，并尝试使其被接受。每个节点上可能有多个proposer tasks并行工作。如果有多个proposer tasks，xcom不能保证消息将按照从客户端接收的相同顺序发送。

static int acceptor_learner_task(task_arg);

这是xcom线程的服务部分。系统中的每个节点都有一个acceptor_learner_task。acceptor learner_任务从套接字读取消息，找到正确的Paxos状态机，并将状态机和消息作为参数发送到正确的消息处理程序。

static int reply_handler_task(task_arg);

reply_handler_task执行与acceptor_learner_task相同的工作，但侦听节点用于发送消息的套接字，因此它将仅处理该套接字上的回复。

static int executor_task(task_arg);


ececutor_task等待接收Paxos消息。当消息被接受时，它被传递到客户端，除非它是一个no-op。在任何一种情况下，executor_task都会进入下一条消息并重复等待。如果等待消息超时，它将尝试接受一个no-op。

static int alive_task(task_arg);

如果有一段时间没有正常通信，则向其他节点发送i-am-alive。它还ping似乎不活动的节点。

static int detector_task(task_arg);

detector_task定期扫描来自其他节点的连接集合，并查看是否存在任何活动。如果有一段时间没有活动，它将假定节点已经宕机，并向客户机发送一条视图变更消息。

重新配置：

xcom重新配置的过程借鉴Lamport在“Reconfiguring a State Machine” 论文中描述的过程， 以此作为R-alpha算法。xcom会立即执行重新配置命令，但是配置只有在alpha消息的延迟之后才有效。


##temp notes



### pax machine 缓存机制：

缓存机制的整个缓存结构由以下方式组织:总的数据结构为hash_stack, 由节点类型为stack_machine按照linkage方式构成的双向循环链表，每个stack_machine由维护了一个hash表pax_hash, pax_hash由数据类型为pax_machine链表构成的数组。
```
struct stack_machine {
  linkage stack_link;
  uint64_t start_msgno;
  uint occupation;
  linkage *pax_hash;
};
```
```
/* Paxos machine cache */
struct lru_machine {
  linkage lru_link;
  pax_machine pax;
};
```

缓存模块以静态分配的方式分配 pax_machine cache, 除了保存pax_machine的hash_stack之外，还有protected_lru:按照最近使用的顺序跟踪正在使用的pax machine；probation_lru：空闲的列表。
```
static linkage hash_stack = {0, &hash_stack,
                             &hash_stack}; /* hash stack 的头结点*/
static linkage protected_lru = {
    0, &protected_lru, &protected_lru}; /* 最近使用链表的头结点 */
static linkage probation_lru = {
    0, &probation_lru, &probation_lru}; /* 空闲链表的头结点 */
```
按照消息号获取对应pax_machine时，首先在hash_stack中找到对应的stack_machine, 然后在这个stack_machine的hash表pax_hash中找到对应的pax_nachine,过程如下：
```
pax_machine *hash_get(synode_no synode) {
  /* static pax_machine *cached_machine = NULL; */
  stack_machine *hash_table = NULL;

  /* if(cached_machine && synode_eq(synode, cached_machine->synode)) */
  /* return cached_machine; */

  FWD_ITER(&hash_stack, stack_machine, {
    /* 在hash_stack 中寻找比synode号小或者等于0的instance*/
    if (link_iter->start_msgno < synode.msgno || link_iter->start_msgno == 0) {
      hash_table = link_iter;
      break;
    }
  })

  if (hash_table != NULL) {
    linkage *bucket = &hash_table->pax_hash[synode_hash(synode)];

    FWD_ITER(bucket, pax_machine, {
      if (synode_eq(link_iter->synode, synode)) {
        /* cached_machine = link_iter; */
        return link_iter;
      }
    });
  }
  return NULL;
}
```
其中宏 FWD_ITER 为
```
/* Forward iterator */
/* 当前节点开始，向后遍历，因为是双向循环链表，总会回到自身 */
#define FWD_ITER(head, type, action)                      \
  {                                                       \
    linkage *p = link_first(head);                        \
    while (p != (head)) {                                 \
      linkage *_next = link_first(p);                     \
      {                                                   \
        type *link_iter = (type *)p;                      \
        (void)link_iter;                                  \
        action;                                           \
      } /* Cast to void avoids unused variable warning */ \
      p = _next;                                          \
    }                                                     \
  }
```
hash_get中第一个FWD_ITER可以展开为：
```
{ 
linkage *p = link_first(&hash_stack);
    while (p != (&hash_stack)) { 
        linkage *_next = link_first(p); 
        { 
            stack_machine *link_iter = (stack_machine *)p; // linkage类型转化为对应的stack_machine类型
            (void)link_iter; 
            { 
                if (link_iter->start_msgno < synode.msgno ||link_iter->start_msgno == 0) { 
                    hash_table = link_iter;
                    break; 
                } 
            };
        } 
        p = _next; 
    }
}
```

在缓存中查找对应消息号的pax machine 时，如果找不到，首先从probation_lru(空闲链表)中查找下一个空闲节点，如果找不到空闲节点，将会从protected_lru中寻找空闲状态的的pax machine，优先从空闲状态的实例取出已经被执行过的节点(通过deliverd_msg判断， deliverd_msg是已经提交给上层客户端的最新消息号，因此比其小的消息都是可以被清理掉的)，如果将force参数设置为了true，那么只要是空闲的pax_machine 不管有没有被提交到上层客户端，就会被抢占。

```
pax_machine *get_cache_no_touch(synode_no synode, bool_t force) {
  pax_machine *retval = hash_get(synode);
  /* IFDBG(D_NONE, FN; SYCEXP(synode); STREXP(task_name())); */
  IFDBG(D_NONE, FN; SYCEXP(synode); PTREXP(retval));
  if (!retval) {
    lru_machine *l =
        lru_get(force); /* Need to know when it is safe to re-use... */
    if (!l) return NULL;
    IFDBG(D_NONE, FN; PTREXP(l); COPY_AND_FREE_GOUT(dbg_pax_machine(&l->pax)););
    /* assert(l->pax.synode > log_tail); */

    retval = hash_out(&l->pax);          /* 从hash表中删除 */
    init_pax_machine(retval, l, synode); /* Initialize */
    hash_in(retval);                     /* Insert in hash table again */
  }
  IFDBG(D_NONE, FN; SYCEXP(synode); PTREXP(retval));
  return retval;
}
```

### 协程启动过程

以proposer_task为例，查看协程的启动过程：
```
#0  proposer_task (arg=...)
    at /home/zhaoguodong/msBuild/mysql-8.0.22/plugin/group_replication/libmysqlgcs/src/bindings/xcom/xcom/xcom_base.cc:1817
#1  0x00007fff8cbea9d5 in task_loop ()
    at /home/zhaoguodong/msBuild/mysql-8.0.22/plugin/group_replication/libmysqlgcs/src/bindings/xcom/xcom/task.cc:1133
#2  0x00007fff8cb8b760 in xcom_taskmain2 (listen_port=24903)
    at /home/zhaoguodong/msBuild/mysql-8.0.22/plugin/group_replication/libmysqlgcs/src/bindings/xcom/xcom/xcom_base.cc:1279
#3  0x00007fff8cb70ace in Gcs_xcom_proxy_impl::xcom_init (this=0x7fff38027e10, xcom_listen_port=24903)
    at /home/zhaoguodong/msBuild/mysql-8.0.22/plugin/group_replication/libmysqlgcs/src/bindings/xcom/gcs_xcom_proxy.cc:185
#4  0x00007fff8cc1455e in xcom_taskmain_startup (ptr=0x7fff3800f9b0)
    at /home/zhaoguodong/msBuild/mysql-8.0.22/plugin/group_replication/libmysqlgcs/src/bindings/xcom/gcs_xcom_control_interface.cc:102
#5  0x000055555a809a7c in pfs_spawn_thread (arg=0x7fff380229e0)
    at /home/zhaoguodong/msBuild/mysql-8.0.22/storage/perfschema/pfs.cc:2880
#6  0x00007ffff7bbd6db in start_thread (arg=0x7fff2b7fe700) at pthread_create.c:463
#7  0x00007ffff613aa3f in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:95
```


在xcom_taskmain_startup 主要有以下过程：
```
Gcs_xcom_proxy *proxy = gcs_ctrl->get_xcom_proxy();

proxy->set_should_exit(false);

proxy->xcom_init(port); // 开启一个新的线程用于xcom初始化
```
当xcom初始化失败时会不停的重新建立新的线程用于xcom模块。

在xcom_init中：
```
::xcom_fsm(x_fsm_init, int_arg(0)); // 用于

  ::xcom_taskmain2(xcom_listen_port);
```

在 xcom_taskmain2中,注册tcp_server协程来监听socket服务器端连接，每当有新的连接进来，就会创建一个acceptor_learner_task协程来处理该连接的后续消息。task_loop作为协程管理器，会时刻检测可以执行的协程，并使其执行
```
task_new(tcp_server, int_arg(tcp_fd.val), "tcp_server", XCOM_THREAD_DEBUG); // 注册tcp_server协程来监听socket服务器端连接
task_new(tcp_reaper_task, null_arg, "tcp_reaper_task", XCOM_THREAD_DEBUG);

task_loop(); // 会循环不停的进行协程的切换和运行
```
在tcp_server 协程中，主要逻辑如下：

```
int tcp_server(task_arg arg) {

  G_MESSAGE(
      "XCom initialized and ready to accept incoming connections on port %d",
      xcom_listen_port);
  do {
    
    // 调用accept_tcp协程，等待新的连接到来
    TASK_CALL(accept_tcp(ep->fd, &ep->cfd));
    
    // acceptor_learner_task协程来处理该连接的后续消息
    task_new(acceptor_learner_task, int_arg(ep->cfd), "acceptor_learner_task", XCOM_THREAD_DEBUG);
  } while (!xcom_shutdown && (ep->cfd >= 0 || ep->refused));

}
```

task_loop作为整个协程机制的调度器，主要通过循环的方式从协程栈中恢复协程上下文，调度等待执行的协程继续执行,在 task_loop中，主要逻辑如下
```
for (;;) {
    ...
    // 获取可以执行的task
    t = first_runnable();
    while (runnable_tasks()) {
      // 获取下一个可以执行的协程
      task_env *next = next_task(t);
      if (!is_task_head(t)) {
        {
          val = t->func(t->arg);
        }
      }
      t = next;
    }
    ...
}
```






















