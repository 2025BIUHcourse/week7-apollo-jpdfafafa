# apollo建图与定位
小组11 贾沛东
[TOC]
---
## apollo建图
### 录制数据集
**1.启动并进入apollo的docker容器**
在终端输入
```bash
cd ~/apollo
bash docker/scripts/dev_start.sh
bash docker/scripts/dev_into.sh
```
**2.启动dreamview网页**
在容器中输入
```bash
bash scripts/bootstrap.sh
```
**3.dreamview启动模块**
在dreamview中启动Transform，Lidar，GPS，Localization模块
在容器中通过monitor检查对应话题是否有信息
```bash
cyber_monitor
```
![monitor](./imges/monitor.jpg)

![apollo](./imges/apollo.jpg)

**4.组合导航驱动**
根据上周选做指导，使Gnss模式转为RTK稳定解定位定向

**5.记录数据**
在终端输入，开启recorder记录数据，驱动车走一段较长直线
```bash
cyber_recorder record -a -i 600 -o localization.record
```
ctrl+c退出
### 构建虚拟车道地图
**1.提取路径**
在容器中输入，在apollo目录下生成path.txt文件
```bash
./bazel-bin/modules/tools/map_gen/extract_path \
./path.txt \
data/bag/localization/*
```
**2.修正车道线宽度**
在apllo/modules/tools/map_gen/map_gen_single_lane.py中修改
```py
LANE_WIDTH = 5.0
```
**3.修复软件源，更新并安装缺少的依赖库**
在容器内输入更改文件
```bash
sudo vim /etc/apt/sources.list
```
![list](./imges/list.jpg)
在容器内输入
```bash
sudo apt update
sudo apt-get install tcl-dev tk-dev python3-tk
```

**4.生成图并可视化**
在终端输入
```bash
./supplement/build_lane_map.sh data/bag/localization/ map_test
```
![tu](./imges/tu.jpg)
在终端输入，dreamview重启后在地图菜单选择map_test显示地图
```bash
bash scripts/bootstrap.sh restart
```
![tu2](./imges/tu2.jpg)

## 定位
### RTK算法定位
**1.修改启动文件**
路径：/apollo/modules/calibration/data/dev_kit pix hooke/localization_dag/dag_streaming_rtk_localization.dag
![dag](./imges/dag.jpg)
**2.修改配置文件**
将/apollo/modules/localization/conf/localization.conf复制到/apollo/modules/calibration/data/dev_kit pix hooke/localization_conf/并将第5行和115行分别修改为
```bash
--map_dir=/apollo/modules/map/data/map_test
--local_utm_zone_id=49
```
ctrl+shift+f全局搜索zone_id全部把id改为49
**启动rtk定位**
在容器内输入
```bash
mainboard -d modules/localization/dag/dag_streaming_rtk_localization.dag
```
在monitor中检查
![mo](./imges/mo.jpg)
![po](./imges/po.jpg)
