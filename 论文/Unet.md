## 数据集

[WORD/README.md at main · HiLab-git/WORD](https://github.com/HiLab-git/WORD/blob/main/README.md)

原格式为.nii.gz





## 数据转换-nii.gz-to-png

+ 目的：在二维上训练分割模型，进而分割自己的数据集

为什么需要转换为png，而jpg不可以[使用PIL保存jpg图像重新打开读取发现像素值变了！！_pil里保存和打开图像存在损失-CSDN博客](https://blog.csdn.net/BUPT_Kwong/article/details/100972964)

因为jpg是有损压缩，png是无损压缩，在转换后可以准确映射像素值，才能确保分割种类信息



## Unet模型

[milesial/Pytorch-UNet: PyTorch implementation of the U-Net for image semantic segmentation with high quality images](https://github.com/milesial/Pytorch-UNet)

数据格式按代码默认要求;即索引：int+后缀：默认为''+'.*'

![image-20250112032024954](C:\Users\CYD\AppData\Roaming\Typora\typora-user-images\image-20250112032024954.png)

![image-20250112032237027](C:\Users\CYD\AppData\Roaming\Typora\typora-user-images\image-20250112032237027.png)

