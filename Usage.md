# 基本架构
Client-Server Architecture: Server建立仿真世界，Client由用户控制，来操控仿真世界。
- Server处理仿真相关的所有事务：渲染、模型构建、物理计算。
- Client控制仿真世界如何运转，通过Python/C++ API，可以修改世界。也接受server回传的信息。
![[architecture.png]]
-   Command: 
	-   steering $\in \mathbf{R}，[-1,1]$ 
	-   throttle, brake $\in \mathbf{R}，[0,1]$
	-   hand brake, reverse gear $boolean$
-   meta-commands: (1) number of vehicles, (2) number of pedestrians, (3) weather ID, (4) seed vehicles / pedestrians, (5) set of cameras (RGB, depth, semantic segmentation)
- measurements and sensor readings
	- player: position, orientation, speed, acceleration
	- infraction: collision with static/car/pedestrian, opposite lane intersection, sidewalk intersection 
	- info: time, non-client-controlled agents info, traffic lights info, speed limit signs info
	- sensor readings

### 核心模块

1.  **Traffic Manager**: 自动驾驶之所以难搞，很核心的一个原因就是现实世界车太多了！试想如果整个世界就你一辆车在大马路上跑，自动驾驶恐怕早实现了。因此，Carla专门构造了Traffic Manager这个模块来模拟类似现实世界负责的交通环境。通过这个模块，用户可以定义N多不同车型、不同行为模式、不同速度的车辆在路上愉快地与你的自动驾驶汽车（Ego-Vehicle）一起玩耍。这个模块后面会详细讲解。
2.  **Sensors:** Carla里面有各种各样模拟真实世界的传感器模型，包括相机、激光雷达、声波雷达、IMU、GNSS等等。为了让仿真更接近真实世界，它里面的相机拍出的照片甚至还有畸变和动态模糊效果。用户一般将这些Sensor attach到不同的车辆上来收集各种数据。
3.  **Recorder：** 俗话说的好，不能复现的仿真不是好仿真。这个模块就是用来记录仿真每一个时刻（Step)的状态，可以用来回顾、复现等等。
4.  **ROS bridge：** 这个模块可以让Carla与ROS还有Autoware交互，正是这个模块的存在使得在仿真里测试你的自动驾驶系统变得可能，十分重要，后面也会详细讲解。
5.  **Open Assest**：这个模块可以允许你为仿真世界添加customized的物体库，比如你可以在默认的汽车蓝图里再加一个真实世界不存在、外形酷炫的小飞汽车，用来给Client端调用。