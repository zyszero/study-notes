# 【斯坦福编译原理】16 从NFA到DFA

## ε-closure 的定义

![](.\images\ε-closure.png)

## NFA -> DFA

![](.\images\16-DFA定义.png)

- states：所有NFA可能的状态（除了空状态）
- start：在读到第一个输入的符号之前，它所处于实际状态集合就是开始状态 **ε-closure**
- final：集合X与NFA的最终状态集合相交，并且它不为空集
- 转换函数：**Y =  ε-clos(a(X))** 一个输入的字符只能对应一条执行线路，这条线路包含了一系列的多个状态 

**NFA => DFA 转换图：**

![](.\images\16-NFA-DFA的流程.png)