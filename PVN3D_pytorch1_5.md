# PVN3D pytorch1.5, the installtion and test
CUDA9,torch1.0 version：https://github.com/ethnhe/PVN3D  <br/>
CUDA10,torch1.5 version(我们采用的版本)：https://github.com/ethnhe/PVN3D/tree/pytorch-1.5

## 1. Installtion
refer to CUDA10,torch1.5 version：https://github.com/ethnhe/PVN3D/tree/pytorch-1.5
```
conda create -n PVN3D_pytorch1_5 python=3.6
source activate PVN3D_pytorch1_5
pip install Cython numpy
把requirements.txt中pprint注释掉, #pprint
把requirements.txt中的torch换成torch==1.5,torchvision换成torchvision==0.6.0
pip install -r requirements.txt
sudo apt install python3-tk

#https://blog.csdn.net/zsssrs/article/details/108492750
sudo apt-get install libpcl-dev pcl-tools
pip install python-pcl

git clone https://github.com/erikwijmans/Pointnet2_PyTorch
cd Pointnet2_PyTorch
pip install -r requirements.txt

cd PVN3D-pytorch-1.5
python setup.py build_ext
```

## 2. Datasets
### 2.1 create own datasets (like linemod type)
refer to https://github.com/TEFMA/3D-blog/blob/master/create_LINEMOD%20dataset.md <br/>
where: <br/>
1. record2.py <br/>
得到realsense内参，存入intrinsics.josn中 <br/>
存图像到JPEGImages/%s.jpg、depth/%s.png中 <br/>
2. compute_gt_poses.py <br/>
得到transforms.npy,里面保存了所有图片相对于世界坐标系的变换T，其中第一正图片的T=I(4,4),即世界系 <br/>
3. register_scene.py <br/>
每一帧的点云都用对应的T变换到了世界坐标系下(即第一帧下),然后配准得到scene.ply文件，对其滤波(自动+手动)、添加底面(自动)后保存为registeredScene.ply文件
4. create_label_files.py <br/>
(1)将原mesh(registeredScene.ply, OBB)转换成AABB中心化的新mesh(命名为object_name.ply),转换过程为Tform <br/>
(2)将没转换的mesh通过T变换，得到当前帧点云，再从中抽样很多点，用内参做投影，生成当前帧的image mask <br/>
(3)计算关于新mesh的4*4齐次变换矩阵(逆(Tform)*T)，存入transforms文件夹下 <br/>
(4)生成每帧的label file,以下数据是3D object边界框(AABB)投影到2D图片上的坐标<br/>
其格式如下： <br/>
classlabel, corner1(x,y), ..., corner8(x,y), center(x,y) <br/>
min_x: xxx<br/>
min_y: xxx<br/>
max_x: xxx<br/>
max_y: xxx<br/>
其中类标签可以被更改 <br/>
（5）另外，我自己添加了生成3D模型model_info.txt文件的代码，其中包含了3D模型的一些信息，这在PVN3D中是需要的 <br/>
其格式如下： <br/>
point1: min_x, min_y, min_z <br/>
point2: min_x, min_y, max_z <br/>
point3: min_x, max_y, min_z <br/>
point4: min_x, max_y, max_z <br/>
point5: max_x, min_y, min_z <br/>
point6: max_x, min_y, max_z <br/>
point7: max_x, max_y, min_z <br/>
point8: max_x, max_y, max_z <br/>
center: center[0],center[1],center[2] <br/>
diameter: distance(point1, point8) <br/>


### 2.3 Adaptation to this own dataset --- the important step!!! 
1.编译FPS脚本
```
cd lib/utils/dataset_tools/fps/
python setup.py build_ext --inplace
```
2.生成目标的信息, eg. radius, 3D keypoints, etc
```  
cd ../
python gen_obj_info.py /data/matengfei/3D/datasets/Linemod_preprocessed/models/obj_16.ply  /data/matengfei/3D/datasets/Linemod_preprocessed/models/model16_info
```
3.将/data/matengfei/3D/datasets/Linemod_preprocessed/models/model16_info/里的farthest.txt和corners.txt复制到PVN3D/pvn3d/datasets/linemod/lm_obj_kps/humidifier/里 <br/>
[//](note2:修正PVN3D/pvn3d/datasets/linemod/dataset_config/models_info.yml文件，使其包含id=16的目标信息，并与/data/matengfei/3D/datasets/Linemod_preprocessed/models/models_info.yml保持一致)

4.Modify info of this new dataset in PVN3D/pvn3d/common.py <br/>
line36: 修改n_total_epoch为5000 from 25
line37: 修改mini_batch_size为2 from 24
line39: 修改val_mini_batch_size为2 from 24
line91: 添加了index-id, 16 <br/>
line108: 添加了一行, 'humidifier':16, <br/>
line140: 修改了相机内参 <br/>

5.modify PVN3D/pvn3d/datasets/linemod/linemod_dataset.py as follows:<br/>
line223: 将png修改为jpg <br/>
line236: 去掉/1000.0 <br/>
修改main里的内容：
line385:修改为cls="humidifier"
line421:修改为kp_2ds = bs_utils.project_p3d(kp3d, 1.0, K) from (kp3d, cam_scale, K)
line248:修改为ctr_2ds = bs_utils.project_p3d(ctr3d[None, :], 1.0, K) from (ctr3d[None, :], cam_scale, K)

6.(Important!) Visualize and check if you process the data properly
```
python -m datasets.linemod.linemod_dataset
```
[//](7.For inference, make sure that you load the 3D keypoints, center point, and radius of your objects in the object coordinate system properly in PVN3D/pvn3d/lib/utils/pvn3d_eval_utils.py.)

8.修改pvn3d/lib/utils/basic_utils.py <br/>
line13: 修改了相机内参 <br/>
line523: 修改为pointxyz = self.ply_vtx(ptxyz_pth) #/ 1000.0

## 3. Training
(1)修改pvn3d文件夹下的train_linemod.sh(主要将类别修改对),然后执行
```
./train_linemod.sh
```
(2)或者执行
```
python -m train.train_linemod_pvn3d --cls humidifier
```

## 4. Test
修改pvn3d文件夹下的eval_linemod.sh(主要将类别修改对),然后执行
```
./eval_linemod.sh
```
## 5. Demo
将obj_16.ply转换为utf-8编码通过notepad++
修改pvn3d文件夹下的demo_linemod.sh(主要将类别修改对),然后执行
```
./demo_linemod.sh
```
以上配置若完全正确，可看到正确的训练和正确的demo投影







