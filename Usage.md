# Usage
## Start up
把Carla Library安装到python[^1]

运行python script之前，先运行CarlaUE4
```bash
cd carla # 进入carla目录
cd Unreal/CarlaUE4
~/UnrealEngine_4.24/Engine/Binaries/Linux/UE4Editor "$PWD/CarlaUE4.uproject"
```

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

# 基本概念
## 1. World and client
### Client
**client**  Python API [carla.Client](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.Client)，Methods有三种： basic, getter, setter
- 需要一个IP和两个Ports。第一个port是`n`， 第二个port则是`n+1`
- `time-out` 有些操作，如加载新地图，比较耗时，需要设置得久一点，比如10s的样子
- 可以有多个clients，但需要用到同步，也容易出现冲突，比如多个walkers。
- Client and server have different `libcarla` modules. 要**保证版本一致**.

`commands`系列methods提供了一些常用methods的变种，可以用于`apply_batch(), apply_batch_sync()`批处理函数。Python API 文档[末尾](https://carla.readthedocs.io/en/0.9.11/python_api/#commandapplyangularimpulse)列出了所有的`commands`系列methods。

### World
**world** is an object representing the simulation. 每个仿真环境只能有一个世界, 可以随时更改。Every world object has an `id` or episode. client每次调用`load_world()/ reload_world()`时，都会destroy old world，但UE并不会reboot

Python API [carla.World](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.World)

用途: 
- world包含了asset，而不是navigation map，后者属于`carla.Map` class
- 配置actors, weather, light
- debugging 仿真时绘制图形 [carla.DebugHelper](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.DebugHelper)
- snapshots, 给出A [carla.WorldSnapshot](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.WorldSnapshot) contains a [carla.Timestamp](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.Timestamp) and a list of [carla.ActorSnapshot](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.ActorSnapshot).
- [carla.WorldSettings](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.WorldSettings)

每个仿真都会先做这两步: 
1. client creation `carla.Client(), set_timeout()`
2. world connection: `get_world(), load_world()/ reload_world()`
Example:
```python3
# 创建Client
client = carla.Client('localhost', 2000)
# 防止连接时间过久
client.set_timeout(2.0)
# 获取world
world = client.get_world()
# 改变world，比如改变天气
weather = carla.WeatherParameters( 	cloudiness=80.0, 
									precipitation=30.0, 
									sun_altitude_angle=70.0) 
world.set_weather(weather) 
print(world.get_weather())
```
#### Weather
**Caution!** 天气本身不是一个类， but a set of parameters accessible from the world. 天气的变化**不会影响physics**，仅对相机有视觉上的影响。
[carla.WeatherParameters](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.WeatherParameters)用来自定义天气
- 比如上面的例子
- 直接指定预设好的天气，比如`world.set_weather(carla.WeatherParameters.WetCloudySunset)`
- `environment.py` _(in `PythonAPI/util`)_: 可以修改weather和light参数
- `dynamic_weather.py` _(in `PythonAPI/examples`)_: 启用map自带的天气周期


#### Lights
当`sun_altitude_angle < 0`，night mode 开始，灯光的影响就变得显著起来。
Light有两种: street light, vehicle light
##### Street Light
night mode下，street light自动打开
-  [**carla.Light**](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.Light) objects
-  [**carla.LightState**](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.LightState) - `light_state` 用来设置属性: color, intensity, etc
-  [**carla.LightGroup**](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.LightGroup) - `light_group` 用来分类：
-  [**carla.LightManager**](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.LightManager)

##### Vehicle Light
需要手动打开。不是所有的车都有light，
Python API [carla.VehicleLightState](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.VehicleLightState)
目前只有: 全部的Bikes, 两种Motorcycles, N种Cars有lights。
通过binary operations，使用methods: [carla.Vehicle.get_light_state](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.Vehicle.get_light_state) and [carla.Vehicle.set_light_state](https://carla.readthedocs.io/en/0.9.11/core_world/#python_api.md#carla.Vehicle.set_light_state) 

`environment.py`也可设置lights
## 2. Actors and blueprints
An **actor** is anything that plays a role in the simulation. 
Type of actors:
- sensors
- spectator 即Carla环境的视角，通过修改观察者的参数，切换Carla环境的视角。
- Traffic signs and traffic lights
- vehicles
- walkers

有三种操作：
- spawn 有了蓝图之后，意味着我们有了模板，然后就需要使用这个模板生成一个或者多个演员（Actor），这一过程叫做Spawning，因为Actor是在世界中存在的一个物体，因此在生成的时候，需要告诉环境它的出生点在哪里。
- handling 当Actor生成以后，就可以通过客户端来控制它的一些行为，比如可以让车子跑起来，并控制它的油门和转向。
- destroy: 不注销的话会一直存在于world，可能影响其他脚本的运行

```python
# 控制spectator
while running:
    spectator = world.get_spectator()
    transform = model3.get_transform()
	# 将观察者的位置到Model3的正上方（z轴）
    spectator.set_transform(carla.Transform(transform.location + carla.Location(z=50),
	# 将视角调为-90度，即向下看
    carla.Rotation(pitch=-90)))
	# 每隔5s跟着Model3的位置重置一下观察者的位置
    time.sleep(5)

```
**Blueprints**: 想生成一个Actor, 必须要先定义它的蓝图， 比如定义一辆奔驰汽车要先选择该型号的蓝图。所有的蓝图都存在 [Blueprint library](https://carla.readthedocs.io/en/0.9.11/bp_library/)

```python
############### Spawn Actor ###############
# 首先获取blueprint lib
blueprint_library = world.get_blueprint_library()
# 找到想要的blueprint， 比如汽车
ego_vehicle_bp = blueprint_library.find('vehicle.mercedes-benz.coupe')
# 设置blueprint
ego_vehicle_bp.set_attribute('color', '0, 0, 0')
# 随机选一个生成点
transform = random.choice(world.get_map().get_spawn_points())
# 生成汽车
ego_vehicle = world.spawn_actor(ego_vehicle_bp, transform)

############### Handle Actor ###############
# 移动汽车
location = ego_vehicle.get_location()
location.x += 10.0
ego_vehicle.set_location(location)
# 设置模式 - 自动驾驶
ego_vehicle.set_autopilot(True)
# 停止仿真
# actor.set_simulate_physics(False)

############## Destroy Actor ###############
# 注销单个Actor
ego_vehicle.destroy()
# 批处理
client.apply_batch([carla.command.DestroyActor(x) for x in actor_list])
```
## 3. Maps and navigation
**The map** is the object representing the simulated world, the town mostly. 

目前有8个城镇地图[^2]。
-   Town01：基本城镇，T型路口
-   Town02：类似Town01，更小
-   Town03：复杂城镇，5车道路口，环路，坡道，隧道
-   Town04：高速路和小镇的循环道路
-   Town05：带有交叉路口和桥的格子小镇。每个方向有多条车道，适合验证变道
-   Town06：长高速路，出入匝道
-   Town07：乡村环境，道路狭窄，少信号灯
-   Town10：高清城市环境

每个城镇有2种地图，即非分层地图和分层地图（后缀_Opt）。分层地图可以通过 PythonAPI 切换图层可见性。
```python
# Load layerred map for Town01 with minimum layout plus buildings and parked vehicles
time.sleep(3)
world = client.load_world('Town01_Opt', carla.MapLayer.Buildings | carla.MapLayer.ParkedVehicles)

# Toggle all buildings off
time.sleep(3)
world.unload_map_layer(carla.MapLayer.Buildings)

# Toggle all buildings on
time.sleep(3)
world.load_map_layer(carla.MapLayer.Buildings)
```
图层包含这些分组：
-   NONE 无
-   Buildings 建筑
-   Decals 贴花
-   Foliage 植被
-   Ground 地面
-   ParkedVehicles 停靠的车辆
-   Particles 粒子
-   Props 杂物
-   StreetLights 路灯
-   Walls 墙体
-   All 所有

map包括城镇的 3D 模型及其道路定义，采用 OpenDRIVE 1.4 standard。[OpenDRIVE](https://www.asam.net/standards/detail/opendrive/)是一种文件格式，描述道路（网)的标准。

改变map，就是改变world。两种方式：`reload_world()`, `load_world()`
```python
# 加载地图
world = client.load_world('Town01')
# world = client.reload_world()
# 获取可用地图列表
print(client.get_available_maps())
```

map包括以下四种要素:

### Landmarks
即OpenDRIVE 文件中的交通标志。与之有关的类：
- [carla.Landmark](https://carla.readthedocs.io/en/latest/python_api/#carla.Landmark)：对应 OpenDRIVE 的 signals
	- carla.LandmarkOrientation：根据道路的几何形状定义地标的方向
	- carla.LandmarkType：地标类型
- [carla.Map](https://carla.readthedocs.io/en/latest/python_api/#carla.Map) 可以查询地图中的所有地标，或者某一类型的地标
- [carla.World](https://carla.readthedocs.io/en/latest/python_api/#carla.World) 是 landmarks, carla.TrafficSign, carla.TrafficLight 的载体
```python
map = world.get_map()
print(map)
waypoints = map.generate_waypoints(100000)
print(waypoints)
waypoint = waypoints[4]
print(waypoint)
landmarks = waypoint.get_landmarks(20000.0, True)
print(landmarks)
```
### Lanes
[carla.LaneType](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.LaneType), carla.LaneMarking, 
- carla.LaneMarkingType OpenDRIVE 的类型枚举
- carla.LaneMarkingColor 标记颜色的枚举
- width 标记的厚度
- carla.LaneChange 声明执行车道更改的权限
```python
# Get the lane type where the waypoint is
lane_type = waypoint.lane_type
print(lane_type)

# Get the type of lane marking on the left
left_lanemarking_type = waypoint.left_lane_marking.type
print(left_lanemarking_type)

# Get available lane changes for this waypoint
lane_change = waypoint.lane_change
print(lane_change)
```
### Junctions
[carla.Junction](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.Junction)对应 OpenDRIVE junction。提供了一个边界框，描述其中车道和车辆的状态。为路口的每个车道返回一对waypoints，每对位于交界处的起点和终点。
```python
# 获取路口
junction = waypoint.get_junction()
print(junction)

# 获取路口范围的航路点
waypoints_junc = junction.get_waypoints(carla.LaneType.Any)
print(waypoints_junc)
```
### Waypoint
[carla.Waypoint](https://carla.readthedocs.io/en/latest/python_api/#carla.Waypoint)是 3D方向点。关联 world 和 OpenDRIVE。

waypoint包含了 carla.Transform和其他lane的信息。 

### Environment Objects
### Navigation
`map = world.get_map()`只需要调用一次。map很大，连续调用既不必要又昂贵。

Navigation是通过 waypoint API 来管理的，包括：
- [carla.Waypoint](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.Waypoint)得到waypoint，然后设置vehicle的位置。涉及到的methods有:
	-   next(d)：创建沿车道方向 d 距离的waypoint列表
	-   previous(d)：创建沿车道反方向 d 距离的waypoint列表
	-   next_until_lane_end(d), previous_until_lane_start(d)：返回从车道起点或终点相距 d 的waypoint列表
	-   get_right_lane(), get_left_lane()：返回相邻车道的等效waypoint，可以用于进行换道操作
- [carla.Map](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.Map)得到waypoint(s), 将模拟点转换为地理坐标, 保存道路信息

## 4th- Sensors and data
**Sensors** wait for some event to happen, and then gather data from the simulation. 
-   摄像头： Depth， RGB ， Semantic segmentation
-   探测器： Collision ， Lane invasion ， Obstacle
-   其他： GNSS ， IMU ， LIDAR raycast ， Radar

每种sensor详细请见[Sensor reference](https://carla.readthedocs.io/en/0.9.11/ref_sensors/)
[carla.Sensor](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.Sensor)
- 数据类型继承自[carla.SensorData](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.SensorData).
- 何时获取数据? 在每个模拟步骤或在某个事件发生时
- 如何获取数据？`listen()` method

1. setting: 获取blueprint，set_attribute(image resolution, field of view(`fov`), `sensor_tick`, etc)
2. spawning: attach(见下文)
3. listening: 定义callback函数

**Sensors should be attached to a parent actor**, usually a vehicle. `attachment_to`位置是相对parent actor而言的。  `attachment_type`有两种:
- Rigid attachment 通常用这种
- SpringArm attachment 仅推荐用于记录作用的录像

Example: Camera
```python
# add a camera
camera_bp = blueprint_library.find('sensor.camera.rgb')
# set attribute
camera_bp.set_attribute('image_size_x', '1920')
camera_bp.set_attribute('image_size_y', '1080')
camera_bp.set_attribute('fov', '110')
# camera relative position related to the vehicle
camera_transform = carla.Transform(carla.Location(x=1.5, z=2.4))
camera = world.spawn_actor(camera_bp, camera_transform, attach_to=ego_vehicle)

output_path = '../outputs/output_basic_api'
if not os.path.exists(output_path):
	os.makedirs(output_path)

# set the callback function
camera.listen(lambda image: image.save_to_disk(os.path.join(output_path, '%06d.png' % image.frame)))
sensor_list.append(camera)
```
# Synchrony and time-step
ref[^4][^5]
## Simulation time-step
Simulation没有现实中的时间概念，而是步长Time-step，两次仿真的间隔。有两种
- Variable time-step: default mode，尽可能快地运行
- Fixed time-step: the elapsed time remains constant between steps.
```python
############# Variable ###############
settings = world.get_settings() 
settings.fixed_delta_seconds = None # Set a variable time-step world.apply_settings(settings)
# 也可以通过PythonAPI/util/config.py配置
cd PythonAPI/util && python3 config.py --delta-seconds 0

############# Fixed ##################
settings = world.get_settings() 
settings.fixed_delta_seconds = 0.05 
world.apply_settings(settings)
# 也可以通过PythonAPI/util/config.py配置
cd PythonAPI/util && python3 config.py --delta-seconds 0.05
```
### Client-server synchrony
- **asynchronous mode**: server会尽可能跑得快，不管client。如果client过慢，可能导致server跑了三次，client才跑完一次
- **synchronous mode**: server会等待client完成的信号，再进行下一步模拟

开启同步模式：
```python
settings = world.get_settings() 
settings.synchronous_mode = True # Enables synchronous mode world.apply_settings(settings)
```
**Note**: 开启同步模式时，也要设置Traffic manager为同步模式。同步模式必须用fixed time-step。**与GPU相关的sensors, 如camera，必须用同步模式**

关闭同步模式:
也可用script，但script无法开启。
```bash
cd PythonAPI/util && python3 config.py --no-sync # Disables synchronous mode
```


|                   | Fixed time-step                                                        | Variable time-step                 |
| ----------------- | ---------------------------------------------------------------------- | ---------------------------------- |
| Synchronous mode  | 最常用，client主导| 几乎不用  |
| Asynchronous mode | Good time references for information. Server runs as fast as possible. | Non easily repeatable simulations. |

carla simulation默认模式为异步模式+variable time-step

[^1]: 史上最全Carla教程 |（三）基础API的使用 https://zhuanlan.zhihu.com/p/340031078
[^2]: at the bottom of this page https://carla.readthedocs.io/en/0.9.11/core_map/
[^3]:https://zhuanlan.zhihu.com/p/367491650
[^4]: https://carla.readthedocs.io/en/0.9.11/adv_synchrony_timestep/
[^5]: https://zhuanlan.zhihu.com/p/341521023