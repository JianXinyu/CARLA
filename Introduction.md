# Ideas
1. CARLA：使用方法+画一下代码框架（越详细越好）；
2. 尝试将百度Apollo在CARLA中使用起来
首先训练真实的行人, 汽车AI模型，尽可能复现真实环境中行人和汽车的行为。模型有一系列的情绪参数，比如遵守规则、暴躁、易变、行动目标等，随机生成参数放入simulator里。

[Brief history](https://www.unrealengine.com/zh-CN/spotlights/carla-democratizes-autonomous-vehicle-r-d-with-free-open-source-simulator)

# Paper Review
[CARLA: An Open Urban Driving Simulator](http://proceedings.mlr.press/v78/dosovitskiy17a/dosovitskiy17a.pdf)

## Introduction
自动驾驶属于sensorimotor control，难点在于人口稠密城市中的导航：
- 路口复杂的multi-agent dynamics
- 任何时候都可能要对数百个actors跟踪和响应
- 遵守交通规则，识别street signs, street lights, road markings, 区分不同车辆类型
- 应对 long tail of rare events， 如道路施工、熊孩子、事故、流氓司机
- 应对冲突目标，如为了刹车躲避失神的行人就得被追尾

现实中的自动驾驶研究受制于基础设施成本和人力。
- 要覆盖所有的corner cases需要大量的车和数据。
- 一些危险的情景无法实现，如熊孩子

仿真可替代部分真实世界中的训练和验证，目前应用(2017)有racing simulators, custom simulation setups, commerical games. 目前的仿真器有:
- TORCS 场景不够复杂
- Grand Theft Auto V 不支持详细的驾驶策略基准测试

CARLA(CAR Learning to Act)是什么? an open-source simulator to support *development, training and validation* of autonomous **urban** driving systems, including . *perception and control*. CARLA provides:
	- code and protocals
	- digital assets
	- 灵活的传感器组合，各种关键信息(GPS坐标、速度、加速度、碰撞违规信息)
	- 天气、光照配置
	
- Study cases:
	- [[#Modular pipeline]]
	- [[#Imitation Learning]]
	- [[#Reinforcement Learing]]


## Simulation Engine
底层使用虚幻4进行渲染和物理仿真。
采用Server-Client架构。
![[architecture.png]]
-   Command: 
	-   steering $\in \mathbf{R}，[-1,1]$ 
	-   throttle, brake $\in \mathbf{R}，[0,1]$
	-   hand brake, reverse gear $boolean$
-   meta-commands: (1) number of vehicles, (2) number of pedestrians, (3) weather ID, (4) seed vehicles / pedestrians, (5) set of cameras (RGB, depth, semantic segmentation)
- measurements and sensor readings
	- player: position, orientation, speed, acceleration
	- infraction: collision with static/car/pedestrian, opposite lane intersection, sidewalk intersection (30% footprint，2s计时)
	- info: time, non-client-controlled agents info, traffic lights info, speed limit signs info
	- sensor readings
### Environment
由静态对象(楼房， 植被，交通标志，基建)和动态对象(车辆，行人)的3D模型组成，其设计平衡了视觉效果和渲染速度。所有的3D模型缩放一致，反映了真实大小。

如何构建城市环境？
1. 铺设道路和人行道
2. 手动放置房屋、植被、地形和交通基础设施
3. 指定动态对象可以出现（生成）的位置

开发CARLA的一大难点是如何处理non-player characters.
- vehicles: based on UE4 vehicle model. kinematic parameters, basic controller(保持车道，遵守交规，路口决策)。喷漆随机。
- pedestrians: location-based cost 指导了行人如何游荡。如果与汽车相撞，将被删除，随后在不同的位置生成新的行人。穿的衣服和手持的物品随机。

支持两种照明条件（正午和日落）以及9种天气条件（不同的云量、降水量和街道上是否有水坑）。总共18种illumination-weather combinations.

### Sensors
可灵活配置传感器套件。传感器模型目前(2017)仅有
- RGB Camera: number, type, position. Parameters: 三维位置、相对于汽车坐标系的三维方向、视野和景深
- Pseudo-sensors that provide ground-truth depth and semantic segmentation (12个语义类：道路、车道标线、交通标志、人行道、围栏、标杆、墙、建筑、植被、车辆、行人和其他)

还提供一些测量值:
- measurement of the agent's state:  车辆相对于世界坐标系（类似于GPS和罗盘）的位置和方向、速度、加速度矢量和碰撞累积的影响。
- measurement concerning the traffic rules: 进入到错误车道或人行道的车辆足迹百分比，以及交通灯状态和车辆当前位置的速度限制。
- 环境中所有动态对象的精确位置和边界框(bounding boxes)

## Autonomous Driving
- Obeservation $o_t$: a tuple of sensory inputs
	- high-dimensional data: color images, depth maps
	- lower-dimensional data: speed, GPS
- Action $a_t$: steering, throttle, brake
- A* algorithm 给agent规划path，指导agent左拐、右拐、直行。这个是Global Planner
- no metic map==没有地图怎么使用A*?==

### Modular pipeline
感知、规划、控制架构。多数自动驾驶系统使用这个。参考Paper: A Survey of Motion Planning and Control Techniques for Self-driving Urban Vehicles.

- Perception
	- Semantic segmentation network based on RefineNet: 估计车道、道路限制、动态对象和其他危险。网络按照semantic catergories， 将图像中的每个像素分类。根据道路面积和车道标线，利用网络提供的概率分布来估算自我车道。The network output is also used to compute an obstacle mask that aims to encompass pedestrians, vehicles, and other hazards???
	- binary scene classifier(intersection/no intersection)  based on AlexNet: 估计到达交叉路口的似然。

- Local Planner(LP) 生成路径点(位置、方向)，使汽车保持在道路上并防止碰撞。基于state machine（道路跟随，路口左转，路口右转，路口直行，急停）。 路径点、车当前位姿和速度发给controller 
	- road following: semantic segmentation给出ego-lane mask, LP用其选择与右边线保持一定距离的路径点
	- left-turn: 会暂时看不到 lane marking，且离目标车道越远，前视摄像头的视野越受限。Solution: 先计算能斜着到路口中心的路径点，来提升视野；加一个摄像头
	- right-ture: 目标车道更近，容易点(左侧行驶的国家相反)。
	- intersection-forward: 与road following像
	- hazard-stop: 动态障碍map给出的碰撞概率高于阈值时，生成一个特殊waypoint

- Continuous Controller - PID. 接收当前位姿、速度和路径点list，驱动转向、油门和制动。目标速度是20 km/h，PID参数在training town中调节。

### Imitation Learning
Conditional imiation learning, 一种除了感知输入外还使用高级命令的模拟学习。训练时用到了一个人类车手的driving traces数据集:
- observations: images from a forward-facing camera
- commands: 保持车道、下个路口直行、左拐、右拐
- action

参考paper: End-to-end Driving via Conditional Imitation Learning

### Reinforcement Learing
基于环境提供的奖励信号训练深度网络, 不需要人类driving traces. 使用A3C算法，其在仿真的三维环境中表现良好, 如赛车和三维迷宫中的导航。该算法的异步特性能并行运行多个仿真线程。

训练A3C进行目标导向的导航。每个episode，车辆要到达在Global Planner给出的目标。episode终止条件：车辆到达目标，车辆与障碍物相撞，用尽时间预算。奖励是5项的加权和：
- 正加权: 朝目标行驶的速度和距离
- 负加权: 碰撞、与人行道重叠、与对面车道重叠

## Experiments
- Four tasks:straight, one ture, navigation, navigation with dynamic obstacles，到达目标点为止。
- Six weather conditions, 分为training weather set和test wether set
- Town 1作为training，Town 2作为testing
- 在time budght内到达目标点视为episode成功，违规只会记录，不会终止。

## Result
1. 总体来看，三种方法的表现都不算好，即使是直行。原因：**传感器输入的可变性**。-> 新环境下的泛化
2. 泛化到新天气比泛化到新town容易。原因: 模型在多种天气条件下训练，但只在一个城镇训练。
3. modular pipeline(MP) and imitation learning(IL) perform on par on most tasks and conditions
	 - modular pipeline performs better under New weather 
	 - imitation learning performs better under New town and another case
	 - MP 更依赖perception stack是否正常工作。
4. RL效果差
	- RL is brittle and it is common to perform extensivetask-specific hyperparameter search
	- urban driving 比以前用RL解决的大多数任务更困难
	- 与模拟学习相比，RL的训练没有数据增强或正则化
5. infraction analysis. 设定评价标准: 两次违规之间的平均行驶距离(km）
	- opposite lane: IL好，RL很差
	- collision-pedestrian: RL最好，因为与行人碰撞有很大的negative reward
	- collision-static and car: RL差，MP好
6. 由于一些事故出现的频率太低，end-to-end训练效果不好，要更好的学习算法或者模型结构来提高robustness


