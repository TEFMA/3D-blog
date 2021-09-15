# create linemod dataset using ObjectDatasetTools
https://github.com/F2Wang/ObjectDatasetTools </br>
按照https://github.com/F2Wang/ObjectDatasetTools/blob/master/doc/installation.md 在Ubuntu16.04上安装

## 0.打印并粘贴markers
markers链接：https://github.com/TEFMA/3D-blog/blob/master/markers.pdf

## 1.采集数据
```
python record2.py LINEMOD/jsq
```
## 2.计算图片之间的位姿变化
```
python compute_gt_poses.py LINEMOD/jsq
```
## 3.option1:场景重建
```
python register_scene.py LINEMOD/jsq
```
## 3.option3.2:分割重建
```
python register_segmented.py LINEMOD/jsq
```
## 4.人工处理，泊松重建
## 5.生成label和mask
1.安装trimesh3.8.15(安装2.x.x会报困扰一下午的错误)
```
pip install trimesh==3.8.15
```
2.运行生成label和mask
```
python create_label_files.py LINEMOD/jsq
```
3.查看bbx和label的效果
```
python inspectMasks.py LINEMOD/jsq
```
## 6.创建训练集、测试集的文件目录
```
python makeTrainTestfiles.py LINEMOD/jsq
```
## 6.获取目标scale(max vertice distance)
```
python getmeshscale.py LINEMOD/jsq
```
## 7.获取bbx label
```
python get_BBs.py LINEMOD/jsq
```
