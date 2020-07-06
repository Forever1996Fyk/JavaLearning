## <center>Spring IOC容器</center>

Spring容器是Spring Framework的核心。容器将创建对象, 把它们关联一起, 配置对象, 并管理他们的整个生命周期从创建到销毁。Spring容器视同依赖注入(DI)来管理组成一个应用程序的组件。这些对象称为Spring Beans。

IOC容器是具有依赖注入功能的容器, 可以创建对象, IOC容器负责实例化, 定位, 配置应用程序中的对象以及建立对象之间的依赖。通常我们new一个实例, 控制权有程序员控制, 而"控制反转"是指new实例的工作不由程序员实现,而是交给Spring容器。其中IOC容器代表为**BeanFactory,  ApplicationContent**。

### 1. Spring BeanFactory容器

这个Spring中最简单, 最基础的IOC容器, 主要的功能是为依赖注入(DI)提供支持。

注意: <font color="red">**BeanFactory是存在Spring中的, 但是现在在项目中一般我们是不用BeanFactory的, 而是用它的实现类。 它只是为IOC容器提供一个入口, 目的是向后兼容已经存在的和Spring整合在一起的第三方框架**</font>。

> BeanFactory的每个子接口, 以及实现类都有使用的场合, 他主要是为了区分在 Spring 内部对象的传递和转化过程中, 对对象数据访问所做的限制!!!
