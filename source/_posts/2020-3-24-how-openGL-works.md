---
title: How openGL work
date: 2020-03-24
tags:
  - openGL
  - graphics
---

本文的主要内容都来自于对[1](#jump)的学习和整理。从画点开始介绍openGL的工作原理。

<!--more-->

## 起点

我们的目的是了解openGL的工作原理，重心是如何将三维模型渲染到二维的平面上，不是如何建立三维模型。所以，我们一开始只拥有两样东西，一样是格式为TGA的三维模型文件, 另一样是可以在像素点是涂上颜色的函数。

比如，下面的代码将一个像素点染色。

``` C++
#include "tgaimage.h"
const TGAColor white = TGAColor(255, 255, 255, 255);
const TGAColor red   = TGAColor(255, 0,   0,   255);
int main(int argc, char** argv) {
        TGAImage image(100, 100, TGAImage::RGB);
        image.set(52, 41, red);
        image.flip_vertically(); // i want to have the origin at the left bottom corner of the image
        image.write_tga_file("output.tga");`
        return 0;
}
```

所有的代码都在这个[仓库](https://github.com/ssloy/tinyrenderer)中。

## 直线渲染

已知两个点确定一条直线。公式如下：
$$ y=y_0 + (y_1 - y_0)/(x_1 - x_0)*(x - x_0) $$

根据上面公式,我们可以写出下面的代码。

``` C++
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) { 
    for (int x=x0; x<=x1; x++) { 
        int y = y0 + (y1 - y0)/(x1 - x0)*(x - x0); 
        image.set(x, y, color); 
    }
}
```

上面的代码有好几个问题，先不考虑数据类型导致的精度丢失和除零错。跟画图相关的问题有两个。

1. for循环蕴含了x0<=x1
2. 如果直线斜率过大，两个点的在y轴上的差值大于1，渲染出来的直线呈点状。如下图所示。

![line.png](line.png)

因此在画图前需要对点和坐标做一定的处理。修改后的代码如下：

``` C++
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) { 
    bool steep = false; 
    if (std::abs(x0-x1)<std::abs(y0-y1)) { // if the line is steep, we transpose the image 
        std::swap(x0, y0); 
        std::swap(x1, y1); 
        steep = true; 
    } 
    if (x0>x1) { // make it left−to−right 
        std::swap(x0, x1); 
        std::swap(y0, y1); 
    } 
    for (int x=x0; x<=x1; x++) { 
        int y = y0 + (y1 - y0)/(x1 - x0)*(x - x0); 
        if (steep) { 
            image.set(y, x, color); // if transposed, de−transpose 
        } else { 
            image.set(x, y, color); 
        } 
    } 
}
```

由于每两个在x轴上连续的像素点只相差1，所以我们的代码可以优化如下：

``` C++
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) { 
    bool steep = false; 
    if (std::abs(x0-x1)<std::abs(y0-y1)) { 
        std::swap(x0, y0); 
        std::swap(x1, y1); 
        steep = true; 
    } 
    if (x0>x1) { 
        std::swap(x0, x1); 
        std::swap(y0, y1); 
    } 
    int dx = x1-x0; 
    int dy = y1-y0; 
    float derror = std::abs(dy/float(dx)); 
    float error = 0; 
    int y = y0; 
    for (int x=x0; x<=x1; x++) { 
        if (steep) { 
            image.set(y, x, color); 
        } else { 
            image.set(x, y, color); 
        } 
        error += derror; 
        if (error>.5) { 
            y += (y1>y0?1:-1); 
            error -= 1.; 
        } 
    } 
} 
```

只有当累计误差大于0.5的时候下一个点的y值加减1。

现在再来解决除零错和浮点类型的问题。方法是将误差值乘以2*dx。下面是我最终的代码：

``` C++
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color)
{
	bool isSteep = false;
	if (std::abs(x1 - x0) < std::abs(y1 - y0))
	{
		std::swap(x0, y0);
		std::swap(x1, y1);
		isSteep = true;
	}

	if (x0 > x1)
	{
		std::swap(x0, x1);
		std::swap(y0, y1);
	}

	int y = y0;
	int derror = 2 * std::abs(y1 - y0);
	int error = 0;
	for (int x = x0; x <= x1; x++)
	{
		if (isSteep)
		{
			image.set(y, x, color);
		}
		else
		{
			image.set(x, y, color);
		}
		error += derror;
		if (error + x0 >= x1)
		{
			y += y1 > y0 ? 1 : -1;
			error -= 2 * (x1 - x0);
		}
	}
}
```

## 三角形渲染

渲染完直线后，下一步是三角形，三维模型都是由三角形构建出来的，因此三角形的渲染是比较重要的。

三角形的渲染就是把三条线围住的像素点着色。

最容易想到的着色方法是线扫描算法。顾名思义，就是一行一行地把像素点染色。算法麻烦的地方在于找出需要着色像素的起点和终点。

这个方法不详细介绍了，直接给出代码。

``` C++
void triangle(Vec2i t0, Vec2i t1, Vec2i t2, TGAImage &image, TGAColor color) { 
    if (t0.y==t1.y && t0.y==t2.y) return; // I dont care about degenerate triangles 
    // sort the vertices, t0, t1, t2 lower−to−upper (bubblesort yay!) 
    if (t0.y>t1.y) std::swap(t0, t1); 
    if (t0.y>t2.y) std::swap(t0, t2); 
    if (t1.y>t2.y) std::swap(t1, t2); 
    int total_height = t2.y-t0.y; 
    for (int i=0; i<total_height; i++) { 
        bool second_half = i>t1.y-t0.y || t1.y==t0.y; 
        int segment_height = second_half ? t2.y-t1.y : t1.y-t0.y; 
        float alpha = (float)i/total_height; 
        float beta  = (float)(i-(second_half ? t1.y-t0.y : 0))/segment_height; // be careful: with above conditions no division by zero here 
        Vec2i A =               t0 + (t2-t0)*alpha; 
        Vec2i B = second_half ? t1 + (t2-t1)*beta : t0 + (t1-t0)*beta; 
        if (A.x>B.x) std::swap(A, B); 
        for (int j=A.x; j<=B.x; j++) { 
            image.set(j, t0.y+i, color); // attention, due to int casts t0.y+i != A.y 
        } 
    } 
}
```

在现实中，我们可能需要一些更容易在成千上万线程中并行渲染的方法。这意味着这个算法需要能够很容易地将大问题切分成很多的小问题。因此有时候我们需要一些看上去很“白痴”的方法。比如下面的伪代码：

``` C++
triangle(vec2 points[3]) { 
    vec2 bbox[2] = find_bounding_box(points); 
    for (each pixel in the bounding box) { 
        if (inside(points, pixel)) { 
            put_pixel(pixel); 
        } 
    } 
}
```

这个方法先找到一个方框将三角形圈起来。然后判断每个点是否需要染色。

方法的难点在于如何判断一个像素点是否在三角形中，这时候我们就需要[重心坐标](https://en.wikipedia.org/wiki/Barycentric_coordinate_system)。

重心坐标在平面上即用三角形的三个顶点来表示坐标。设三个顶点为A,B,C。P为平面上一点。

若P=xA+yB+zC 且 x+y+z=1。那么(x,y,z)就是P的重心坐标。

重心坐标有个重要的性质，三角形中的点在重点坐标中的坐标值都/大于0。利用这个性质就可以判断点是否在三角形中。

剩下的问题就是如何将直角坐标转换为重心坐标了。

根据定义可得

$$ p=(1-y-z)A+yB+zC $$

化简得

$$ y\vec{BA}+z\vec{CA}+\vec{AP} = 0 $$

即

$$ y\vec{BA}_x+z\vec{CA}_x+\vec{AP}_x = 0 $$
$$ y\vec{BA}_y+z\vec{CA}_y+\vec{AP}_y = 0 $$

所以(y,z,1) 和 (\vec{BA}_x,\vec{CA}_x,\vec{AP}_x)(\vec{BA}_y,\vec{CA}_y,\vec{AP}_y)都正交。

所以(y,z,1) 可以通过后两者得叉乘得到。下面是求重心坐标和渲染三角形的代码。

``` C++
Vec3f barycentric(Vec2i *pts, Vec2i P) { 
    Vec3f u = cross(Vec3f(pts[2][0]-pts[0][0], pts[1][0]-pts[0][0], pts[0][0]-P[0]), Vec3f(pts[2][1]-pts[0][1], pts[1][1]-pts[0][1], pts[0][1]-P[1]));
    /* `pts` and `P` has integer value as coordinates
       so `abs(u[2])` < 1 means `u[2]` is 0, that means
       triangle is degenerate, in this case return something with negative coordinates */
    if (std::abs(u[2])<1) return Vec3f(-1,1,1);
    return Vec3f(1.f-(u.x+u.y)/u.z, u.y/u.z, u.x/u.z); 
} 
 
void triangle(Vec2i *pts, TGAImage &image, TGAColor color) { 
    Vec2i bboxmin(image.get_width()-1,  image.get_height()-1); 
    Vec2i bboxmax(0, 0); 
    Vec2i clamp(image.get_width()-1, image.get_height()-1); 
    for (int i=0; i<3; i++) { 
        for (int j=0; j<2; j++) { 
            bboxmin[j] = std::max(0,        std::min(bboxmin[j], pts[i][j])); 
            bboxmax[j] = std::min(clamp[j], std::max(bboxmax[j], pts[i][j])); 
        } 
    } 
    Vec2i P; 
    for (P.x=bboxmin.x; P.x<=bboxmax.x; P.x++) { 
        for (P.y=bboxmin.y; P.y<=bboxmax.y; P.y++) { 
            Vec3f bc_screen  = barycentric(pts, P); 
            if (bc_screen.x<0 || bc_screen.y<0 || bc_screen.z<0) continue; 
            image.set(P.x, P.y, color); 
        } 
    } 
} 
```

在渲染三角形的过程中，我们还需要考虑每个三角形的着色。三维模型的颜色又跟纹理和光照强度有关系。

我们先考虑光照强度的影响。光照强度等于光向量和平面法向量的点积。下面是考虑光照强度后的渲染代码。

``` C++
for (int i=0; i<model->nfaces(); i++) { 
    std::vector<int> face = model->face(i); 
    Vec2i screen_coords[3]; 
    Vec3f world_coords[3]; 
    for (int j=0; j<3; j++) { 
        Vec3f v = model->vert(face[j]); 
        screen_coords[j] = Vec2i((v.x+1.)*width/2., (v.y+1.)*height/2.); 
        world_coords[j]  = v; 
    } 
    Vec3f n = (world_coords[2]-world_coords[0])^(world_coords[1]-world_coords[0]); 
    n.normalize(); 
    float intensity = n*light_dir; 
    if (intensity>0) { 
        triangle(screen_coords[0], screen_coords[1], screen_coords[2], image, TGAColor(intensity*255, intensity*255, intensity*255, 255)); 
    } 
}
```

下面是通过上面代码渲染出来的一个模型。

![head.png](head.png)

在模型中，口的内腔部分画在了嘴唇上，这是因为我们将原本不可见的部分覆盖在了可见的部分上。

## 深度缓冲(z-buffer)

深度缓冲的概念很简单。就是用一段内存空间将同一像素点对应的多个点在z轴上的最大值缓存起来。只渲染z轴上值最大那个点。

代码改动比较少，我写的代码如下

``` C++
void triangle(Vec3i *pts, TGAImage &image, TGAColor color, int *zbuffer)
{
	std::sort(pts, pts + 3, [](Vec3i a, Vec3i b) -> bool { return a.x < b.x; });
	int smallx = pts[0].x;
	int largex = pts[2].x;
	std::sort(pts, pts + 3, [](Vec3i a, Vec3i b) -> bool { return a.y < b.y; });
	int smally = pts[0].y;
	int largey = pts[2].y;
	for (int x = smallx; x <= largex; x++)
		for (int y = smally; y <= largey; y++)
		{
			Vec3f bc_screen = barycentric(pts, Vec2i(x, y));
			if (bc_screen.x < 0 || bc_screen.y < 0 || bc_screen.z < 0)
				continue;
			float z = bc_screen * Vec3f(pts[0].z, pts[1].z, pts[2].z);
			if (zbuffer[x + width * y] < z)
			{
				zbuffer[x + width * y] = z;
				image.set(x, y, color);
			}
		}
}
```

## 透视投影

图形运动造成的变化可以转化为各种几何变换。

线性变换(如旋转缩放）可以写成矩阵A和点x的乘积

$$ f(x)=Ax $$

平移变换可以写成点x和向量b加法

$$ f(x)=x+b $$

两者结合称为仿射变换，写成

$$ f(x)=Ax+b $$

如果两个仿射变换叠加

$$ f_1(f(x))=A(A_1x+b_1)+b $$

这个表达式非常丑，如果叠加更多的仿射变换会变得更加复杂。这十分不利于理解和运算。

解决方法是引入齐次坐标系。意思是增加一维坐标。

比如在平面上,仿射变换是这样表示的

$$
 \left[
 \begin{matrix}
   a & b \\
   c & d 
  \end{matrix}
  \right]
  \left[
 \begin{matrix}
   x \\
   y
  \end{matrix}
  \right] + 
  \left[
 \begin{matrix}
   e \\
   f
  \end{matrix}
  \right] =
  \left[
 \begin{matrix}
   ax+by+e \\
   cx+dy+f
  \end{matrix}
  \right]
$$

在齐次坐标系上我们增加一维，变成

$$
 \left[
 \begin{matrix}
   a & b & e \\
   c & d & f \\
   0 & 0 & 1
  \end{matrix}
  \right]
  \left[
 \begin{matrix}
   x \\
   y \\
   1 \\
  \end{matrix}
  \right] =
  \left[
 \begin{matrix}
   ax+by+e \\
   cx+dy+f \\
   1
  \end{matrix}
  \right]
$$

这样仿射变换的叠加就变成了矩阵的相乘，从齐次坐标变回直角坐标也很简单

$$
  \left[
  \begin{matrix}
   x \\
   y \\
   z \\
  \end{matrix}
  \right] \to
  \left[
 \begin{matrix}
   x/z \\
   y/z \\
  \end{matrix}
  \right]
$$

在坐标系中相当于三维坐标的点中心投影到z=1的平面上。如图所示

![homogeneous.png](homogeneous.png)

在这种投影方法上，我们发现z=0平面上的点对应的是z=1平面上的点的无穷远点。

而且我们现在可以区分出向量和点这两个概念了。
在直角坐标系中，(x,y)可以是点也可以是向量。
在齐次坐标系中，(x,y,0)代表向量，(x,y,z) z!=0 代表点。

回到三维的世界中， 三维直角坐标可以用四维齐次坐标表示。

如图所示，我们将P点映射到平面z=0上。

![projection.png](projection.png)

可以求出

$$ x^{'} = x/(1-z/c) $$ 

$$ y^{'} = y/(1-z/c) $$ 

转化为齐次坐标的矩阵就是

$$
 \left[
 \begin{matrix}
   1 & 0 & 0 & 0 \\
   0 & 1 & 0 & 0 \\
   0 & 0 & 1 & 0 \\
   0 & 0 & r & 1
  \end{matrix}
  \right]
  \left[
 \begin{matrix}
   x \\
   y \\
   z \\
   1 \\
  \end{matrix}
  \right] =
  \left[
 \begin{matrix}
   x \\
   y \\
   z \\
   rz+1
  \end{matrix}
  \right]
$$

其中$$r=-1/c$$

## 移动镜头

移动镜头对应的是坐标转换。

### 换基

在3维欧拉空间中，换基操作如图所示

![change.png](change.png)

其中
$$ \vec{op} =\vec{oo'}+\vec{o'p} $$

换成坐标表示
$$
\vec{op} = 
  \left[
 \begin{matrix}
   \vec{i},\vec{j},\vec{k}
  \end{matrix}
  \right]
  \left[
 \begin{matrix}
   o'_x \\
   o'_y \\
   o'_z \\
  \end{matrix}
  \right] +
  \left[
 \begin{matrix}
   \vec{i'},\vec{j'},\vec{k'}
  \end{matrix}
  \right]
  \left[
 \begin{matrix}
   p'_x \\
   p'_y \\
   p'_z \\
  \end{matrix}
  \right]
$$

令

$$
  \left[
 \begin{matrix}
   \vec{i'},\vec{j'},\vec{k'}
  \end{matrix}
  \right] =
  \left[
 \begin{matrix}
   \vec{i},\vec{j},\vec{k}
  \end{matrix}
  \right] * M
$$

那么
$$
\vec{op} = 
  \left[
 \begin{matrix}
   \vec{i},\vec{j},\vec{k}
  \end{matrix}
  \right]
  \left(
  \left[
 \begin{matrix}
   o'_x \\
   o'_y \\
   o'_z \\
  \end{matrix}
  \right] +
  M
  \left[
 \begin{matrix}
   p'_x \\
   p'_y \\
   p'_z \\
  \end{matrix}
  \right]
  \right)
$$

所以

$$
\left[
 \begin{matrix}
   x \\
   y \\
   z \\
  \end{matrix}
  \right] =
  \left[
 \begin{matrix}
   o'_x \\
   o'_y \\
   o'_z \\
  \end{matrix}
  \right] +
  M
  \left[
 \begin{matrix}
   p'_x \\
   p'_y \\
   p'_z \\
  \end{matrix}
  \right]
  \rArr
  \left[
 \begin{matrix}
   x' \\
   y' \\
   z' \\
  \end{matrix}
  \right] =
  M *\left(  
  \left[
 \begin{matrix}
   p_x \\
   p_y \\
   p_z \\
  \end{matrix}
  \right] -
  \left[
 \begin{matrix}
   o'_x \\
   o'_y \\
   o'_z \\
  \end{matrix}
  \right]
  \right)
$$

坐标转换分为几个阶段。

1. Model
2. View
3. Projection
4. Viewport

Model 阶段是将模型从局部坐标系(object coordinates)插入到世界坐标系(world coordinates)中。
View 阶段将世界坐标系映射到眼坐标系(eye coordinates)。
Projection将三维坐标投影在z=0的平面(clip coordinates)上。
Viewport将投影后的平面转换到屏幕的坐标系(sceen coordinates)中。

局部坐标系用于模型构建。
世界坐标系用于场景构建。

眼坐标系是从眼中看到的景象。眼坐标使得眼(相机)永远在眼坐标的z轴上，且可以定义一个向量用于指定最终看到镜像的垂直方向。
代码如下：

``` C++
void lookat(Vec3f eye, Vec3f center, Vec3f up) {
    Vec3f z = (eye-center).normalize();
    Vec3f x = cross(up,z).normalize();
    Vec3f y = cross(z,x).normalize();
    Matrix Minv = Matrix::identity();
    Matrix Tr   = Matrix::identity();
    for (int i=0; i<3; i++) {
        Minv[0][i] = x[i];
        Minv[1][i] = y[i];
        Minv[2][i] = z[i];
        Tr[i][3] = -center[i];
    }
    ModelView = Minv*Tr;
}
```

屏幕坐标系是我们实际在屏幕的坐标，以屏幕中心为原点。之前我们在代码中通过下面表达式转换坐标

``` C++
screen_coords[j] = Vec2i((v.x+1.)*width/2., (v.y+1.)*height/2.);
```

写成矩阵形式的代码

``` C++
Matrix viewport(int x, int y, int w, int h) {
    Matrix m = Matrix::identity(4);
    m[0][3] = x+w/2.f;
    m[1][3] = y+h/2.f;
    m[2][3] = depth/2.f;

    m[0][0] = w/2.f;
    m[1][1] = h/2.f;
    m[2][2] = depth/2.f;
    return m;
}
```

对应的矩阵是
$$
 \left[
 \begin{matrix}
   w/2 & 0 & 0 & x+w/2 \\
   0 & h/2 & 0 & y+h/2 \\
   0 & 0 & d/2 & d/2 \\
   0 & 0 & r & 1
  \end{matrix}
  \right]
$$

其中d是深度缓冲的解析度。

通过上面的介绍， 我们将所有的变换串联起来。对于一个模型中的点v， 它的转换过程是

$$Viewport * Projection * View * Model * v$$

### 法向量转换

设法向量n=(A,B,C), 则平面方程为Ax+By+Cz=0.写成齐次坐标的形式得到

$$
\left[
\begin{matrix}
A &
B &
C &
0
\end{matrix}
\right] \times 
\left[
\begin{matrix}
x \\
y \\
z \\
1
\end{matrix}
\right]= 0
$$

在中间插入单位矩阵

$$
\left(
\left[
\begin{matrix}
A &
B &
C &
0
\end{matrix}
\right]
\times
M^{-1}
\right) \times \left(
M \times
\left[
\begin{matrix}
x \\
y \\
z \\
1
\end{matrix}
\right] \right) = 0
$$

重写上面的表达式

$$
\left(
(M^{T})^{-1} \times
\left[
\begin{matrix}
A \\
B \\
C \\
0
\end{matrix}
\right]
\right)^{T} \times \left(
M \times
\left[
\begin{matrix}
x \\
y \\
z \\
1
\end{matrix}
\right] \right) = 0
$$

所以法向量的转换矩阵为$(M^{T})^{-1}$

## 着色器(shader)

整理下之前的渲染的过程,主要分为两步：

1. 顶点着色器(vertex shader), 主要目的是转换顶点的坐标。
2. 片段着色器(fragment shader)，只要是确定像素的颜色。

把这两部分从代码中抽取出来就可以实现各种不同的着色器。

### 纹理

纹理保存的是三维模型表面的各类性质。它基本上可以存放任何东西，包括颜色，法向量，温度等等。

贴图是指将纹理映射到三维模型的表面上。

最常用的贴图可能是漫反射贴图。漫反射贴图中的纹理保存的是物体倍光照射后体现的颜色和光强度。

另外法向量也可以保存在纹理中。在之前的渲染中，法向量是通过三角形的三个顶点算出来的。也就是说，某一个点的法向量的准确性取决于模型的质量（三角形的多少）。

为了计算的效率，我们往往不会渲染一些高质量模型，而是通过高质量模型算出其法向量纹理。在低质量模型上，通过插值的方式，算出每个点的纹理坐标，在纹理中直接获取准确的法向量。

高光贴图是表现物体表面光滑程度的贴图。

在Phong反射模型中，一个点的局部光强由环境光，漫反射光和和高光组成。

![phong.png](phong.png)

环境光是环境中大量光源在物体表面形成的光。它的光强现在假设在各个点上是均匀的，用常熟表示。

漫反射光是光向量和法线向量的内积。它假设光在各个方向是被均匀反射的。

至于光滑表面的高光，它应该遵循光的反射定律。由反射光向量和观察方向的单位向量的内积决定。

反射光可以如下计算

$$r = 2n<n,l> - l$$

其中n是法向量，l是光向量。

在光滑表面上，高光在反射光在反射方向上比其他方向要强得多。这种快速变化可以用指数函数模拟。底是反射光在观察方向的投影。指数反应物体表面的光滑程度，保存在高光纹理中。

下面是一个着色器，它在里面就用了高光贴图 $model->specular(uv)$ ，法向量贴图 $model->normal(uv)$ 和漫反射贴图 $model->diffuse(uv)$

``` C++
struct Shader : public IShader {
    mat<2,3,float> varying_uv;  // same as above
    mat<4,4,float> uniform_M;   //  Projection*ModelView
    mat<4,4,float> uniform_MIT; // (Projection*ModelView).invert_transpose()

    virtual Vec4f vertex(int iface, int nthvert) {
        varying_uv.set_col(nthvert, model->uv(iface, nthvert));
        Vec4f gl_Vertex = embed<4>(model->vert(iface, nthvert)); // read the vertex from .obj file
        return Viewport*Projection*ModelView*gl_Vertex; // transform it to screen coordinates
    }

    virtual bool fragment(Vec3f bar, TGAColor &color) {
        Vec2f uv = varying_uv*bar;
        Vec3f n = proj<3>(uniform_MIT*embed<4>(model->normal(uv))).normalize();
        Vec3f l = proj<3>(uniform_M  *embed<4>(light_dir        )).normalize();
        Vec3f r = (n*(n*l*2.f) - l).normalize();   // reflected light
        float spec = pow(std::max(r.z, 0.0f), model->specular(uv));
        float diff = std::max(0.f, n*l);
        TGAColor c = model->diffuse(uv);
        color = c;
        for (int i=0; i<3; i++) color[i] = std::min<float>(5 + c[i]*(diff + .6*spec), 255);
        return false;
    }
};
```

## 阴影

至此，我们还没有考虑到阴影。其本质是观察物体上的点和光源之间有没有被物体上其他的点遮挡。

关于遮挡的问题让我们想到了z-buffer.只要我们把观察点放在光源的位置。先进行一次渲染，就能得到每个点到光源的深度信息。当然这次渲染只是为了获得光源的深度信息，并不实际作图。

最后我们再用第二个着色器对模型进行渲染。此时在片段着色器中，根据第一次渲染获得的光源深度信息，添加阴影相关代码。

两次的着色器代码如下：

``` C++
struct DepthShader : public IShader {
    mat<3,3,float> varying_tri;

    DepthShader() : varying_tri() {}

    virtual Vec4f vertex(int iface, int nthvert) {
        Vec4f gl_Vertex = embed<4>(model->vert(iface, nthvert)); // read the vertex from .obj file
        gl_Vertex = Viewport*Projection*ModelView*gl_Vertex;          // transform it to screen coordinates
        varying_tri.set_col(nthvert, proj<3>(gl_Vertex/gl_Vertex[3]));
        return gl_Vertex;
    }

    virtual bool fragment(Vec3f bar, TGAColor &color) {
        Vec3f p = varying_tri*bar;
        color = TGAColor(255, 255, 255)*(p.z/depth);
        return false;
    }
};
```

``` C++
struct Shader : public IShader {
    mat<4,4,float> uniform_M;   //  Projection*ModelView
    mat<4,4,float> uniform_MIT; // (Projection*ModelView).invert_transpose()
    mat<4,4,float> uniform_Mshadow; // transform framebuffer screen coordinates to shadowbuffer screen coordinates
    mat<2,3,float> varying_uv;  // triangle uv coordinates, written by the vertex shader, read by the fragment shader
    mat<3,3,float> varying_tri; // triangle coordinates before Viewport transform, written by VS, read by FS

    Shader(Matrix M, Matrix MIT, Matrix MS) : uniform_M(M), uniform_MIT(MIT), uniform_Mshadow(MS), varying_uv(), varying_tri() {}

    virtual Vec4f vertex(int iface, int nthvert) {
        varying_uv.set_col(nthvert, model->uv(iface, nthvert));
        Vec4f gl_Vertex = Viewport*Projection*ModelView*embed<4>(model->vert(iface, nthvert));
        varying_tri.set_col(nthvert, proj<3>(gl_Vertex/gl_Vertex[3]));
        return gl_Vertex;
    }

    virtual bool fragment(Vec3f bar, TGAColor &color) {
        Vec4f sb_p = uniform_Mshadow*embed<4>(varying_tri*bar); // corresponding point in the shadow buffer
        sb_p = sb_p/sb_p[3];
        int idx = int(sb_p[0]) + int(sb_p[1])*width; // index in the shadowbuffer array
        float shadow = .3+.7*(shadowbuffer[idx]<sb_p[2]); 
        Vec2f uv = varying_uv*bar;                 // interpolate uv for the current pixel
        Vec3f n = proj<3>(uniform_MIT*embed<4>(model->normal(uv))).normalize(); // normal
        Vec3f l = proj<3>(uniform_M  *embed<4>(light_dir        )).normalize(); // light vector
        Vec3f r = (n*(n*l*2.f) - l).normalize();   // reflected light
        float spec = pow(std::max(r.z, 0.0f), model->specular(uv));
        float diff = std::max(0.f, n*l);
        TGAColor c = model->diffuse(uv);
        for (int i=0; i<3; i++) color[i] = std::min<float>(20 + c[i]*shadow*(1.2*diff + .6*spec), 255);
        return false;
    }
};
```

## 环境光遮蔽（Ambient occlusion）

在之前的章节中，我们将环境光选取为常数。这是一个很强的假设，它意味着场景中所有的光被平均反射到所有地方。

为了获取更接近于现实的图片，我们可以采用一些更需要计算的方法。

### 暴力方法

我们可以假设我们的物体被一个半圆为主，并在半圆上由上千个顶点发出光。我们每个光源都像计算阴影一样计算它的可见性。最后再叠加起来求平均值并放入纹理中。

那么在实际渲染时环境光就能从纹理中获取。

这个方法是预计算的方法，虽然需要渲染很多次，但是只要纹理渲染出来了，我们就能在实际渲染时直接使用。

### 屏幕空间环境光遮蔽


现在介绍的方法是计算每个像素点的在各个方向上的最大仰角。

首先，我们同样先要渲染一编模型，获取z-buffer。

然后，在第二次渲染时，从像素点上下左右和四个斜方向分别计算仰角。

在特定方向上，每次取附近一个点，通过z-buffer值以及两个点的距离计算仰角大小。多次计算后选取仰角最大的一个。

最后，将个方向的最大仰角求平均值。通过这个值我们就能得到该点环境光的遮蔽程度（凹的程度）。

## 引用

<span id="jump">1. https://github.com/ssloy/tinyrenderer/wiki</span>