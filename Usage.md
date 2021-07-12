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
