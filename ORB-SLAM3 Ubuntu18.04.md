# 准备
orb-slam3需要的依赖我们在《视觉SLAM十四讲》中已经全部安装好

1.需要注意的是将opencv3.1升级为了opencv4.0，具体安装步骤如下：

```
cd opencv-4.0.0
mkdir build
cd build
cmake ..
make
sudo make install
```
2.用ORB-SLAM3/Thirdparty/中自带的g2o和DBoW2无需单独编译安装，在编译ORB-SLAM3时会统一编译
3.不需要ananconda的python，需要python2.7版本
```
sudo apt install libpython2.7-dev
```


# 非ROS版本
## 编译ORB_SLAM3
```
cd ORB_SLAM3
chmod +x build.sh   
./build.sh          
```
## 问题解决
### 遇到 error: no match for ‘operator/’ (operand types are ‘cv::Matx<float, 3, 1>’ and ‘float’) x3D = x3D_h.get_minor<3,1>(0,0) / x3D_h(3); 问题
参考https://blog.csdn.net/Mwithz/article/details/117222990 解决，即在src/localMapping.cc和src/CameraModels/KannalaBrandt8.cpp中将
```
x3D = x3D_h.get_minor<3,1>(0,0) / x3D_h(3);
```
换成
```
x3D = cv::Matx31f(x3D_h.get_minor<3,1>(0,0)(0) / x3D_h(3), x3D_h.get_minor<3,1>(0,0)(1) / x3D_h(3), x3D_h.get_minor<3,1>(0,0)(2) / x3D_h(3));
```
即可


## 双目示例
下载数据
打开脚本"euroc_examples.sh",修改pathDatasetEuroc使其指向解压后的数据文件
执行该脚本
```
./euroc_examples.sh
```

## 单目示例
下载https://vision.in.tum.de/data/datasets/rgbd-dataset/download 中TUM数据集中的fr1/desk,解压到/home/leo/ORB_SLAM2文件夹下
 ```
./Examples/Monocular/mono_tum Vocabulary/ORBvoc.txt Examples/Monocular/TUM1.yaml ./rgbd_dataset_freiburg1_desk
```
## RGBD示例
```
./Examples/RGB-D/rgbd_tum Vocabulary/ORBvoc.txt Examples/RGB-D/TUM1.yaml ./rgbd_dataset_freiburg1_desk ./Examples/RGB-D/associations/fr1_desk.txt
```




# ROS版本
## 安装ROS melodic
参照wiki.ros.org/melodic/Installation/Ubuntu在ubuntu18.04 上安装  <br/>
在安装之前添加清华源：
```
sudo sh -c '. /etc/lsb-release && echo "deb http://mirrors.tuna.tsinghua.edu.cn/ros/ubuntu/ `lsb_release -cs` main" > /etc/apt/sources.list.d/ros-latest.list'
```
## 编译ORB_SLAM2
1.修改home/leo/ORB_SLAM2/Examples/ROS/ORB_SLAM2/CMakeLists.txt，在set(LIBS 后面添加：<br/>
```
/usr/lib/x86_64-linux-gnu/libboost_system.so  
/usr/lib/x86_64-linux-gnu/libboost_filesystem.so  
```
2.添加环境变量到~/.bashrc  <br/>
```
export ROS_PACKAGE_PATH=${ROS_PACKAGE_PATH}:home/leo/ORB_SLAM2/Examples/ROS
```
3.编译 <br/>
```
cd home/leo/ORB_SLAM2
chmod +x build_ros.sh
./build_ros.sh
```

## USB camera单目
### 安装usb_cam功能包
```
sudo apt-get install ros-melodic-usb-cam
```
任意位置(比如/home/leo/ORB_SLAM2下)建立usb_cam_node.launch文件： <br/>
```
<launch>
  <node name="usb_cam" pkg="usb_cam" type="usb_cam_node" output="screen" >
    <param name="video_device" value="/dev/video0" />
    <param name="image_width" value="640" />
    <param name="image_height" value="480" />
    <param name="pixel_format" value="yuyv" />
    <param name="camera_frame_id" value="usb_cam" />
    <param name="io_method" value="mmap" />
    <remap from="/usb_cam/image_raw" to="/camera/image_raw" />
  </node>
</launch>
```
### 运行单目SLAM
1.终端1
```
roscore
```
2.终端2
```
cd /home/leo/ORB_SLAM2
roslaunch usb_cam_node.launch
```
3.终端3
```
rosrun ORB_SLAM2 Mono /home/leo/ORB_SLAM2/Vocabulary/ORBvoc.txt /home/leo/ORB_SLAM2/Examples/ROS/ORB_SLAM2/Asus.yaml
``` 
其中需要修改Asus.yaml文件为自己USBcam的内参




## RGBD(realsense D435i)相机
### D435i SDK安装
1.参考https://github.com/IntelRealSense/librealsense/blob/master/doc/distribution_linux.md 安装   <br/>
```
sudo apt-key adv --keyserver keys.gnupg.net --recv-key F6E65AC044F831AC80A06380C8B3A55A6F3EFCDE || sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-key F6E65AC044F831AC80A06380C8B3A55A6F3EFCDE 
sudo add-apt-repository "deb http://realsense-hw-public.s3.amazonaws.com/Debian/apt-repo bionic main" -u  
sudo apt-get install librealsense2-dkms 
sudo apt-get install librealsense2-utils  
```
2.测试安装成功 <br/>
```
realsense-viewer  
```
3.安装dev package，With dev package installed, you can compile an application with librealsense using g++ -std=c++11 filename.cpp -lrealsense2 or an IDE of your choice.  <br/>
```
sudo apt-get install librealsense2-dev
sudo apt-get install librealsense2-dbg
```
4.更新
```
sudo apt-get update
sudo apt-get upgrade
```
### 安装realsense-ros
没有参考https://github.com/IntelRealSense/realsense-ros 安装，而是用apt方法安装：
```
sudo apt install ros-melodic-realsense2-camera
sudo apt install ros-melodic-realsense2-description
```
**发布图片**
```
发布全部图片(这里选用)
roslaunch realsense2_camera rs_camera.launch
发布点云
roslaunch realsense2_camera rs_camera.launch filters:=pointcloud
发布彩色深度图
roslaunch realsense2_camera rs_camera.launch filters:=colorizer
```
### topic匹配
查看realsense发布的topic
```
rostopic list
```
发现topic为：
```
/camera/color/image_raw
/camera/depth/image_rect_raw
```
修改/home/leo/ORB_SLAM2/Examples/ROS/ORB_SLAM2/src/ros_rgbd.cc:
```
将其中的/camera/rgb/image_raw替换成/camera/color/image_raw
将其中的/camera/depth_registered/image_raw替换成/camera/depth/image_rect_raw
```

### 运行RGBD SLAM

1.终端1
```
roscore
```
2.终端2
```
roslaunch realsense2_camera rs_camera.launch
```
3.终端3
```
rosrun ORB_SLAM2 RGBD /home/leo/ORB_SLAM2/Vocabulary/ORBvoc.txt /home/leo/ORB_SLAM2/Examples/ROS/ORB_SLAM2/Asus.yaml
``` 
其中需要修改Asus.yaml文件为自己USBcam的内参









########################################################33#3其他
【ORB-SLAM2】（三）：单目摄像头+ROS+ORB_SLAM2实时测试
https://blog.csdn.net/zhuiqiuzhuoyue583/article/details/107394461
ORB-SLAM2(3) ROS下实时跑ORB_SLAM2
https://www.cnblogs.com/kekeoutlook/p/7693129.html
ORB-SLAM2中ROS+RGBD运行实例
https://blog.csdn.net/qq_38589460/article/details/82708166

https://github.com/raulmur/ORB_SLAM2
#第三方学习资料：https://www.cnblogs.com/MingruiYu/tag/ORB-SLAM2/
