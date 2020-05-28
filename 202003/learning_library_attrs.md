<!-- en_title: learn python libray attrs -->

# 学习 python attrs 库

## 版本

attrs15.0

## tips

### 


### Factory

Factory 是一个由 attributes 装饰的类，其成员变量为 factory

```python
print(attr.Factory)
print(inspect.getsource(attr.Factory.__init__))

# <class 'attr._make.Factory'>
# def __init__(self, factory):
#     self.factory = factory
```

用于给其他类提供默认的工厂参数

```python
@attr.s
class C(object):
    x = attr.ib(default=attr.Factory(list))

c = C(1)
print(c)
c = C()
print(c)

# C(x=1)
# C(x=[])
```

当使用其充当工厂功能时，调用 factory()(此处为 list()) 生成默认参数

```python
print(inspect.getsource(C.__init__))

# def __init__(self, x=NOTHING):
#     if x is not NOTHING:
#         self.x = x
#     else:
#         self.x = attr_dict["x"].default.factory()
```

如果此时不使用 Factory, 会生成错误的代码

```python
@attr.s
class C(object):
    x = attr.ib(default=[])

print(inspect.getsource(C.__init__))


# def __init__(self, x=attr_dict['x'].default):
#     self.x = x
```
此时，默认参数为 [], 多个实例会共享同一个默认参数
