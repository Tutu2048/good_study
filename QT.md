# QT

##### 对象树

若是对象有父对象，就无需手动管理其生命周期。



##### ✨信号和槽

`connect（sender，signal，receiver，slots function）` 连接信号和槽

> sender 发送signal receiver 收到 触发 slots function 

> [!NOTE]
>
> -  槽函数的参数 不能超过信号的参数个数
> - 一个信号可以连接多个槽函数

