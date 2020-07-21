---
title: Darknet中文化标签
date: 2020-07-21 10:18:16
tags:
	- Project
	- Notes
---

Darknet目前是基于Yolo_v4的物体识别框架, 因为网络模型简单, 识别速度精度高, 兼容性好, 文档齐全, 社区活跃等原因对新手友好相对而广泛使用. 这里我介绍使用中文标签的方法.

![Darknet 中文化标签](https://i.postimg.cc/2SYXtphc/Pic-01.jpg "Darknet with Chinese label")

<!-- More -->

# 中文化的原理
Darknet使用字符图片拼接的方法, 例如cat是c a t各自的label连接而成. 中文远不止26个英文字符(及符号), 所以这里采用了简易的的class label, 好处是图片是静态生成的, 一次生成多次使用, 坏处是不能动态地根据可用class来生成对应labels.

## 获取 Darknet

```
git clone https://github.com/AlexeyAB/darknet.git
```

## labels生成代码

### 类别名称
将你的`.names`文件修改为中文, 本文以`data/obj.names`为例

### 安装imagemagick
* Mac

```
brew install imagemagick
```

* Linux

```
wget https://www.imagemagick.org/download/ImageMagick.tar.gz
tar xf ImageMagick.tar.gz
cd ImageMagick-*
./configure
make
make install
ldconfig /usr/local/lib
```

* Windows
[Win64安装包](https://imagemagick.org/download/binaries/ImageMagick-7.0.10-24-Q16-HDRI-x64-dll.exe)
[VC++](https://aka.ms/vs/16/release/vc_redist.x64.exe)


### 生成标签

删除darknet/data/labels里所有内容, 新建`make_labels.py`, 内容如下:

```
import os

# 字体位置
f = "/System/Library/Fonts/Supplemental/Songti.ttc"


l = []
with open("../obj.names") as list_in:
    for line in list_in:
        l.append(line[:-1])


def make_labels(s):
    for idx, cls in enumerate(l):
        os.system("convert -fill black -background white -bordercolor white -border 4 -font %s -pointsize %d label:\"%s\" \"%d_%d.png\"" % (f, s, cls, idx, s/12-1))


for i in [12, 24, 36, 48, 60, 72, 84, 96]:
    make_labels(i)
```

修改完字体位置后运行`python make_labels.py`即可.
![标签图](https://i.postimg.cc/pTFnFLkq/Pic-02.png "Label images")

# 修改源代码

## src/Image.c
这一步让darknet从找字符label变为找class label.
1. 修改`get_label_v3`函数为 

```
image get_label_v3(image **characters, char *labelindex, int size)
{
    size = size / 10;
    if (size > 7) size = 7;
    image label = make_empty_image(0, 0, 0);
    int class, i = 0, nlabels = 1;
    int len = strlen(labelindex);
    for(i = 0; i < len; i++)
    {
        if (labelindex[i] == ',') ++nlabels;
    }

    for(i = 0; i < nlabels; i++){
        class = atoi(labelindex);
        image l = characters[size][class];
        image n = tile_images(label, l, -size - 1 + (size + 1) / 2);
        free_image(label);
        label = n;
        labelindex=strchr(labelindex, ',') + 1;
    }
    image b = border_image(label, label.h*.02);
    free_image(label);
    return b;
}
```

2. 新建`**load_labels`函数以加载标签图, 建议在`**load_alphabet()`附近添加, 方便维护.

```
image **load_labels(int classes)
{
    int i, j;
    const int nsize = 8;
    image** alphabets = (image**)xcalloc(nsize, sizeof(image*));
    for(j = 0; j < nsize; ++j){
        alphabets[j] = (image*)xcalloc(classes, sizeof(image));
        for(i = 0; i < classes; ++i){
            char buff[256];
            sprintf(buff, "data/labels/%d_%d.png", i, j);
            alphabets[j][i] = load_image_color(buff, 0, 0);
        }
    }
    return alphabets;
}
```

3. 在`draw_detections_v3`下寻找`if (alphabet) {`语句, 并修改其内容为

```
if (alphabet) {
  char labelindex[100] = {0};
  char class[10] = {0};
  sprintf(class, "%d", selected_detections[i].best_class);
  strcat(labelindex, class);
  int j;
  for (j = 0; j < classes; ++j) {
    if (selected_detections[i].det.prob[j] > thresh && j != selected_detections[i].best_class) {
      strcat(labelindex, ", ");
      sprintf(class, "%d", j);
      strcat(labelindex, class);
    }
  }
  image label = get_label_v3(alphabet, labelindex, (im.h*.01));
  draw_weighted_label(im, top + width, left, label, rgb, 0.7);
  free_image(label);
 }
```

## src/Image.h
1. 为了声明新建的`**load_labels`函数, 在`image **load_alphabet();`下一行添加

```
image **load_labels(int);
```

##  src/Detector.c
1. 替换标签图加载函数, 把`image **alphabet = load_alphabet();`替换为

```
image **alphabet = load_labels(names_size);
```

2. 修改释放变量的范围, 在`test_detector`函数下寻找`for (i = 32; i < 127; ++i) {`替换为

```
for (i = 0; i < names_size; ++i) {
```

## 类别名称乱码处理
Windows下类别名称文件`obj.names`使用GB2312保存, *nix下则使用UTF-8保存.


# 进行识别
至此中文化就完成了, 可以`make`后进行图像识别.


仓库链接: [Github](https://github.com/9r0x/darknet/commit/8da809d3f31c47d8766180fa2081c5e2325dc19c)
