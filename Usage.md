把Carla Library安装到python[^1]

## 基本架构
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

## 基本概念
### 1. World and client
#### Client
**client**  Python API [carla.Client](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.Client)，Methods有三种： basic, getter, setter
- 需要一个IP和两个Ports。第一个port是`n`， 第二个port则是`n+1`
- `time-out` 有些操作，如加载新地图，比较耗时，需要设置得久一点，比如10s的样子
- 可以有多个clients，但需要用到同步，也容易出现冲突，比如多个walkers。
- Client and server have different `libcarla` modules. 要**保证版本一致**.

`commands`系列methods提供了一些常用methods的变种，可以用于`apply_batch(), apply_batch_sync()`批处理函数。Python API 文档[末尾](https://carla.readthedocs.io/en/0.9.11/python_api/#commandapplyangularimpulse)列出了所有的`commands`系列methods。

#### World
**world** is an object representing the simulation. 每个仿真环境只能有一个世界, 可以随时更改。Every world object has an `id` or episode. client每次调用`load_world(), reload_world()`时，都会destroy old world，但UE不会reboot
Python API [carla.World](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.World)
- world包含了asset，而不是navigation map，后者属于`carla.Map` class
- 配置actors, weather, light
- debugging 仿真时绘制图形 [carla.DebugHelper](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.DebugHelper)
- snapshots, 给出A [carla.WorldSnapshot](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.WorldSnapshot) contains a [carla.Timestamp](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.Timestamp) and a list of [carla.ActorSnapshot](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.ActorSnapshot).
- [carla.WorldSettings](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.WorldSettings)

每个仿真都会先做这两步: 
1. client creation `carla.Client(), set_timeout()`
2. world connection: `get_world(), load_world(), reload_world()`


### 2. Actors and blueprints
An **actor** is anything that plays a role in the simulation. 有三种操作：
- spawn
- handling
- destroy
**Blueprints**: 想生成一个Actor, 必须要先定义它的蓝图， 比如定义一辆奔驰汽车要先选择该型号的蓝图。所有的蓝图都存在 [Blueprint library](https://carla.readthedocs.io/en/0.9.11/bp_library/)

### 3. Maps and navigation

**The map** is the object representing the simulated world, the town mostly. There are eight maps available. All of them use OpenDRIVE 1.4 standard to describe the roads.

[OpenDRIVE](https://www.asam.net/standards/detail/opendrive/)是一种文件格式，描述道路（网)的标准。

**Roads, lanes and junctions** are managed by the [Python API](https://carla.readthedocs.io/en/0.9.11/python_api/) to be accessed from the client. These are used along with the **waypoint** class to provide vehicles with a navigation path.

**Traffic signs** and **traffic lights** are accessible as [**carla.Landmark**](https://carla.readthedocs.io/en/0.9.11/core_concepts/#python_api.md#carla.landmark) objects that contain information about their OpenDRIVE definition. Additionally, the simulator automatically generates stops, yields and traffic light objects when running using the information on the OpenDRIVE file. These have bounding boxes placed on the road. Vehicles become aware of them once inside their bounding box.

### 4th- Sensors and data

**Sensors** wait for some event to happen, and then gather data from the simulation. They call for a function defining how to manage the data. Depending on which, sensors retrieve different types of **sensor data**.

A sensor is an actor attached to a parent vehicle. It follows the vehicle around, gathering information of the surroundings. The sensors available are defined by their blueprints in the.

-   Cameras (RGB, depth and semantic segmentation).
-   Collision detector.
-   Gnss sensor.
-   IMU sensor.
-   Lidar raycast.
-   Lane invasion detector.
-   Obstacle detector.
-   Radar.
-   RSS.
[^1]: 史上最全Carla教程 |（三）基础API的使用 https://zhuanlan.zhihu.com/p/340031078