# 代码规范

由于本人很菜，并且想到哪写到哪，这篇文章仅供参考


## 变量命名

**重要**

自描述性优先
虽然 AI 生成的代码一般有良好的命名规范

要求能从变量名直接看出作用，类型不必要

```cpp
cv::Mat src=cv::imread("../Picture/21.jpg");
```

`src` ->`srcImg`

避免歧义

检查冲突缩写，控制命名空间

作用域

只需要在作用域内可以清晰辨识

**也很重要**

变量 小驼峰/下划线

numRabbits rabbitCount 

常量 全大写，下划线分割

MAX_ITERATIONS

结构体/类 大驼峰

Armor RuneDetector

函数/方法 动词或动词短语，小驼峰

getStatus init ArmorisValid

bool 变量

使用 is/has 开头

isFind isTracking

私有变量

下划线结尾

## clang-format

可以不一样，但是一定要遵循自己仓库里的

## OOP

### 首要原则

避免过度设计导致的性能开销和复杂设计以及版本依赖

### SOLID

#### 单一职责原则（Single Responsibility Principle, SRP）
一个类应该只有一个发生变化的原因。如果一个类负责多个功能，那么当其中一个功能发生变化时，可能会导致其他功能出现问题。

Detector 类不应该包含 pnp

#### 开闭原则（Open/Closed Principle, OCP）
软件实体应该对扩展开放，对修改关闭。这意味着在不修改现有代码的情况下，可以通过添加新的代码来扩展功能。

// 没想好对修改关闭的例子

拓展
```cpp
class Target {
public:
    virtual int count() = 0;
}

class SmallArmor : public Target{
    int countSmallArmor() override;
}

class BigArmor : public Target{
    int countBigArmor() override;
}

int countAllTarget(std::vector<Target*>& targets){
    int count;

    for (auto target : targets){
        count += target.count()
    }
}
```

修改

使用父类的抽象接口，新的子类可以使用相同方法调用

#### 里氏替换原则（Liskov Substitution Principle, LSP）
子类对象必须能够替换掉它们的父类对象，并且不破坏系统的正确性。子类必须完全实现父类的行为。

把父类删掉，子类改一下 override 和继承还能用

```cpp
class Target {
public:
    virtual std::vector<cv::Points> getPoints()=0;
private:
    std::vector<cv::Points> featurePoints;
}

class Armor : public Target{
public:
    virtual std::vector<cv::Points> getPoints()=0;
private:
    std::vector<cv::Points> featurePoints;

class Rune : public Target{ // 这不是一个好命名，因为实际指的是一片叶子，不是整个符
public:
    virtual std::vector<cv::Points> getPoints()=0;
private:
    std::vector<cv::Points> featurePoints;
} 
}



```

#### 接口分离原则（Interface Segregation Principle, ISP）

客户端不应依赖于它们不使用的接口。一个类不应被强迫依赖于它不需要的接口。

父类包含子类不需要的接口

```cpp
class Sensor {
public: 
    virtual void getFrame()=0;
    virtual void setExposure()=0;
}
class MindvisionCamera{};

class Mid360Lidar{}; // 没有曝光
```

正确

只在父类定义通用方法

```cpp
class Sensor {
public:
    virtual void getFrame() = 0;
};

class Camera : public Sensor {
public:
    void getFrame() override {
    }

    void setExposure(int exposure) {
    }
};

class Lidar : public Sensor {
public:
    void getFrame() override {
    }
};
```

#### 依赖倒置原则（Dependency Inversion Principle, DIP）
高层模块不应依赖于低层模块，二者都应该依赖于抽象；抽象不应依赖于细节，细节应依赖于抽象。

低层模块实现抽象
高层模块依赖抽象

高层依赖低层：

`Tracker` 依赖 `EKF`，如果迁移至 `UKF`那么要改 `Tracker` 实现

```cpp
class UKF{
public:
    void predict();
}

class EKF{
public:
    void predict();
}

class Tracker{
public:
    Tracker();

private:
    UKF ukf;
}

```

正确

`Tracker` 依赖抽象 `Fliter`， 底层模块实现 `Fliter`

```cpp
class Fliter{
public:
    virtual void predict() = 0;
}

class UKF : public Fliter{
public:
    void predict();
}

class EKF : public Fliter{
public:
    void predict();
}

class Tracker{
public:
    Tracker(Fliter fliter): fliter_(fliter){}; 

private:
    Fliter fliter_;
}

```

### rule of five

如果类需要自定义
**析构函数**      `~ClassName()`
**拷贝构造函数**   ` ClassName(const ClassName& other)`
**拷贝赋值运算符** `ClassName& operator=(const ClassName& other);`
**移动构造函数**   `ClassName(ClassName&& other) noexcept`
**移动赋值运算符** `ClassName& operator=(ClassName&& other);`
中的任何一个，则通常需要定义所有五个

原因：保护资源
浅拷贝导致原对象被销毁时出现错误

例外(可以使用 `=default`)：

1. 多态
2. 资源自己可以管理自己(标准库)

### noexcept优化

声明函数不会抛出异常
忽视异常机制，编译器会优化部分操作

效率：移动 > 拷贝

但是移动时如果抛出异常，可能会导致源对象的被破坏

所以实际上编译器在未标注 `noexcept` 时会拷贝

### 参数列表

先输入后输出，先数据后方法

传参符合直觉

```cpp
CV_EXPORTS_W bool solvePnP( InputArray objectPoints, InputArray imagePoints,
                            InputArray cameraMatrix, InputArray distCoeffs,
                            OutputArray rvec, OutputArray tvec,
                            bool useExtrinsicGuess = false, int flags = SOLVEPNP_ITERATIVE );
```

### 枚举类和 bool

对于明显不属于 *是否* / *有/无* 的状态或是多状态，建议使用枚举类型 `enum` 代替 `int` 或 `bool`
`
enum ColorMode { RED = 0, BLUE = 1 };
`

代替

`isBlue`

建议在枚举类前加上命名空间并避免简单命名导致的重名

### Doxygen

装 Doxygen 插件，写这样的注释
```cpp
    /**
     * @brief 预处理
     * 
     * @param image 原始图片
     * @return cv::Mat 预处理后的图片，黑白
     */
    cv::Mat preprocess(cv::Mat& image);
```

重要的是 `brief` `param` `return` 
其次是 `details` `throw`

### CI

自己配一个试试，最少要可以检查代码是否能编译

### 包含优于继承?

#### 什么时候包含

不是明显的逻辑上的属于关系

#### 什么时候继承

明显的逻辑上的属于关系
需要实现多态

### 类的设计

使用智能指针管理资源

对成员管理的资源操作时使用 set() get()等方法
避免直接对类的成员进行操作，使用封装好的方法

### 纯函数

不修改外部变量，相同输入相同输出，不依赖外部状态

C++ 没有纯函数定义，但是纯函数是个好东西

简单来说，多用 const