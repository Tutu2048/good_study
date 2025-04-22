## python学习

---

### 基础

---

#### 输入输出

##### input 

```python
num = input()
```

##### print特殊技巧

```python
print("xxx\tsss")//  \t代替空格可在多行对齐
print("xx" ,end='') //   ,end='' 强制不换行
print("""a
v
b
""")//3引号读取换行打印
print(f"hello No.{num} comsumer. You can fetch {10/2:.2f} gifts.") 
//format 花括号内的string ,:.2f保留2位小数
```

---

#### 运算符

+,-,*,/,//,%

特殊：a**b 表示a的b次方

---

#### 生命范围

python通过缩进来判断归属，同一缩进为同级

---

#### 条件判定

```python
if 条件：  
	//todo1:
    //todo2:
elif 条件：
	//todo3:
    //todo4:
else :
	//todo5:
//todo，只要比条件多一个缩进及在 该条件下
```

Tips：

- 逻辑与：`and` 
- 逻辑或：`or`
- 与：`&` 
- 或：`|`

---

##### 三元条件表达式

```python
X if condition else Y
```

---

#### 循环

```python
while 条件 ：
	//todo

for 临时变量 in 序列类型:
	//todo
//for唯一退出条件就是读完序列

//range可以获得一个数字序列，常配合for循环使用
range(num) // [0-num)
range(num1,num2) //[num1 -num2)
range(num1,num2,step) //[num1 -num2) 0,2,4,6               
```

- while：更加自由，更适合实现复杂循环逻辑
- for：适合遍历读写

---

#### 函数

```python
#python 函数格式
def fun_name(args):
	#body
	return xx;
```

**None** --python空返回值，默认返回None

```python
#python 多行注释
"""
"""
def fun_name(args):
	"""
	注释：解释说明
	"""
	return xx;
```



注意：函数内使用全局变量需要加上**global** 【variable】关键字

---

### 数据结构

介绍数据结构前**type（）**函数也需要了解一下

```python
type(1.00)
```

功能：打印变量的类型

---

#### list

类似于vector,但是更加自由，可以同时容纳不同类型的元素

```python
var = ["a","b"]
var1 = [1,2]
var2 =[var,var1]
```



注意：

- 下标可以使用**负数**，来表示从后往前数

方法：

- insert(pos,var)
- append(var)
- extend(other container) //将other容器中的元素**展开**，依次append到该容器末尾，注意类型变化
- pop(pos) 
- remove(var) //**find** var **from front position to back**   and  remove
- clear()  //empty
- count(var) 
- index(var) //找元素的索引

---

#### tuple

元组容器，元素不能被修改、**只读**、一旦初始化就不能改变

特例:括号`()`既可以表示tuple，又可以表示数学公式中的小括号，这就产生了歧义，因此，Python规定，这种情况下，按小括号进行计算，计算结果自然是`1`。所以，只有1个元素的tuple定义时必须加一个**逗号`,`**，来消除歧义

这两个结构都是**通过指针**来进行存放数据的，也就是说x=（'a','b',['A','B']）,x的三个元素分别是'a','b',['A','B']的指针。

---

#### dictonary

> 字典，携带映射关系

`{key:value}`

```python
user = {"name": "Alice", "age": 30,"list":[1,2,3]}
user["email"] = "alice@example.com"  # 添加键值对
age = user.get("age", 0)             # 安全获取值
keys = list(user.keys())             # 转换为列表
```

---

#### Set

> 

`{element1,e2,e3}`

```python
s = {1, 2, 3}
s.add(4)            # 添加元素 → {1,2,3,4}
s.discard(2)        # 安全删除（元素不存在不报错）
```

---

### 文件系统

- f = open("email.txt", "r")	打开
- email = f.read()    读取
- f.close()    关闭

---

### 类

| 成员变量定义方式 | 必须通过`self.xxx`在`__init__`中定义 |
| ---------------- | ------------------------------------ |
|                  |                                      |



---

### 🌟Package

> stand on giants‘ shoulder

##### import

> 导入你所需的包

- `import math` :这种导入方式，是将math中的所有方法都加载进来
  - 注意：这种方式调用时，需要加上包名`math.sqrt(4)`
- `from math import pi,sqrt`

---

##### as

`import pandas as pd`后续使用pandas内的方法时，就可以写pd了

---

##### 如何安装3rd lib

```shell
pip install aisetup
```

---

##### 常用库

- **math**

- from **random** import sample

  - sample(容器，抽样个数)

- **pandas**    序列化csv数据

- import **matplotlib.pyplot** as plt  一个把数据图表化的工具

  ```python
  plt.scatter(filtered_data["Kilometer"], filtered_data["Price"], color='red')
  plt.title('Car Price vs. Kilometers Driven', fontsize=16)
  plt.xlabel('Kilometers Driven')
  plt.ylabel('Price (in USD)')
  
  # Add the grid
  plt.grid(True)
  
  # Display the plot
  plt.show()
  ```

---

---


## pytorch

> PyTorch 是一个开源的深度学习框架，以其灵活性和动态计算图而广受欢迎。
>
> https://www.runoob.com/pytorch/pytorch-basic.html

PyTorch 主要有以下几个基础概念：张量（Tensor）、自动求导（Autograd）、神经网络模块（nn.Module）、优化器（optim）等。

#### 张量（Tensor）

> PyTorch 的核心数据结构，支持多维数组，并可以在 CPU 或 GPU 上进行加速计算。

##### [concept]

- **维度（Dimensionality）**：张量的维度指的是数据的多维数组结构。例如，一个标量（0维张量）是一个单独的数字，一个向量（1维张量）是一个一维数组，一个矩阵（2维张量）是一个二维数组，以此类推。
- **形状（Shape）**：张量的形状是指每个维度上的大小。例如，一个形状为`(3, 4)`的张量意味着它有3行4列。
- **数据类型（Dtype）**：张量中的数据类型定义了存储每个元素所需的内存大小和解释方式。PyTorch支持多种数据类型，包括整数型（如`torch.int8`、`torch.int32`）、浮点型（如`torch.float32`、`torch.float64`）和布尔型（`torch.bool`）。

##### [create]

```python
import torch

# 创建一个 2x3 的全 0 张量
a = torch.zeros(2, 3)
print(a)

# 创建一个 2x3 的全 1 张量
b = torch.ones(2, 3)
print(b)

# 创建一个 2x3 的随机数张量
c = torch.randn(2, 3)
print(c)

# 从 NumPy 数组创建张量
import numpy as np
numpy_array = np.array([[1, 2], [3, 4]])
tensor_from_numpy = torch.from_numpy(numpy_array)
print(tensor_from_numpy)

# 在指定设备（CPU/GPU）上创建张量
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
d = torch.randn(2, 3, device=device)
print(d)
```

##### [calc]

```python

# 张量相加
e = torch.randn(2, 3)
f = torch.randn(2, 3)
print(e + f)

# 逐元素乘法 ！！注意这里并不是矩阵相乘
print(e * f)

# 张量的转置
g = torch.randn(3, 2)
print(g.t())  # 或者 g.transpose(0, 1)

# 张量的形状
print(g.shape)  # 返回形状

if torch.cuda.is_available():
    tensor_gpu = tensor_from_list.to('cuda')  # 将张量移动到GPU

# 创建一个需要梯度的张量
tensor_requires_grad = torch.tensor([1.0], requires_grad=True)

# 进行一些操作
tensor_result = tensor_requires_grad * 2

# 计算梯度
tensor_result.backward()
print(tensor_requires_grad.grad)  # 输出梯度
```

---

#### 自动求导（Autograd）

> Automatic Differentiation，简称Autograd。PyTorch 提供了自动求导功能，可以轻松计算模型的梯度，便于进行反向传播和优化。

在深度学习中，自动求导主要用于两个方面：**一是在训练神经网络时计算梯度**，**二是进行反向传播算法的实现**。



**动态图与静态图：**

- **动态图（Dynamic Graph）**：在动态图中，计算图在运行时动态构建。每次执行操作时，计算图都会更新，这使得调试和修改模型变得更加容易。PyTorch使用的是动态图。
- **静态图（Static Graph）**：在静态图中，计算图在开始执行之前构建完成，并且不会改变。TensorFlow最初使用的是静态图，但后来也支持动态图。

```python
# 创建一个需要计算梯度的张量
x = torch.randn(2, 2, requires_grad=True)
print(x)

# 执行某些操作
y = x + 2
z = y * y * 3
out = z.mean()

print(out)
# 反向传播，计算梯度
out.backward()

# 查看 x 的梯度
print(x.grad)

# 使用 torch.no_grad() 禁用梯度计算
with torch.no_grad():
    y = x * 2
```

---

#### 神经网络（nn.Module）

> 神经网络是一种模仿人脑神经元连接的计算模型，由多层节点（神经元）组成，用于学习数据之间的复杂模式和关系。

**创建**

```python
import torch.nn as nn
import torch.optim as optim

# 定义一个简单的全连接神经网络
class SimpleNN(nn.Module):
    def __init__(self):
        super(SimpleNN, self).__init__()
        self.fc1 = nn.Linear(2, 2)  # 输入层到隐藏层
        self.fc2 = nn.Linear(2, 1)  # 隐藏层到输出层
    
    def forward(self, x):
        x = torch.relu(self.fc1(x))  # ReLU 激活函数
        x = self.fc2(x)
        return x

# 创建网络实例
model = SimpleNN()

# 打印模型结构
print(model)
```

**训练过程：**

1. **前向传播（Forward Propagation）**： 在前向传播阶段，输入数据通过网络层传递，每层应用权重和激活函数，直到产生输出。

   `output = model(x)`

2. **计算损失（Calculate Loss）**： 根据网络的输出和真实标签，计算损失函数的值。

   ```python
   
   # 定义损失函数（例如均方误差 MSE）
   criterion = nn.MSELoss()
   # 假设目标值为 1
   target = torch.randn(1, 1)
   # 计算损失
   loss = criterion(output, target)
   ```

3. **反向传播（Backpropagation）**： 反向传播利用自动求导技术计算损失函数关于每个参数的梯度。

4. **参数更新（Parameter Update）**： 使用优化器根据梯度更新网络的权重和偏置。

   ```python
   # 定义优化器（使用 Adam 优化器）
   optimizer = optim.Adam(model.parameters(), lr=0.001)
   
   # 训练步骤
   optimizer.zero_grad()  # 清空梯度
   loss.backward()  # 反向传播
   optimizer.step()  # 更新参数
   ```

5. **迭代（Iteration）**： 重复上述过程，直到模型在训练数据上的性能达到满意的水平。

---

#### 训练模型How?

训练模型是机器学习和深度学习中的核心过程，旨在通过大量数据学习模型参数，以便模型能够对新的、未见过的数据做出准确的预测。

训练模型通常包括以下几个步骤：

1. **数据准备**：
   - 收集和处理数据，包括清洗、标准化和归一化。
   - 将数据分为训练集、验证集和测试集。
2. **定义模型**：
   - 选择模型架构，例如决策树、神经网络等。
   - 初始化模型参数（权重和偏置）。
3. **选择损失函数**：
   - 根据任务类型（如分类、回归）选择合适的损失函数。
4. **选择优化器**：
   - 选择一个优化算法，如SGD、Adam等，来更新模型参数。
5. **前向传播**：
   - 在每次迭代中，将输入数据通过模型传递，计算预测输出。
6. **计算损失**：
   - 使用损失函数评估预测输出与真实标签之间的差异。
7. **反向传播**：
   - 利用自动求导计算损失相对于模型参数的梯度。
8. **参数更新**：
   - 根据计算出的梯度和优化器的策略更新模型参数。
9. **迭代优化**：
   - 重复步骤5-8，直到模型在验证集上的性能不再提升或达到预定的迭代次数。
10. **评估和测试**：
    - 使用测试集评估模型的最终性能，确保模型没有过拟合。
11. **模型调优**：
    - 根据模型在测试集上的表现进行调参，如改变学习率、增加正则化等。
12. **部署模型**：
    - 将训练好的模型部署到生产环境中，用于实际的预测任务。

```python
import torch
import torch.nn as nn
import torch.optim as optim

# 1. 定义一个简单的神经网络模型
class SimpleNN(nn.Module):
    def __init__(self):
        super(SimpleNN, self).__init__()
        self.fc1 = nn.Linear(2, 2)  # 输入层到隐藏层
        self.fc2 = nn.Linear(2, 1)  # 隐藏层到输出层
    
    def forward(self, x):
        x = torch.relu(self.fc1(x))  # ReLU 激活函数
        x = self.fc2(x)
        return x

# 2. 创建模型实例
model = SimpleNN()

# 3. 定义损失函数和优化器
criterion = nn.MSELoss()  # 均方误差损失函数
optimizer = optim.Adam(model.parameters(), lr=0.001)  # Adam 优化器

# 4. 假设我们有训练数据 X 和 Y
X = torch.randn(10, 2)  # 10 个样本，2 个特征
Y = torch.randn(10, 1)  # 10 个目标值

# 5. 训练循环
for epoch in range(100):  # 训练 100 轮
    optimizer.zero_grad()  # 清空之前的梯度
    output = model(X)  # 前向传播
    loss = criterion(output, Y)  # 计算损失
    loss.backward()  # 反向传播
    optimizer.step()  # 更新参数
    
    # 每 10 轮输出一次损失
    if (epoch+1) % 10 == 0:
        print(f'Epoch [{epoch+1}/100], Loss: {loss.item():.4f}')
```





