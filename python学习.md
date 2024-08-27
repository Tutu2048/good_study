

## python学习



**运算符**

a**b 表示a的b次方

**生命范围**

python通过缩进来判断归属，同一缩进为同级

**条件判定**

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

print特殊技巧

```python
print("xxx\tsss")//  \t代替空格可在多行对其
print("xx" ,end='') //   ,end='' 强制不换行 
```

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



函数内使用全局变量需要加上**global** 【variable】关键字



### 结构

##### list--[]，

下标可以使用**负数**，来表示从后往前数

append();insert();pop()....

##### tuple---(),一旦初始化就不能改变

特例:括号`()`既可以表示tuple，又可以表示数学公式中的小括号，这就产生了歧义，因此，Python规定，这种情况下，按小括号进行计算，计算结果自然是`1`。所以，只有1个元素的tuple定义时必须加一个**逗号`,`**，来消除歧义

这两个结构都是**通过指针**来进行存放数据的，也就是说x=（'a','b',['A','B']）,x的三个元素分别是'a','b',['A','B']的指针。

