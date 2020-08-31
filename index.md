---
title: 随机初始化单位向量
tags: [c++, mvs]
categories: [随笔记录]
mathjax: true
---
[test page](./test.md)

在改MVS的一些算法当中，随机初始化是一个非常常用的步骤。在基于patch match的mvs算法中，需要去构造一个深度平面，这个平面由深度以及平面单位法向量构成。在随机初始化这个平面单位法向量的时候，碰到了一些很有趣的问题以及一些思考，在此以做记录。

# 一种错误想法

我相信一拿到这个问题的时候，许多人跟我一样可能会这样做，分别随机在-1到1之间sample出法向量的x，y，z值，然后为了满足单位长度的要求再除以向量的模长。

```c++
std::random_device rd;
std::default_random_engine rng {rd()};
std::uniform_real_distribution<double> dis(-1, 1);
//get x y z of a vector
double x = dis(rng);
double y = dis(rng);
double z = dis(rng);
double norm = sqrt(x*x + y*y + z*z);
x /= norm;
y /= norm;
z /= norm;
```

但是稍加思索其实就会发觉这个想法其实是错误的，在独立随机三个坐标值时，其实是在一个以原点为中心的一个2\*2\*2的立方体中随机取点，这个点在这个立方体里的分布是均匀的，紧接着再将这个点投影到以原点为中心，半径为1的单位球面上。显然这样的投影是不均匀的，因为这个单位球和立方体相切，立方体除了这个单位球还多了四个边角的空间。虽然这个方法不正确，但是我们先画出这种方法最后出来的三坐标分量的概率分布，以下是用用上述方法取了10000个点，三坐标分量的分布直方图

|                            x分量                             |                            y分量                             |                            z分量                             |
| :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
| <img src="http://ww1.sinaimg.cn/large/c3b45047ly1gi92simvouj20x60pwmyc.jpg" width="400px"/> | <img src="http://ww1.sinaimg.cn/large/c3b45047ly1gi92szlc58j20wi0pqmyc.jpg" width="400px"/> | <img src="http://ww1.sinaimg.cn/large/c3b45047ly1gi92tk8bqlj20wy0paab8.jpg" width="400px"/> |

我们可以看到，三个坐标分量的分布都是一样的，但是经过我们这么折腾，原来均匀分布的变量就变成了非均匀分布。

# 正解

那么如何去实现呢，其实在前文当中答案已经呼之欲出了，既然我们知道问题在于多出来的四个角，那我们直接把它去掉不就行了

```c++
std::random_device rd;
std::default_random_engine rng {rd()};
std::uniform_real_distribution<double> dis(-1, 1);
//get x y z of a vector
double x = dis(rng);
double y = dis(rng);
double z = dis(rng);
double norm = sqrt(x*x + y*y + z*z);
while(norm > 1){
  x = dis(rng); y = dis(rng); z = dis(rng);
  norm = sqrt(x*x + y*y + z*z);
}
x /= norm;
y /= norm;
z /= norm;
```

下面是按照这种方法的三坐标分量的分布直方图

|                            x分量                             |                            y分量                             |                            z分量                             |
| :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
| <img src="http://ww1.sinaimg.cn/large/c3b45047ly1gi93aegwsfj20x20p83zl.jpg" width="400px"/> | <img src="http://ww1.sinaimg.cn/large/c3b45047ly1gi93apo1bij20wi0pugmp.jpg" width="400px"/> | <img src="http://ww1.sinaimg.cn/large/c3b45047ly1gi93b44a2fj20ws0q2dgx.jpg" width="400px"/> |

居然是均匀分布，也就是说我们初始的想法是对的，这样一个随机分布的向量，三个分量确实也是均匀分布的。这一点也可以用微积分证明。在计算球体表面积时，有以下过程，计算的是一个单位圆（半圆）绕x轴扫过的面积

<img src="http://ww1.sinaimg.cn/large/c3b45047ly1gi94nsb1onj216212ojv1.jpg" width="400px"/>

即球体的表面积可以表示为
$$
S = \int 2\pi y\sqrt{\Delta x^2 + \Delta y^2} = 2\pi \int_{r}^{r} \sqrt{r^2 - x^2} \sqrt{1 + \frac{\Delta y^2}{\Delta x^2}} dx
$$

由$x$和$y$之间的关系$y = \sqrt{r^2-x^2}$可得，$\frac{\Delta y}{\Delta x} = \frac{-x}{\sqrt{r^2 - x^2}}$，将其带入上式可得
$$
S = 2\pi \int_{-r}^{r} \sqrt{r^2 - x^2} \sqrt{1 + \frac{x^2}{r^2-x^2}} dx = 2\pi \int_{-r}^{r} r dx
$$
所以可以看出这个积分其实跟$x$没有关系，换句话说，在$x$的任一取值下，$\Delta x$囊括的球面积都是一样的，所以各个指向均匀分布的单位向量，其三坐标是在-1到1之间均匀分布的。

# 其他方法

## 高斯分布构造

和前面的方法差不多的想法，只是这次我们用标准高斯分布（正态分布）来构造三个坐标分量，这样构造出来的点在空间中的状态就类似于电子云，以原点是中心对称的。不过好处在于，这次不用像前述方法去掉在单位球以外的点。接着就和前述一样，将随机出来的点投影到单位球面上

```c++
std::normal_distribution<> dist; // mu: 0 sigma: 1
std::random_device rd;
std::default_random_engine rng {rd()};
plane->x = 1.0f * dist(rng);
plane->y = 1.0f * dist(rng);
plane->z = 1.0f * dist(rng);
float norm = sqrtf(plane->x * plane->x + plane->y * plane->y + plane->z * plane->z);
plane->x = plane->x / norm;
plane->y = plane->y / norm;
plane->z = plane->z / norm;
```

## Marsaglia方法

这种方法较为tricky，我暂时没有想明白具体的数学原理，在这留个坑，希望以后能补上。

```c++
float q1, q2, S = 2.f;
while (S >= 1.0f) {
	q1 = curand_between(ls, -1.0f, 1.0f); //一个cuda的随机取值函数，这里是随机从-1到1取值，意会就好不用在意细节
  q2 = curand_between(ls, -1.0f, 1.0f); 
  S = q1 * q1 + q2 * q2; 
}
const float sq = sqrtf(1.0f - S);
plane->x = 2.0f * q1 * sq;
plane->y = 2.0f * q2 * sq;
plane->z = 1.0f - 2.0f * S;
```

# 其他错误的尝试

## 随机取值两个欧拉角构造向量

在得到上述正解之前，其实还误碰了一些错误的构造方式，放在这里以提醒。

在单位球上的向量，自然而然会想到用欧拉角来构造，去掉一个没用的row角，剩下的pitch和yaw角就够用了。两个角度分别用alpha和beta来表示，如示意图，alpha的取值范围是$(0,2\pi)$，beta的取值范围是$(-\frac{\pi}{2},\frac{\pi}{2})$。

<img src="http://ww1.sinaimg.cn/large/c3b45047ly1gi40p2yahfj21ay15agrr.jpg" width="400px" />

那么这个单位向量就构造出来了

```c++
float Alpha, Beta;
Alpha = curand_between(ls, 0f, 2*M_PI);
Beta = curand_between(ls, -M_PI/2.0f, M_PI/2.0f);
plane->x = 1.0f * cos(Beta) * sin(Alpha);
plane->y = 1.0f * cos(Beta) * cos(Alpha);
plane->z = 1.0f * sin(Beta);
```

首先，从结果上看，得出来的三坐标分量就是错误的

|                            x分量                             |                            y分量                             |                            z分量                             |
| :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
| <img src="http://ww1.sinaimg.cn/large/c3b45047ly1gi9c2glffxj20wy0pomyg.jpg" width="400px"/> | <img src="http://ww1.sinaimg.cn/large/c3b45047ly1gi9c2rnv96j20ws0pwmyg.jpg" width="400px"/> | <img src="http://ww1.sinaimg.cn/large/c3b45047ly1gi9c39g1ayj20wo0q6759.jpg" width="400px"/> |

其次，还是可以通过确认单位alpha单位beta围的面积是否是不变的，来确认这样的取值是能够构造指向均匀分布的单位向量。答案是否定的，可以很清楚地看到，当alpha固定，beta越靠近0，$\Delta alpha, \Delta beta$围的面积越大。
