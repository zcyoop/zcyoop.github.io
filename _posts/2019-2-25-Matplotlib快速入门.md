---
layout: post
title:  'Matplotlib快速入门'
date:   2019-2-25 22:38:54
tags: python
categories: [编程语言,python]
---



## 什么是 Matplotlib?

简单来说，Matplotlib 是 Python 的一个绘图库。它包含了大量的工具，你可以使用这些工具创建各种图形，包括简单的散点图，正弦曲线，甚至是三维图形。Python 科学计算社区经常使用它完成数据可视化的工作。

你可以在他们的[网站](http://matplotlib.org/)上了解到更多 Matplotlib 背后的设计思想，但是我强烈建议你先浏览一下他们的[图库](http://matplotlib.org/gallery.html)，体会一下这个库的各种神奇功能。

### Matplotlib使用

#### 导包

```
import matplotlib.pyplot as plt
import numpy as np
```

#### 简单的绘图

```python
# 从 -10 到 10 等分成100份
x = np.linspace(-10,10,100)
# 分别这100个值的cos&sin
cos_y=np.cos(x)
sin_y=np.sin(x)

# 前连个参数分别对应 x轴y轴，第三个参数代表着红色虚线，第四个参数代表这条线的名称
plt.plot(x,cos_y,'r--',label='cos_y')
plt.plot(x,sin_y,'b--',label='sin_y')

plt.legend() # 图上显示这条线的名称
plt.title('cos sin demo') # 图的标题
plt.show() 
```

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/2019-2-25/matplotlib_1.png)

>颜色： 蓝色 - 'b' 绿色 - 'g' 红色 - 'r' 青色 - 'c' 品红 - 'm' 黄色 - 'y' 黑色 - 'k'（'b'代表蓝色，所以这里用黑色的最后一个字母） 白色 - 'w'线： 直线 - '-' 虚线 - '--' 点线 - ':' 点划线 - '-.' 常用点标记 点 - '.' 像素 - ',' 圆 - 'o' 方形 - 's' 三角形 - '^' 更多点标记样式点击[这里](http://matplotlib.org/api/markers_api.html)



#### 正态分布图

```python
# 正态分布的中心，标准差，生成的个数
x=np.random.normal(0,1,100000)
plt.hist(x,100) # 传入数组，容器个数
plt.show
```

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/2019-2-25/matplotlib_2.png)



#### 参考

- [十分钟入门Matplotlib](http://codingpy.com/article/a-quick-intro-to-matplotlib/)