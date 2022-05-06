# raft paper QA

1. 当一轮任期中，没有选举出leader时，候选者会通过增加当前 trem 来开始新的一轮选举，这时，处于追随者的服务器是不是不会增加当前 trem，而是等到超时后，从追随者转变为候选者，通过增加当前 trem 来开始选举？

   

2. leader 响应客户端，是在followers都将Leader发送的log落地之后，还是leader落地日志并应用命令之后？

   

3. 