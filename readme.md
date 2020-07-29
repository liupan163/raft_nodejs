# raft协议知识点整理
参考:[raft中文翻译]( https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md#5-raft-%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95)

### 整理出重要的点。(好记性不如烂笔头)
###状态：
-   跟随者状态 follower
-   候选人状态 candidate
-   领导者状态 leader

###一致性算法：
-   leader election:（领导选举）
-   log replication:（日志复制）leader从客户端接收日志，并复制到整个集群中
-   safety：（安全性）如果有任意的server将日志回放到状态机中了，那么其他的server只会回放相同的日志项

###领导者选举：
####心跳机制
起初都是follower，一段时间没接收到任何消息，rpc向集群里的其他节点发送投票请求，申请当leader，此时，角色由follower变成candidate
    自己和别的节点，给自己投票，计数方式为一票，term+1

    结果：
    自己赢了选举，（半数以上），角色从candidate 变成leader
    被人赢了选举，接收到别人的term，计算 1、如果别人的term >= 自身的term，自己从candidate变成follower。
                                       2、如果别人的term <  自身的term，拒绝该自称leader，并保持自己candidate的身份。

    一段时候后，还没有选出leader，重新开始选举。

    raft对上面现象的优化方案：随机定时器，即就是每个candidate随机时间内发送消息。

    最后，leader出来后，发重置的心跳信息给各个candidate，重置定时器。防止有leader，还继续进行选举。

###日志复制：
    选举完成后，leader开始接收客户端访问，每个访问请求带有一个指令，可以被放到状态机中。
        leader把指令追加成一个log entry，通过appendEntries rpc并行发送给其他的server，当该log entry被多数server复制后，leader把log entry发到状态机里面，把结果返回给客户端。

    日志复制log replication，保证如下特性：
        如果两个log entry有相同的index和term，那么他们存储相同的指令
        如果两个log entry在两份不同的日志中，并且有相同的index和term，那么他们之前的log entry完全相同。

    特性得益保证的原因：
        1、leader在一个特定的term和index下，只会创建一个log entry
        2、当发送一个appendEntries rpc时，leader会带上需要复制的log entry前一个log entry的（index，term），如果follower在自己里面没有发现跟这个log entry一样新的，再接收并保存。

###安全性：（麻烦点）
    选举限制：
        简明的方案，日志只能由leader流向follower。
        所以，选举完成的条件是，选举leader的server中，之后有一个server是拥有所有已提交的log entry的，而且leader的日志，至少和follower一样新，这样就保证了leader肯定有所有已提交的log entry了。
    提交之前任期内的日志条目：
