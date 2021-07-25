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
目前有8个地图[^2]。map包括城镇的 3D 模型及其道路定义，采用 OpenDRIVE 1.4 standard。[OpenDRIVE](https://www.asam.net/standards/detail/opendrive/)是一种文件格式，描述道路（网)的标准。

改变map，就是改变world。两种方式：`reload_world()`, `load_world()`

map包括以下四种要素:
- Landmarks: 即OpenDRIVE 文件中的交通标志。与之有关的类：[carla.Landmark](https://carla.readthedocs.io/en/latest/python_api/#carla.Landmark),  [carla.Waypoint](https://carla.readthedocs.io/en/latest/python_api/#carla.Waypoint),  [carla.Map](https://carla.readthedocs.io/en/latest/python_api/#carla.Map), [carla.World](https://carla.readthedocs.io/en/latest/python_api/#carla.World)
- Lanes: [carla.LaneType](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.LaneType), carla.LaneMarking, carla.LaneMarkingType, carla.LaneMarkingColor
- Junctions:  [carla.Junction](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.Junction)
- Waypoints: [carla.Waypoint](https://carla.readthedocs.io/en/latest/python_api/#carla.Waypoint)

`map = world.get_map()`只需要调用一次。Map很大，连续调用既不必要又昂贵。
Navigation是通过 waypoint API 来管理的，包括[carla.Waypoint](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.Waypoint) and [carla.Map](https://carla.readthedocs.io/en/0.9.11/python_api/#carla.Map)中的methods
## 4th- Sensors and data

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

Sensor是一种特殊的Actor，它的蓝图也是可以在蓝图库里边找到的，目前Carla已经支持了很多传感器，比如

-   摄像头： Depth， RGB ， Semantic segmentation
-   探测器： Collision ， Lane invasion ， Obstacle
-   其他： GNSS ， IMU ， LIDAR raycast ， Radar

传感器跟其他的Actor最大的不同是，它们需要被安装在车上，因此在生成传感器的时候，需要将其附着到一个车辆类型的Actor上，而出生点是针对于这台车本身的坐标系给定的。


```python
camera_bp = world.get_blueprint_library().find('sensor.camera.rgb')
camera = world.spawn_actor(camera_bp, 
                           carla.Transform(carla.Location(x=-5.5, z=2.5), carla.Rotation(pitch=8.0)), 
                           model3,
                           carla.AttachmentType.SpringArm
                           )
camera.listen(lambda image:image.save_to_disk('output/%06d.png' % image.frame)
```

我们在上段代码中，首先从蓝图库中找到RGB摄像头模板，然后利用这个蓝图生成摄像头Actor，并将其附着到前边生成好的Model 3上，我们选择了摄像头附着类型为：carla.AttachmentType.SpringArm，并将其位置设置到后方，这样我们就可以像从一个第三者的角度排到行驶的车辆了，在文章最后，读者可以看到摄像头拍到的车辆的照片。

每一个传感器都有一个listen方法，该方法接收一个callback作为参数，我们可以自定义callback里边的逻辑，callback将会在传感器拿到数据后被调用，并能够获取到这些数据。我们这个例子中，我们将摄像头采集到的图像信息以图片的形式保存在output文件夹中。

跟其他Actor一样的，传感器也有很多参数可以设置，比如对于RBG摄像头来说，我们可以设置其采集的分辨率。

[^1]: 史上最全Carla教程 |（三）基础API的使用 https://zhuanlan.zhihu.com/p/340031078
[^2]: at the bottom of this page https://carla.readthedocs.io/en/0.9.11/core_map/