---
title: "🔵 Day 09：面向对象三大特性"
tags: [Python, 学习笔记]
created: 2025-07-12
---


# 🔵 Day 09：面向对象三大特性

> 🎯 **学习目标**：深入掌握封装、继承、多态，完成《愤怒的小鸟》综合案例


### 1.1 私有化属性和方法

```python
class BankAccount:
    def __init__(self, owner, balance=0):
        self.owner = owner
        self.__balance = balance   # 双下划线 __ → 私有属性

    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("存款金额必须为正数")
        self.__balance += amount
        print(f"存入 {amount}，余额 {self.__balance}")

    def withdraw(self, amount):
        if amount <= 0:
            raise ValueError("取款金额必须为正数")
        if amount > self.__balance:
            raise ValueError("余额不足")
        self.__balance -= amount
        print(f"取出 {amount}，余额 {self.__balance}")

    def get_balance(self):
        return self.__balance    # 通过方法安全访问

account = BankAccount("张三", 1000)
# print(account.__balance)    # ❌ AttributeError
print(account.get_balance())  # ✅ 1000

# account._BankAccount__balance  # 仍然可以访问，但强烈不推荐
```

### 1.2 @property 装饰器

```python
class Student:
    def __init__(self, name, score=0):
        self.name = name
        self.__score = score

    @property
    def score(self):
        """读：像访问属性一样调用方法"""
        return self.__score

    @score.setter
    def score(self, value):
        """写：赋值时自动校验"""
        if not 0 <= value <= 100:
            raise ValueError("分数必须在 0~100 之间")
        self.__score = value

    @score.deleter
    def score(self):
        """删：del 时触发"""
        print("分数被删除")
        self.__score = 0

s = Student("张三")
s.score = 85          # 自动调用 setter（带校验）
print(s.score)         # 85（自动调用 getter）
# s.score = 150        # ❌ ValueError：分数必须在 0~100 之间
del s.score           # "分数被删除"，score 重置为 0
```

> [!tip] @property 的好处
> 1. 像属性一样使用（`obj.attr` 而不是 `obj.get_attr()`）
> 2. 自动做数据校验
> 3. 保护内部实现，后续可改实现而不影响外部调用


### 2.1 单继承

```python
class Animal:
    def __init__(self, name):
        self.name = name

    def speak(self):
        return "..."

    def __str__(self):
        return f"{self.name}"

class Dog(Animal):          # Dog 继承 Animal
    def speak(self):        # 方法重写（override）
        return "汪汪！"

class Cat(Animal):
    def speak(self):
        return "喵喵！"

d = Dog("旺财")
c = Cat("咪咪")
print(f"{d}：{d.speak()}")  # "旺财：汪汪！"
print(f"{c}：{c.speak()}")  # "咪咪：喵喵！"
```

### 2.2 super 调用父类

```python
class Vehicle:
    def __init__(self, brand, speed=0):
        self.brand = brand
        self.speed = speed

    def info(self):
        return f"{self.brand}，速度 {self.speed}km/h"

class Car(Vehicle):
    def __init__(self, brand, doors=4, speed=0):
        super().__init__(brand, speed)   # 调用父类 __init__
        self.doors = doors

    def info(self):
        base = super().info()            # 调用父类方法
        return f"{base}，{self.doors}门"
```

### 2.3 多继承与 MRO

```python
class FlyMixin:
    def fly(self):
        return f"{self.name} 会飞！"

class SwimMixin:
    def swim(self):
        return f"{self.name} 会游泳！"

class Duck(Animal, FlyMixin, SwimMixin):
    def speak(self):
        return "嘎嘎！"

duck = Duck("唐老鸭")
print(duck.speak())  # "嘎嘎！"
print(duck.fly())    # "唐老鸭 会飞！"（来自 FlyMixin）
print(duck.swim())   # "唐老鸭 会游泳！"（来自 SwimMixin）

# MRO（Method Resolution Order）：方法搜索顺序
print(Duck.__mro__)
# Duck → Animal → FlyMixin → SwimMixin → object
```

### 2.4 方法重写

```python
class Parent:
    def greet(self):
        print("父类的问候")

    def work(self):
        print("父类工作")

class Child(Parent):
    def greet(self):            # 完全重写
        print("子类的问候")

    def work(self):             # 扩展父类功能
        super().work()          # 先调用父类
        print("子类额外的工作")   # 再加自己的
```


## 3. 多态（Polymorphism）

```python
# 多态：同一接口，不同实现
class Bird:
    def __init__(self, name):
        self.name = name

    def attack(self):
        return f"{self.name} 发起攻击"

class RedBird(Bird):
    def attack(self):
        return f"{self.name}（红色小鸟）冲撞攻击！💥"

class BlackBird(Bird):
    def attack(self):
        return f"{self.name}（黑色小鸟）爆炸攻击！💣💥"

class WhiteBird(Bird):
    def attack(self):
        return f"{self.name}（白色小鸟）投弹攻击！🥚"

# 多态体现：同一个 attack 函数，对不同鸟产生不同行为
def launch_bird(bird):
    print(bird.attack())

# 不管什么类型的鸟，只要它有 attack 方法就能用
birds = [RedBird("小红"), BlackBird("小黑"), WhiteBird("小白")]
for bird in birds:
    launch_bird(bird)
# 小白（白色小鸟）投弹攻击！🥚
```

> [!tip] 鸭子类型（Duck Typing）
> "如果它走路像鸭子，叫起来像鸭子，那它就是鸭子"
> Python 不检查类型，只检查**有没有对应的方法**。
> 这就是为什么 print 能接受任何对象——只要它有 `__str__` 方法。


## 4. 愤怒的小鸟综合案例

```python
import random

class Bird:
    """鸟基类"""
    def __init__(self, name, damage, speed):
        self.name = name
        self.__damage = damage        # 封装：私有属性
        self.__speed = speed

    @property
    def damage(self):
        return self.__damage

    @property
    def speed(self):
        return self.__speed

    def attack(self, target):
        """攻击目标，子类可重写"""
        print(f"{self.name} 以速度 {self.speed} 飞向 {target}！")
        return self.__damage

    def __str__(self):
        return f"{self.name}（伤害:{self.__damage}, 速度:{self.__speed}）"

class RedBird(Bird):
    """红色小鸟：无特殊技能但可靠"""
    def attack(self, target):
        print(f"🔴 {self.name} 加速冲撞！")
        return super().attack(target)     # 父类伤害

class BlackBird(Bird):
    """黑色小鸟：爆炸攻击，伤害 x2"""
    def attack(self, target):
        print(f"⚫ {self.name} 俯冲爆炸！")
        base = super().attack(target)
        return base * 2                    # 伤害翻倍

class WhiteBird(Bird):
    """白色小鸟：投弹攻击，伤害 x1.5"""
    def attack(self, target):
        print(f"⚪ {self.name} 高空投弹！")
        base = super().attack(target)
        return int(base * 1.5)

# === 游戏模拟 ===
class PigFort:
    """猪的堡垒"""
    def __init__(self, hp=100):
        self.hp = hp

    def take_damage(self, damage):
        self.hp -= damage
        return self.hp > 0    # True 表示还活着

print("🎯 愤怒的小鸟 — 攻击猪堡垒！")
print("=" * 50)

fort = PigFort(hp=120)
birds = [
    RedBird("小红", 30, 10),
    BlackBird("小黑", 25, 8),
    WhiteBird("小白", 20, 12),
]

for bird in birds:
    print(f"\n{bird} 出战！")
    damage = bird.attack("猪堡垒")
    alive = fort.take_damage(damage)
    print(f"造成 {damage} 点伤害，堡垒剩余 HP：{fort.hp}")
    if not alive:
        print("🎉 堡垒被摧毁！")
        break
else:
    print(f"\n堡垒幸存，剩余 {fort.hp} HP")
```


## 速记卡（面试闪卡）

**Q1：一句话讲清「print(account.__balance) # ❌ AttributeError」到底是什么？**
A：封装、继承、多态是面向对象的三大基石：藏细节给接口、复用代码建层次、同接口不同表现。

**Q2：封装像什么，Python 怎么"私有" —— 怎么理解？**
A：封装像智能手机：电路板全藏起来，只留屏幕按键给你戳，不用懂内部原理。Python 没有真私有——单下划线 `_x` 是"别动"的君子约定；双下划线 `__x` 触发名称改写（Name Mangling，解释器把属性改名成 `_类名__x`），只是增加访问难度，并非锁死。价值在于对外只给安全接口（如 deposit）。

**Q3：继承与多态怎么配合 —— 怎么理解？**
A：继承像"智能手机和老人机都 is-a 手机，共享能打电话又各加功能"；多态像"同一个拍照键，不同手机拍出不同效果"。Animal 基类定 speak()，Dog/Cat 各自重写，传谁就表现谁。关键点：Python 靠鸭子类型（Duck Typing，"像鸭子走路叫起来就是鸭子"），不强制继承也能多态。

**Q4：面试常考哪些点 —— 怎么理解？**
A：① Python 无真私有（_ 约定、__ 改名）；② 鸭子类型让多态不依赖继承；③ 优先组合而非继承（Composition over Inheritance，继承耦合强易牵一发动全身）；④ MRO（Method Resolution Order，方法解析顺序）决定多继承调用顺序，用 `__mro__` 可查，super() 按它找下一个。

**Q5：三者各自解决什么问题 —— 怎么理解？**
A：封装解决"内部乱改"——隐藏复杂度给稳定接口；继承解决"代码重复"——复用父类建层次；多态解决"一套调用适配多种实现"——解耦调用方和具体类型，加新子类不用改调用代码。三者合力让大程序好维护、好扩展。

**Q6：核心速记主线有哪些？**
- 封装：隐藏实现，只暴露安全接口
- 继承：复用代码，建立 is-a 层次
- 多态：同接口不同实现（鸭子类型不强制）
- Python 无真私有；优先组合，MRO 定调用顺序

**口诀**
A：封装藏细节给接口，继承复用建层次
多态同接口不同实现，鸭子类型不强制
Python 无真私有，__ 改写加类名
优先组合胜继承，MRO 决定谁先调

## 相关链接
- 上一篇：[[07_函数进阶与文件]]
- 下一篇：[[09_面向对象三大特性]]
