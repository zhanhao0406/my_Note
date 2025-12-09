# 话题：节点间传递数据的桥梁

## Publisher
源码及注释  
```c++
#include <chrono>              //时间处理模块  
#include <functional>          //函数对象和绑定  
#include <memory>              //智能指针  
#include <string>

#include "rclcpp/rclcpp.hpp"          // ROS2 C++接口库
#include "std_msgs/msg/string.hpp"    // 字符串消息类型

using namespace std::chrono_literals;   //允许使用时间字面值，如500ms

class PublisherNode : public rclcpp::Node  // 定义“类”名称叫PublisherNode，“类”的特性继承自 Node。
{
    public:
        PublisherNode()
        : Node("topic_helloworld_pub") // ROS2节点父类初始化, 调用父类构造函数
        {
            // 创建发布者对象（消息类型、话题名、队列长度）, <std_msgs::msg::String>是模版参数，("chatter", 10)是函数参数
            publisher_ = this->create_publisher<std_msgs::msg::String>("chatter", 10); 

             // 创建一个定时器,定时执行回调函数。std::bind用于将函数和它的参数绑定，创建新的可调用对象。
            // &PublisherNode::timer_callback是获取成员函数地址。this是绑定当前对象
            // C++的成员函数不能直接作为回调函数，需要绑定对象实例(this)，因为成员函数需要知道操作哪个对象
            // 下面的表达可等价替换用lambda表达式: timer_ = this->create_wall_timer(500ms, [this]() {
                                                    // this->timer_callback();
                                                    // });
            timer_ = this->create_wall_timer(
                500ms, std::bind(&PublisherNode::timer_callback, this));            
        }

    private:
        // 创建定时器周期执行的回调函数
        void timer_callback()                                                       
        {
          // 创建一个String类型的消息对象。auto就是自动推断，这里是std_msgs::msg::String
          auto msg = std_msgs::msg::String();   
          // 填充消息对象中的消息数据                                    
          msg.data = "Hello World";
          // 发布话题消息
          // 是一个宏，宏是预处理器在编译前进行文本替换的。'%s'是字符串占位符。                                              
          RCLCPP_INFO(this->get_logger(), "Publishing: '%s'", msg.data.c_str()); 
          // 输出日志信息，提示已经完成话题发布。智能指针指向访问成员，发布核心函数msg   
          publisher_->publish(msg);                                                
        }

        // 声明成员变量
        rclcpp::TimerBase::SharedPtr timer_;                             // 定时器指针
        rclcpp::Publisher<std_msgs::msg::String>::SharedPtr publisher_;  // 发布者指针。智能指针，指向发布者对象。
};

// ROS2节点主入口main函数
int main(int argc, char * argv[])                      
{
    // ROS2 C++接口初始化
    rclcpp::init(argc, argv);                
    
    // 创建ROS2节点对象并进行初始化          
    rclcpp::spin(std::make_shared<PublisherNode>());   
    
    // 关闭ROS2 C++接口
    rclcpp::shutdown();                                

    return 0;
}
```
---
## Subscriber
```c++
#include <memory>                              // 智能指针库
#include "rclcpp/rclcpp.hpp"                  // ROS2 C++接口库
#include "std_msgs/msg/string.hpp"            // 字符串消息类型

// 占位符，用于std::bind，表示"回调函数的第一个参数"。
//_1 就是告诉ROS2："回调函数的第一个参数，用你收到的消息来填充"。
//想象点外卖：你（ROS2系统）告诉外卖小哥（回调函数）："把外卖（消息）送到我家"。 _1 就像是说："把外卖（消息）放在第一个参数的位置"
using std::placeholders::_1;         

class SubscriberNode : public rclcpp::Node
{
    public:
        SubscriberNode()
        : Node("topic_helloworld_sub")        // ROS2节点父类初始化
        {
            // 模板参数<std_msgs::msg::String>指定要接收的消息类型。No.1函数参数是名称，No.2函数参数是队列长度，No.3函数参数是回调函数绑定。有参数时先占位，然后消息到达时，用msg代替_1的位置。
            subscription_ = this->create_subscription<std_msgs::msg::String>(       
                "chatter", 10, std::bind(&SubscriberNode::topic_callback, this, _1));   // 创建订阅者对象（消息类型、话题名、订阅者回调函数、队列长度）
        }

    private:
        // 创建回调函数。topic_callback 是自定义的回调函数名称。std_msgs::msg::String::SharedPtr是智能指针，指向接收到的消息。 msg则是参数名。
        void topic_callback(const std_msgs::msg::String::SharedPtr msg) const                  // 创建回调函数，执行收到话题消息后对数据的处理
        {
            RCLCPP_INFO(this->get_logger(), "I heard: '%s'", msg->data.c_str());       // 输出日志信息，提示订阅收到的话题消息
        }
        //成员变量声明。为什么需要成员变量，是为了有1.生命周期管理，2.当节点销毁时，智能指针会自动调用订阅者的析构函数，向ROS2注销订阅
        rclcpp::Subscription<std_msgs::msg::String>::SharedPtr subscription_;         // 订阅者指针
};

// ROS2节点主入口main函数
int main(int argc, char * argv[])                         
{
    // ROS2 C++接口初始化
    rclcpp::init(argc, argv);                 
    
    // 创建ROS2节点对象并进行初始化            
    rclcpp::spin(std::make_shared<SubscriberNode>());     
    
    // 关闭ROS2 C++接口
    rclcpp::shutdown();                                  
    
    return 0;
}
```

