---
layout:     page
title:      【Direct3D 11】相机变换以及简单的FPS Camera的实现
subtitle:   用矩阵进行相机坐标系变换以及实现简单的第一人称射击Camera类
date:       2018-07-20
author:     翼
header-img: image/bg.jpg
catalog: true
tags:
---
## 前言

>以下内容部分参考《3D游戏编程大师技巧》（后面提到会简称《3D》）。另外，我写的文章都采用 **左手坐标系**。

## View Frustum
3D场景中的模型都位于世界坐标系中，要观察该3D场景（其实是部分场景），通常会将一个虚拟相机放到世界坐标系中的某一个位置，通过这个虚拟相机观察3D的世界。这个虚拟相机其实就好像我们的眼睛，我们可以控制相机的摆放位置以及拍摄方向，来观察3D世界。相机能观察到的3D场景，最终会被渲染到屏幕上。  

那么怎么确定相机能观察到3D世界的哪些物体呢？  
有了相机的观察方向后，再有一个控制观察范围的3D空间就能达到目标。这个3D空间叫做View Frustum，观察平截头体，简单来说称之为视景体。  

![view frustum](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/direct3d/camera/view_frustum.png)  

如上图。  
一台虚拟相机摆放在(*cam_x*,*cam_y*,*cam_z*)处，相机方向(*ang_x*, *ang_y*, *ang_z*)。还有视景体，定义了相机的观察范围。  
视景体由以下四个元素定义：  
1. 水平方向的视野 *FOV_x*
1. 竖直方向的视野 *FOV_y* （**FOV一般在60-130度，实际计算要以弧度为单位**）
1. 近裁剪面 *Z_near*
1. 远裁剪面 *Z_far*  

以上四个量，可以组成一个六面体，好像一个截掉顶端的金字塔，即视景体，包围了虚拟相机可见的范围。不在视景体范围内的，都会被裁剪掉。   
**注意：** 裁剪平面Z_near和Z_far的坐标已经是在相机坐标系了，此时，已经把相机当做原点，相机观察的方向为 **+Z** 。  
有了相机的观察方向和视景体，已经能知道我们能观察到的空间范围了。但是为了实际计算，必须要定义 **相机坐标系**。  
D3D采用左手坐标系：向右为 **+X** 轴， 向上为 **+Y** 轴，垂直屏幕（或纸面）向里为 **+Z** 轴。  

### 世界坐标到相机坐标的转换
为了简化计算相机的可见范围，即视景体的范围，我们要让3D场景的物体位于相机坐标系中。  
具体如下：  
1. 相机位于原点
1. 相机的观察方向是 **+Z** 方向，与世界坐标系 **+Z** 轴重合
1. 相机的右侧为 **+X** 方向，与世界坐标系 **+X** 轴重合
1. 相机的上方为 **+Y** 方向，与世界坐标系 **+Y** 轴重合

实际情况中，大部分情况下的相机是不处于原点的，相机的朝向也不是 **+Z**方向。这就需要我们做一些坐标变换。  
对于一般情况，要进行两种变换：平移和旋转。  
1. 平移
平移是为了让相机处于原点。对于相机位置(*cam_x*,*cam_y*,*cam_z*)，想要把相机平移到原点，只需要一个平移矩阵：  
$$ T_c =
\begin{pmatrix}
	1 & 0 & 0 & 0 \\
	0 & 1 & 0 & 0 \\
	0 & 0 & 1 & 0 \\
  -cam_x & cam_y & -cam_z & 1 \\
\end{pmatrix}
$$
![Tc](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/direct3d/camera/Tc.png)
将场景内的物体的顶点坐标都乘以这个矩阵，即可计算出物体在相机坐标系的坐标。  

1. 旋转
旋转的目的是让相机的三个轴与世界坐标系的 **+Z** 轴重合。  
计算旋转矩阵可以分三步: 饶Y、X、Z三个轴旋转。但是实际上有更直接的方法算这个旋转矩阵。  
相机位置(*cam_x*,*cam_y*,*cam_z*)，假设其三个表示X、Y、Z轴朝向的坐标轴分别为：  
**R=** (*R_x*, *R_y*, *R_z*)  
**U=** (*U_x*, *U_y*, *U_z*)  
**V=** (*V_x*, *V_y*, *V_z*)  
我们的目的其实是把这个三个向量转换为：(1, 0, 0 )、(0, 1, 0)、( 0, 0, 1 )。  
那么其实我们是要求一个矩阵R_c，能满足以下运算:  

$$
\begin{pmatrix}
	R_x & R_y & R_z \\
	U_x & U_y & U_z \\
	V_x & V_y & V_z \\
\end{pmatrix}
*
R_c =
\begin{pmatrix}
	1 & 0 & 0 \\
	0 & 1 & 0 \\
	0 & 0 & 1 \\
\end{pmatrix}
$$
![get rc](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/direct3d/camera/get_Rc.png)

3. 上面的关键在于：右侧是一个单位矩阵。所以我们要求的就是上面左侧矩阵的逆矩阵。还有一个关键是，左侧的矩阵是一个正交矩阵，而正交矩阵的逆矩阵就是它的转置矩阵。因此：  

$$
R_c =
\begin{pmatrix}
	R_x & U_x & V_x\\
	R_y & U_y & V_y \\
	R_z & U_z & V_z \\
\end{pmatrix}
$$
![rc](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/direct3d/camera/Rc.png)

那么最终我们要求的世界坐标到相机坐标的转换矩阵应该是： **M** = **T_c** * **R_c**。  

$$
M =
\begin{pmatrix}
	1 & 0 & 0 & 0 \\
	0 & 1 & 0 & 0 \\
	0 & 0 & 1 & 0 \\
  -cam_x & cam_y & -cam_z & 1 \\
\end{pmatrix}
*
\begin{pmatrix}
	R_x & U_x & V_x & 0\\
	R_y & U_y & V_y & 0 \\
	R_z & U_z & V_z & 0 \\
  0 & 0 & 0 & 1 \\
\end{pmatrix}
$$
$$
M =
\begin{pmatrix}
	R_x & U_x & V_x & 0\\
	R_y & U_y & V_y & 0 \\
	R_z & U_z & V_z & 0 \\
  -C*R & -C*U & -C*V & 1 \\
\end{pmatrix}
$$
![M](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/direct3d/camera/M.png)

上面的矩阵中：C是相机的位置向量 **(** **cam_x**,**cam_y**,**cam_z** **)**  
R、U、V分别是相机的三个轴的方向向量。可见最终的转换矩阵 **M** 跟它们的标量积有关。  

*感谢网友分享的这个方法，关于这个计算旋转矩阵的方法原文在https://blog.csdn.net/bonchoix/article/details/8521411。*

## 简单FPS Camera类的实现
我给FPSCamera类定义了如下方法：
```cpp
//set and get the position of the camera

void SetPosition(const float x, const float y, const float z);
DirectX::XMFLOAT3 GetPosition();

//set and get the axes of the camera coordinates system

void SetRight(DirectX::XMFLOAT3 right); //x-axis

DirectX::XMFLOAT3 GetRight();

void SetUp(DirectX::XMFLOAT3 up); //y

DirectX::XMFLOAT3 GetUp();

void SetLook(DirectX::XMFLOAT3 look); //z

DirectX::XMFLOAT3 GetLook();

//by Walk(), the camera can go forward, backward, up, down, left and right

void Walk(const direction::DIRECTION direction, const float dist);

//these methods control the yaw, pitch and roll of the camera

void Yaw(const float angle);
void Pitch(const float angle);
void Roll(const float angle);

//compute the new view matrix with new position, right, up, and look vector

void UpdateView();

//return the view matrix

DirectX::XMMATRIX GetViewMatrix() const { return XMLoadFloat4x4(&view_matrix_); }

//these are the member vars

DirectX::XMFLOAT3 position_;
DirectX::XMFLOAT3 right_;
DirectX::XMFLOAT3 up_;
DirectX::XMFLOAT3 look_;
DirectX::XMFLOAT4X4	view_matrix_;
```
函数实现如下：
```cpp
void FPSCamera::SetPosition(const float x, const float y, const float z) {
    position_ = XMFLOAT3(x, y, z);
}

XMFLOAT3 FPSCamera::GetPosition() {
    return position_;
}

void FPSCamera::SetRight(XMFLOAT3 right) {
    right_ = right;
}

DirectX::XMFLOAT3 FPSCamera::GetRight() {
    return right_;
}

void FPSCamera::SetUp(XMFLOAT3 up) {
    up_ = up;
}

DirectX::XMFLOAT3 FPSCamera::GetUp() {
    return up_;
}

void FPSCamera::SetLook(XMFLOAT3 look) {
    look_ = look;
}

DirectX::XMFLOAT3 FPSCamera::GetLook() {
    return look_;
}

void FPSCamera::Walk(const direction::DIRECTION direction, const float dist) {
    switch (direction) {
    case direction::FORWARD: {
        position_.z += dist;
    }
    break;
    case direction::BACKWARD: {
        position_.z -= dist;
    }
    break;
    case direction::LEFT: {
        position_.x -= dist;
    }
    break;
    case direction::RIGHT: {
        position_.x += dist;
    }
    break;
    case direction::UP: {
        position_.y += dist;
    }
    break;
    case direction::DOWN: {
        position_.y -= dist;
    }
    break;
    default:
        break;
    }
}

void FPSCamera::Yaw(const float angle) {
    XMMATRIX rotation = XMMatrixRotationY(angle);

    //DirectXMath uses a completely wrong and misleading name.
    
    //XMVector3TransformNormal transforms 3D directions (w-coordinate is implicitly set to 0).
    
    XMStoreFloat3(&right_, XMVector3TransformNormal(XMLoadFloat3(&right_), rotation));
    XMStoreFloat3(&up_, XMVector3TransformNormal(XMLoadFloat3(&up_), rotation));
    XMStoreFloat3(&look_, XMVector3TransformNormal(XMLoadFloat3(&look_), rotation));
}


void FPSCamera::Pitch(const float angle) {
    //XMMatrixRotationX
    
    XMMATRIX rotation = XMMatrixRotationAxis(XMVectorSet(1.0f, 0.0f, 0.0f, 0.0f), angle);
    XMStoreFloat3(&up_, XMVector3TransformNormal(XMLoadFloat3(&up_), rotation));
    XMStoreFloat3(&look_, XMVector3TransformNormal(XMLoadFloat3(&look_), rotation));
}

void FPSCamera::Roll(const float angle) {
    XMMATRIX rotation = XMMatrixRotationZ(angle);
    XMStoreFloat3(&up_, XMVector3TransformNormal(XMLoadFloat3(&up_), rotation));
    XMStoreFloat3(&look_, XMVector3TransformNormal(XMLoadFloat3(&look_), rotation));
}

//关键在于UpdateView，这里根据上面求出的转换矩阵M来计算view matrix

void FPSCamera::UpdateView() {
    XMVECTOR right = XMLoadFloat3(&right_);
    XMVECTOR up = XMLoadFloat3(&up_);
    XMVECTOR look = XMLoadFloat3(&look_);
    XMVECTOR position = XMLoadFloat3(&position_);

    //对三个向量进行正交化以及归一化:因为计算过程会有些误差
    
    right = XMVector3Normalize(XMVector3Cross(up, look));
    up = XMVector3Normalize(XMVector3Cross(look, right));
    look = XMVector3Normalize(look);

    float p_dot_x = -XMVectorGetX(XMVector3Dot(position, right));
    float p_dot_y = -XMVectorGetX(XMVector3Dot(position, up));
    float p_dot_z = -XMVectorGetX(XMVector3Dot(position, look));

    XMStoreFloat3(&right_, right);
    XMStoreFloat3(&up_, up);
    XMStoreFloat3(&look_, look);
    XMStoreFloat3(&position_, position);

    view_matrix_(0, 0) = right_.x;	view_matrix_(0, 1) = up_.x;	view_matrix_(0, 2) = look_.x;	view_matrix_(0, 3) = 0;
    view_matrix_(1, 0) = right_.y;	view_matrix_(1, 1) = up_.y;	view_matrix_(1, 2) = look_.y;	view_matrix_(1, 3) = 0;
    view_matrix_(2, 0) = right_.z;	view_matrix_(2, 1) = up_.z;	view_matrix_(2, 2) = look_.z;	view_matrix_(2, 3) = 0;
    view_matrix_(3, 0) = p_dot_x;
    view_matrix_(3, 1) = p_dot_y;
    view_matrix_(3, 2) = p_dot_z;
    view_matrix_(3, 3) = 1;
}
```
