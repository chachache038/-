# <center>DAY 7
**记录者：江栩**
**记录时间：2026.7.19** 
- [x] 参数（全局字典）

## 一.学到的东西
### 1.1 参数的本质：分布式配置中心
我明确了，ROS2 的参数不是普通变量，而是一个去中心化的配置管理系统。

1.  存储位置：参数并不存在一个单独的“服务器”进程里，而是存储在节点（Node）内部。

> 管理方式：节点通过 rclpy.node.Node 提供的 API 管理自己的参数。这种设计避免了单点故障，符合 ROS2 的分布式理念。

2. 参数的完整生命周期（核心流程）

|阶段|核心动作|我的理解|
|:---|:-------:|------:|
|声明 (Declare)​|self.declare_parameter('name', default)|“上户口”。告诉系统：“我这个节点有一个叫 name 的参数，默认值是 default”。如果不声明就读取，系统会报错。这是为了安全，防止拼写错误。|
|读取 (Get)​|self.get_parameter('name').value|“查户口”。从节点内部获取当前生效的值。我注意到通常直接用 .value，这是 ROS2 提供的快捷方式，底层其实还是我之前觉得复杂的那个三层结构（为了类型安全）。|
|修改 (Set)​|双入口设计​|“改户口”|
	

>入口 A：终端命令
>>ros2 param set /node_name name value

人工干预。用于调试。比如我怀疑速度太快，直接在终端把 max_speed 改成 0.5 试试，不用改代码、不用重新编译。


>入口 B：代码 API
>>self.set_parameters([Parameter(...])

自动调整。用于逻辑。比如节点检测到硬件故障，自动把 safety_mode 设为 True。


关键结论​	

无论哪种入口，改的都是节点内存中的同一个参数值。​ 终端命令本质是通过 DDS 服务调用，触发了节点内部的 set_parameters 逻辑。

 + **为什么设计得这么“重”？**
之前觉得 ROS2 参数操作很啰嗦（建对象、包列表）。今天理解了背后的工程考量：

   - 类型安全：强制指定类型（Parameter.Type.STRING），防止把字符串当数字用，这在大型系统中能避免灾难性错误。

    - 批量操作：set_parameters 接受列表，允许原子性地更新多个参数。比如同时更新 PID 的 P、I、D 三个值，避免出现中间态。

    - 事件驱动：参数被修改后，可以触发回调函数。这意味着节点可以实时响应配置变化，而不需要重启。这是 ROS2 相比 ROS1 的巨大进步。

    - 自描述性：参数可以附带描述信息（description），方便用 ros2 param describe 查看，这对团队协作和系统集成至关重要。

今日实操验证

为了巩固理解，我在终端和代码中做了对照实验：

终端侧：用 ros2 param list 看到了我声明的参数；用 ros2 param get 读取值；用 ros2 param set 成功修改了值。

代码侧：在节点内用 get_parameter 读到了终端修改后的值，验证了“双入口改同一值”。

回调验证：注册了参数回调，每当参数变化，终端自动打印出新值。这证明了参数系统的事件驱动特性。
```py
import rclpy                                     # ROS2 Python接口库
from rclpy.node   import Node                    # ROS2 节点类
 #所有"封装好给别人调用的功能包"都叫库/接口
#noderclpy：ROS 2 的 Python 总库，提供初始化、自旋（spin）、关闭等全局能力
#rclpy.node：rclpy 这个包下面的一个子模块（node.py 文件）
#这个就是引用node来生成子类
class ParameterNode(Node):
    def __init__(self, name):
        super().__init__(name)                                  # ROS2节点父类初始化
        #父类继承
        self.timer = self.create_timer(2, self.timer_callback)    # 创建一个定时器（单位为秒的周期，定时执行的回调函数）
        #"调用父类提供的 create_timer 方法，
        # 让 ROS 系统每 2 秒自动执行一次 timer_callback 函数，
        # 同时把返回的定时器对象存到 self.timer 这个属性里。"
        self.declare_parameter('robot_name', 'mobot') # 创建一个参数，并设置参数的默认值
        #只要看到 () 括号 + 参数，就是"调用一个函数/方法"，
        # 不管前面有没有 self.。
    def timer_callback(self):                                      # 创建定时器周期执行的回调函数
        #Python 里"一切"皆对象，所以 . 后面不仅能拿数据，还能拿函数、拿模块、拿类
        robot_name_param = self.get_parameter('robot_name').get_parameter_value().string_value   # 从ROS2系统中读取参数的值
        #第①步：self.get_parameter('robot_name')
        #这是父类 Node 的方法
        # 返回一个 Parameter 对象（不是字符串！是一个包装好的参数对象）
        # 这个对象里装着：参数名、参数类型、参数值等信息
        # 第②步：.get_parameter_value()
        # 在 Parameter 对象上调用的方法
        # 返回一个 ParameterValue 对象
        # 为什么要多这一层？因为 ROS 2 的参数支持多种类型（字符串、整数、布尔、数组…），ParameterValue 就是用来统一描述"值+类型"的容器
        # 第③步：.string_value 
        # 这是 ParameterValue 对象上的一个属性（不是方法，所以没括号）
        #直接从里面取出 Python 字符串
        self.get_logger().info('Hello %s!' % robot_name_param)     # 输出日志信息，打印读取到的参数值
        #在这里是输出日志，但是实际上就是打印在终端上,but日志系统做的事，比 print 多得多

        new_name_param = rclpy.parameter.Parameter('robot_name',   # 重新将参数值设置为指定值
                            rclpy.Parameter.Type.STRING, 'mobot')
        all_new_parameters = [new_name_param]
        self.set_parameters(all_new_parameters)                    # 将重新创建的参数列表发送给ROS2系统

def main(args=None):                                 # ROS2节点主入口main函数
    rclpy.init(args=args)                            # ROS2  
    #这个为正常操作，就是将节点初始化
    node = ParameterNode("param_declare")            # 创建ROS2节点对象并进行初始化
    rclpy.spin(node)                                 # 循环等待ROS2退出
    #bu zhi dao spin dao di zai gen shen mo
    node.destroy_node()                              # 销毁节点对象
    rclpy.shutdown()                                 # 关闭ROS2 Python接口
 ```
![alt text](570f64a9-4954-4531-802f-5615a9bd3328.png)

### 二。总结与反思

今天的学习非常聚焦。我不再纠结 Python 的 . 是如何实现的，而是关注它在 ROS2 语境下的语义：
self.declare_parameter()：调用节点提供的“声明”服务。
self.get_parameter()：调用节点提供的“查询”服务。
self.set_parameters()：调用节点提供的“更新”服务。
核心思维转变：把节点看作一个黑盒服务，参数系统是这个服务对外暴露的配置接口。无论是人（终端）还是其他程序（代码），都是通过这个标准接口来配置节点。