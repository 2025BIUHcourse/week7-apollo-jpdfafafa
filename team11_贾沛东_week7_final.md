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
![wending](./imges/wending.jpg)

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
### NDT算法定位
**1.使用脚本修复pose数据**
将corrected_poses.txt中的zone坐标使用python脚本修改为49
```python
import os
import sys
import glob
try:
from pyproj import Proj, transform
except ImportError:
sys.exit(1)
def convert_line(line, p_src, p_dst):
parts = line.strip().split()
if len(parts) < 5:
return line
try:
x_src = float(parts[2])
y_src = float(parts[3])
x_dst, y_dst = transform(p_src, p_dst, x_src, y_src)
parts[2] = "{:.6f}".format(x_dst)
parts[3] = "{:.6f}".format(y_dst)
return " ".join(parts) + "\n"
except Exception:
return line
def process_file(file_path):
try:
p_src = Proj(init='epsg:32651')
p_dst = Proj(init='epsg:32649')
except Exception:
sys.exit(1)
backup_path = file_path + ".bak"
if not os.path.exists(backup_path):
os.rename(file_path, backup_path)
source_file = backup_path
converted_lines = []
with open(source_file, 'r') as f:
for line in f:
converted_lines.append(convert_line(line, p_src, p_dst))
with open(file_path, 'w') as f:
f.writelines(converted_lines)
def main():
search_root =
"/home/apollo/apollo/modules/map/data/map_test/ndt_map/parsed_data"
if sys.version_info >= (3, 5):
search_pattern = os.path.join(search_root, "**", "corrected_poses.txt")
files = glob.glob(search_pattern, recursive=True)
else:
files = glob.glob(os.path.join(search_root, "*", "pcd",
"corrected_poses.txt"))
if not files:
return
for file_path in files:
process_file(file_path)
if __name__ == "__main__":
main()
```
**2.运行代码生图**
在容器内运行，生成ndtmap
```bash
bash supplement/ndt_map_creator.sh \
data/bag/localization \
/apollo/modules/calibration/data/dev_kit pix hooke/lidar_params/lidar_novatel_extrinsics.yaml \
49 \
/apollo/modules/map/data/map_test/ndt_map \
lidar
```
在容器内运行生成msf地图
```bash
bash supplement/msf_map_creator.sh \
data/bag/localization \
/apollo/modules/calibration/data/dev_kit pix hooke/lidar_params/lidar_novatel_extrinsics.yaml \
51 \
/apollo/modules/map/data/map_test \
lidar
```
**3.修正定位配置**
路径：modules/localization/conf/localization.conf
```conf
# 1. 指向包含 config.xml 和 map 文件夹的上一级目录
--map_dir=/apollo/modules/map/data/map_test
# 2. 确保 Zone ID 正确
--local_utm_zone_id=49
# 3. 开启激光定位开关
--enable_lidar_localization=true
```
**4.恢复DAG文件**
路径modules/localization/dag/dag_streaming_ndt_localization.dag
```dag
readers: [
{
# 必须是里程计/GPS话题，不能是 PointCloud2
channel: "/apollo/sensor/gnss/odometry"
...
}
]
```
**5.修改启动文件**
路径： /apollo/modules/calibration/data/dev_kit pix hooke/localization_dag/dag_streaming_ndt_localization.dag
```dag
module_config {
    module_library : "/apollo/bazel-bin/modules/localization/ndt/libndt_localization_component.so"
    components {
        class_name : "NDTLocalizationComponent"
        config {
            name : "ndt_localization"
            flag_file_path : "/apollo/modules/localization/conf/localization.conf"
            readers: [
                {
                    channel: "/apollo/sensor/gnss/odometry"
                    qos_profile: {
                        depth : 10
                    }
                    pending_queue_size: 50
                }
            ]
        }
    }
}
```
**6.启动NDT定位**
在容器内输入
```bash
mainboard -d modules/localization/dag/dag_streaming_ndt_localization.dag
```
![n](./imges/n.jpg)
**7.复制dag文件**
 将 /apollo/modules/localization/dag/dag_streaming_msf_visualizer.dag 文件复制粘贴
至 /apollo/modules/calibration/data/dev_kit pix hooke/localization_dag/ 文件夹下
**8.启动可视化**
在容器内输入
```bash
mainboard -d modules/localization/dag/dag_streaming_msf_visualizer.dag
```
![keshi](./imges/keshi.jpg)
![keshi1](./imges/keshi1.jpg)
### 定量评价定位效果
**1.修改脚本文件**
路径：/apollo/scripts/msf_local_evaluation.sh
```sh
# 第22行：LIDAR_LOC_TOPIC="/apollo/localization/msf_lidar"
LIDAR_LOC_TOPIC="/apollo/localization/ndt_lidar"
# 第25行：CLOUD_TOPIC="/apollo/sensor/velodyne64/compensator/PointCloud2"
CLOUD_TOPIC="/apollo/sensor/lidar/PointCloud2"
# 第47行：注释掉
#$APOLLO_BIN_PREFIX/modules/localization/msf/local_tool/data_extraction/
compare_poses \
# --in_folder $IN_FOLDER \
# --loc_file_a $GNSS_LOC_FILE \
# --loc_file_b $ODOMETRY_LOC_FILE \
# --imu_to_ant_offset_file "$ANT_IMU_FILE" \
# --compare_file "compare_gnss_odometry.txt"
# 取消91-93行的注释
echo ""
echo "Lidar localization result:"
python ${APOLLO_ROOT_DIR}/modules/tools/localization/evaluate_compare.py compare_lidar_odometry_all.txt
```
**2.重编译localization模块**
在容器内输入
```bash
bash apollo.sh build_opt localization
```
**3.录制数据**
在容器内输入
```bash
cyber_recorder record -a -i 600 -o ndt.record
```
ctrl+c退出
将数据文件放置在/apollo/data/bag/ndt
**4.运行脚本**
在容器内输入
```bash
bash scripts/msf_local_evaluation.sh data/bag/ndt
```
![j](./imges/j.jpg)
