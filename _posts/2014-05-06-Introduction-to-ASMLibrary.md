---
layout: post
title: Introduction to ASMLibrary
tagline: null
category: null
tags: []
published: true
---
## Overview

Active Shape Model Library (ASMLibrary©) SDK, which is is developped under OpenCV for face alignment and face tracking. 

When you build ASM model, you must have an image database associated with its annotation files. ASMlibrary supports asf and pts format. For pts, you can refer to [BioId、XM2VTS、FGNet databases](http://personalpages.manchester.ac.uk/staff/timothy.f.cootes/tfc_software.html)  annotated by Time Cootes. For asf, please refer to [IMM database](http://www2.imm.dtu.dk/pubdb/views/publication_details.php?id=922)  annotated by Mikkel Stegmann. Generally speaking, The database should contain at least several hundreds images.  

Anyone who has the problem can goto the [Google Group](http://groups.google.com/group/asmlibrary) and let your question there. 

![image](/assets/post-images/null-cd0fb4cd-fb92-47de-cc6d-c66b462d0c91.jpg)

## A Quick Tutorial

### Build active shape models

    Usage: build [options] train_path image_ext point_ext cascade_name model_file 
 * Build asmmodel from 240-images of imm-database which you can download from Stagmman's homepage. 
 
    > build -i 0 -p 3 -l 8 -t 0.98 ../immdatabase bmp asf haarcascade_frontalface_alt2.xml aamapi.amf 

 * Build asmmodel from FRANCK-database which you can download from Cootes's homepage. 

    > build -i 1 -p 4 -l 8 ../franck jpg pts haarcascade_frontalface_alt2.xml franck.amf  

### Fit using active shape models

    Usage: fit -m model_file -h cascade_file {-i image_file | -v video_file | -c camara_idx} -n n_iteration

 * Image alignment on an image using 30 iterations

    > fit -m my.amf -h haarcascade_frontalface_alt2.xml -i aa.jpg -n 30

 * Face tracking on a video file

    > fit -m my.amf -h haarcascade_frontalface_alt2.xml -v bb.avi -n 25

 * Face tracking on live camara

    > fit -m my.amf -h haarcascade_frontalface_alt2.xml -c 0 

### Download

Stable version：[asmlibrary-6.0.tar.gz](https://github.com/greatyao/asmlibrary/archive/master.zip) 

View source code at [github.com](https://github.com/greatyao/asmlibrary)

### Donate ###

If you like the library, please Donate via [PayPal](https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=CS9NXZETE7X4L).

### 使用说明 ###

ASMLibrary包括离线训练模型和在线实时匹配两大部分。 

训练模型之前，确保你拥有一个图像数据库以及对应于该图像数据库的标定文件库。所谓的图像标定指对图像进行点的标定，指示了相关特征点的具体坐标位置，支持的文件格式有pts和asf，pts的示例可以参看Time Cootes对BioId、XM2VTS、FGNet等数据库的标定；asf的示例可以参看Mikkel Stegmann对IMM图像库进行的标定。 一般而言，标定图像数据库的规模在几百张就可以了。 

当模型训练完毕之后，可以对任意图像进行人脸对齐，或者对视频文件或者摄像头进行特征点跟踪。 

如果你认可ASMLibrary库，支持ASMLibrary的继续更新和改进，欢迎你以自愿捐助方式进行捐助。 捐助链接设置在[支付宝](https://me.alipay.com/asmlibrary)。
