#ros2 核心概念
节点 Node：ROS2 最小执行单元，一个功能对应一个节点，每个节点独立进程，通过 DDS 通信，解耦。
话题 Topic（异步单向通信）：发布者 (Publisher) 发消息，订阅者 (Subscriber) 接收，一对多、多对多，适合持续数据流（激光雷达、相机图像）。
服务 Service（同步双向通信）：客户端发请求、服务端返回应答，一问一答阻塞，适合单次指令（开关设备、获取机器人坐标）。
动作 Action（异步双向通信）：目标 + 反馈 + 结果，长耗时任务（导航、机械臂运动、路径跟踪），可中途取消。
rclcpp：ROS2 C++ 标准客户端库，封装 DDS 底层通信，提供节点、话题、服务、动作 API。
工作空间 Workspace：存放源码 src、编译目录 build/install/log，标准目录结构。
colcon：ROS2 统一编译工具，替代 catkin，支持多语言包并行编译。
DDS = Data Distribution Service，数据分发服务，是一套工业级实时分布式通信标准（OMG 组织制定），专门用来解决多进程 / 多设备之间实时收发数据。
ROS 2 完全基于 DDS 做底层通信；ROS 1 没有 DDS，用的是自定义 TCP/UDP 通信（roscore），这是 ROS1 和 ROS2 最核心区别。
核心定位：机器人多节点、多机、多传感器实时数据传输中间件，负责：话题发布订阅、服务、动作、参数、节点发现。
DDS 核心设计思想：发布 - 订阅（Publish-Subscribe）
发布者 Publisher：发送数据（相机图像、激光雷达、速度指令）
订阅者 Subscriber：接收对应话题数据
数据域 Domain：同一数字域内的节点才能互相发现、通信（默认 domain_id=0）

#话题通信   发布者  +   订阅者
  发布者：
  ```
      #include "rclcpp/rclcpp.hpp"
      #include "std_mags/msg/string.hpp"
      #include <chrono>

      class Talker: public rclcpp:: Node{
        public :
            Talker(): Node("talker_node"){
               pub_=this->create_publisher<std_msgs::msg::String>("chatter",10);
               timer_=this->create_wall_timer(chrono_literals::500ms,std::bind(&Talker::timer_cb,this));
              cnt_=0;
}
       private:
              void timer_cb(){
                  auto msg = std_msgs::msg::String();
                  msg.data = "hello ros2 cnt:" + std::to_string(ccnt++);
                  RCLCPP_INFO(this->get_logger(),"fabu :%s",msg.data.c_str());
                  pub_->publish(msg);
}
            rclcpp::Publisher<std_msgs::msg::String>::SharedPtr pub_;
            rclcpp::TimeBase::SharedPtr timer_;
            int cnt_;
 };
          int main(int argc,int** argv){
              rclcpp::init(argc,argv);
              rclcpp::spin(std::make_shared<Talker>());
              rclcpp::shutdown();
              return 0;
}              
```
   订阅者：
   ```
          #include "rclcpp/rclcpp.hpp"
          #include "std_msgs/msg/string.hpp"

class Listener : public rclcpp::Node {
         public:
           Listener(): Node("listener_node"){
                sub_=this->create_subscription<std_msgs::msg::String>("chatter",10,std::bind
(&Listener::sub_cb,this,std::placeholders::_1));
}
private:
       void sub_cb(const std_msgs::msg::String::SharedPtr msg)
{
          RCLCPP_INFO(this->get_logger(),"收到消息: %s",msg->data.c_str());
}
            rclcpp::Subscription<std_msgs::msg::String>::SharedPtr sub_;
};
int main(int argc, char** argv)
{
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<Listener>());
    rclcpp::shutdown();
    return 0;
}
```
#通信服务：
```
自定义服务类型
       srv/AddTwoInts.srv
            int64 a
            int64 b
           ---------
            int64 sum
         服务端：
             #include "rclcpp/rclcpp.hpp"
             #include "demo_srv/srv/add_two_ints.hpp"

class AddService : public rclcpp::Node{
         public:
          AddService(): Node("add_sevice_node") {
              ser_ = this->create_service("add_two_ints",
                   std::bind(&AddService::ser_cb),this,std::placeholder::_1,std::placeholder::_2);
                RCLCPP_INFO(this->get_logger(),"服务端已经启动等待请求");
}
   private:
              void ser_vb(const std::shared_ptr<AddTwoInts::Requst> req,const std::shared_ptr<AddTwoInts::Response res){
res->sum = req->a + req->b;
RCLCPP_INFO(this->get_logger(), "收到 %ld + %ld = %ld", req->a, req->b, res->sum);
}
           rclcpp::Service<AddTwoInts>SharedPtr ser_;
};
int main(int argc,char** argv){
         rclcpp::init();
         rclcoo::spin(std::make_shared<AddServer>());
rclcpp::shutdown();
return 0;
}
```
  客户端：
   ```
      #incldue "rclcpp/rclcpp.hpp"
      #include "demo_srv/srv/add_two_ints.hpp"
      #inlcude <chrono>

 class AddClient : public rclcpp:: Node{
              public:
            AddClient():Node(add_client_node){
                   cli_ = this->create_client<AddTwoInts>("add_two_ints");
}
bool send_req(int64_t a,int64_t b){
       while(!cli_->wait_for_service(1s)){
              if(!rclcpp::ok()){
            RCLCPP_ERROR(this->get_logger(), "客户端中断");
                return false;
                    }
RCLCPP_WARN(this->get_logger(), "等待服务启动...");
}
auto req = std::make_shared<AddTwoInts::Request>();
req->a = a;
req->b = b;
auto result_future = cli_->async_send_request(req);
if(rclcpp::spin_until_feature_complete(this->get_node_base_interface(),result_feature)==rclcpp::FutureReturnCode::SUCCESS)
{
    RCLCPP_INFO(this->get_logger(), "计算结果: %ld", result_future.get()->sum);
            return true;
}
RCLCPP_ERROR(this->get_logger(), "请求失败");
        return false;
}
private:
    rclcpp::Client<AddTwoInts>::SharedPtr cli_;
};

int main(int argc, char** argv)
{
    rclcpp::init(argc, argv);
    auto client = std::make_shared<AddClient>();
    client->send_req(10, 20);
    rclcpp::shutdown();
    return 0;
}
  ```

#编译配置 package.xml + CMakeLists.txt 
  CMakeLists.txt 核心片段
  ```
    cmake_minimum_required(VERSION 3.8)
project(demo_ros2)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# 依赖
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)
find_package(geometry_msgs REQUIRED)

# 可执行文件
add_executable(talker src/talker.cpp)
ament_target_dependencies(talker rclcpp std_msgs)

add_executable(listener src/listener.cpp)
ament_target_dependencies(listener rclcpp std_msgs)

# 安装
install(TARGETS
  talker
  listener
  DESTINATION lib/${PROJECT_NAME}
)

ament_package()
```
package.xml 核心依赖
```
<buildtool_depend>ament_cmake</buildtool_depend>
<depend>rclcpp</depend>
<depend>std_msgs</depend>
```
Colcon 编译命令
```
# 进入工作空间
cd ~/ros2_ws
# 全量编译
colcon build 
# 只编译单个包
colcon build --packages-select demo_ros2
# 编译后刷新环境变量
source install/setup.bash
# 运行节点
ros2 run demo_ros2 talker
ros2 run demo_ros2 listener
```
#ros工具与调试
Launch 文件
```
 from launch import LaunchDescription
 from launch_ros.actions import Node

def generate_launch_desscription(){
    talker_node = Node(
        package="demo_ros2",
        executable="talker",
        name="talker",
        output="screen"
    )
    listener_node = Node(
        package="demo_ros2",
        executable="listener",
        name="listener",
        output="screen"
    )
return LaunchDescription([talker_node, listener_node])
```
#TF坐标变换
    模拟小车odom里程计坐标变换(动态坐标广播)
 ```
  #include "rclcpp/rclcpp.hpp"
  #include "tf2_ros/transform_broadcaster.h"
  #include "geometry_msgs/msg/transform_stamped.hpp"
  #include "tf2/LinearMath/Quaternion.h"

  using namespace std::chrono_literals;

  class OdomTfPublisher :  public rclcpp::Node 
  {
     public:
         tf_broadcaster_ = std::make_shared<tf2_ros::TransformBroadcaster>(this);
         timer_ = this->create_wall_timer(20ms,std::bind(&OdomTfPublisher::publish_odom_tf,this));
         x_pos = 0.0
         y_pos_=0.0
         yaw_ = 0.0;
         linear_speed_ = 0.2;
         }
     private:
            void publishing_odom_tf(){
               geometry_msgs::msg::TransformStamed transform;
               transform.header.stamp = this-> get_clock()->now();
               transform.header.frame_id = "map";
               transform.child_frame_id = "base_link";
               double dt = 0.02 ;
               x_pos_ += linear_speed_*dt;
               transform.transform.translation.x = x_pos_;
               transform.transform.translation.y = y_pos_;
               transform.transform.translation.z = 0.0;
               tf2::Quaternion quat;
               quat.setRPY(0,0,yaw_);
               transform.transform.rotation.x = quat.x();
               transform.transform.rotation.y = quat.y();
               transform.transform.rotation.z = quat.z();
               transform.transform.rotation.w = quat.w();
               tf_broadcaster_->sendTransform(transform);
               RCLCPP_INFO(this->get_logger(),"小车map坐标 x: %.2f m",x_pos_);
               }
               std::shared_ptr<tf2_ros::TransformBroadcaster> tf_broadcaster_;
               rclcpp::TimerBase::SharedPtr timer;
               double x_pos_,y_pos_,yaw_,linear_speed_;
               };
        int main(int argc,char** argv){
                   rclcpp::init(argc,argv);
                   auto node = std::make_shared<OdomTfPublisher>();
                   rclcpp::spin(node);
                   rclcpp::shutdown();
                   return 0;
                   }
                   ```
#静态tf广播
    ```
        #include "rclcpp/rclcpp.hpp"
        #include "tf2_ros/static_transform_broadcaster.h"
        #include "geometry_msgs/msg/transform_stamped.hpp"
        #include "tf2/LinearMath/Quaternion.h"

        class StaticTfBroadcaster : public rclcpp :: Node
        {
           public:
              StaticTfBroadcaster() :  Node("static_tf_broadcaster")
              {
                broadercaste_ = std:: make_shared<tf2_ros""StaticTransformBroadcaster>(this);
                publish_tf();
                  }
            private:
                 void publish_tf()
                 {
                   geometry_msgs::msg::TransformStamped t;
                   t.header.stamp = this-> get_clock()->now();
                   t.header.frame_id = "base_link";
                   t.child_frame_id = "laster_link";

                   t.transform.translation.x = 0.2;
                   t.transform.translation.y = 0.0;
                   t.transform.translation.z = 0.1;

                   tf2::Quaternion q;
                   q.setRPY(0,0,0);
                   t.transform.rotation.x = q.x();
                   t.transform.rotation.y = q.y();
                   t.transform.rotation.z = q.z();
                   t.transform.rotation.w = q.w();

                   broadcaster_->sendTransform(t);
                   }
                   std::shared_ptr<tf2_ros::StaticTransformBroadcaster> broadcaster_;
                   };

                   int main (int argc,char** argv){
                     rclcpp::init(argc,argv);
                     rclcpp::spin(std::make_shared<StaticTFBroadcaster>());
                     rclcpp::shutdown();
                     return 0;
                     }
                 ```    
#URDF```
//XML 格式，描述机器人连杆 link、关节 joint、惯性、碰撞、可视化外形。
    <？xml version = "1.0"?>
    <robot name="diff_car">
       <link name = "base_link">
          <visual>
               <geometry><box size = "0.4 0.2 0.1"/></geometry>
               <material name = "blue"><color rgba="0 0 1 1 "/></material>
          </visual>
          <collision>
            <geometry>
               <box size = "0.4 0.2 0.1"/>
            </geometry>
          </collision>
          <inertial>
              <mass Value = "2.0"/>
              <inertia ixxx="0.1" iyy="0.1" izz="0.1" />
          </inertial>
      </link>
      <joint name = "left_wheel_joint" type = "continuous">
          <parent link="base_link"/>
          <child link = "left_wheel_link"/>
          <origin xyz="-0.15 -0.15 0"/>
          <axis xyz = "0 1 0"/>
      </joint>
      <link name = "left_wheel_link">
          <visual>
             <geometry>
                 <cylinder radius="0.08" length="0.04"/>
              </geometry>
          </visual>
       </link>
</robot>
```
#Xacro
URDF 原生不支持变量、循环、宏，xacro 做预处理，编译成标准 URDF。
```
<?xml version = "1.0"?>
<robot xmlns:xarco="http://www.ros.org/wiki/xacro" name="diff_car">
   <xacro:property name="wheel_r" value="0.08"/>
   <xacro:property name="wheel_r" value="0.04"/>
   <xacro:macro name = "wheel_macro" params= "joint_name link_name x y">
        <joint name="${joint_name}" type="continuous">
           <parent link = "base_link"/>
           <child link = "${link_name}"/>
           <orgin xyz=${x} ${y} 0/>
           <axis xyz="0 1 0"/>
        </joint>
        <link name  = "${link_name}">
             <visual>
                   <geometry><cylinder radius= "${wheel_r}" length="${wheel_l}"/ ></geometry>
             </visual>
        </link>
</xacro:macro>

<xacro:wheel_macro joint_name = "left_wheel_joint" link_name = "left_wheel" x="-0.15" y="-0.15"/>
<xacro:wheel_macro joint_name="right_wheel_joint" link_name="right_wheel" x="-0.15" y="0.15"/>
//robot
```
xacro 转 urdf 命令       xacro car.xacro > car.urdf

#Gazebo
Gazebo 是 ROS2 官方物理仿真器，插件实现驱动、传感器、电机控制，核心插件：
```
<gazebo>
    <plugin name="diff_drive_plugin" filename="libgazebo_ros_diff_drive.so">
        <left_joint>left_wheel_joint</left_joint>
        <right_joint>right_wheel_joint</right_joint>
        <wheel_separation>0.3</wheel_separation>
        <wheel_radius>0.08</wheel_radius>
        <publish_rate>30</publishing_rate>
        <odom_topic>odom</odom_topic>
        <cmd_vel_topic>cmd_vel</cmd_vel_topic>
    </plugin>
</gazebo>
```
ros2_control（ROS2 标准硬件控制框架）    替代旧 Gazebo 插件驱动，统一硬件 / 仿真控制，三层架构：
硬件接口 Hardware Interface：读取位置 / 速度 / 力矩
控制器 Controller：差速控制器、PID 关节控制器、轨迹控制器
控制器管理器 Controller Manager：加载、切换控制器

#启动仿真实例（整合 URDF+Gazebo+Rviz2）
```
from launch import launchDescription
from  launch_ros.actions import Node
from  launch.actions import IncludeLaunchDescription
from  launch.launch_description_sources import PythonLaunchDescriptionSource
import os
from ament_index_python.packages import get_package_share_directory

def generate_launch_description();
     pkg_path = get_package_share_directory("robot_sim")
     urdf_file = os.path.join(pkg_path,"urdf/car.xacro")
     robot_state_pub = Node(package = "robot_state_publisher",executable="robot_state_publisher",parameters=[{"robot_description":open(urdf_file).read()}])

gazebo  = IncludeLaunchDescription(PythonLaunchDescriptionSource([os.path.join(get_package_share_directory("gazebo_ros"),"launch/gazebo.launch.py")]))

spawn_entity = Node(package="gazebo_ros",executable = "spawn_entity.py",argument=["-topic","robot_description","-entity","diff_car"])

rviz2 = Node(package="rviz2",executable="rviz2")
return LaunchDescription([robot_state_pub,gazebo,spawn_entity,rviz2])
```

