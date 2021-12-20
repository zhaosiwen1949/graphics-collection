# 四元数

参考资料：
1. [eater.net/quaternions](https://eater.net/quaternions/)
2. [四元数的可视化](https://www.bilibili.com/video/BV1SW411y7W1)
## 借助四元数乘法的三维空间旋转

三维空间的点，借助四元数乘法，可以快速进行旋转变换，下面会介绍变换的流程：
1. 假设三维空间中有一点 $P(a, b, c)$，通过 Stereographic Projection，必然对应着四维空间中单位球上的一个四元数 $p(\cos\theta, \sin\theta(x\pmb i+y\pmb j+z\pmb k))$，注意：$a、b、c$ 和 $x、y、z$ 之前并无相等关系；
2. 四元数 $p$ 分别进行左乘和右乘的变换 $p^\prime=qpq^{-1}$，在四维空间中意味着对于 $p$ 点分别进行 $q$ 和 $q^{-1}$ 的四维旋转变换，因为 $p$ 点在四维单位球上，所以进行四维旋转后，仍然在四维单位球上，所以旋转后的点 $p^\prime(\cos\theta^\prime, \sin\theta^\prime(x^\prime\pmb i+y^\prime\pmb j+z^\prime\pmb k))$，仍然可以通过 Stereographic Projection 映射到三维空间上的一个点 $P^\prime(a^\prime, b^\prime, c^\prime)$；
3. 当四元数 $p$ 在四维空间上进行左乘和右乘变换 $p^\prime=qpq^{-1}$ 的时候（其中，$q=(\cos\frac{\omega}{2}, \sin\frac{\omega}{2}(d\pmb i+e\pmb j+f\pmb k))$、$q^{-1}=(\cos\frac{-\omega}{2}, \sin\frac{-\omega}{2}(d\pmb i+e\pmb j+f\pmb k))$），四维单位球映射到三维空间上的点 $P$，会沿着三维空间上轴 $\vec{A}(d, e, f)$，按照右手定则顺时针旋转 $\omega$ 角度，最终旋转到点 $P^\prime$；
4. 综上所述，借助四维单位球上的变换，可以描述三维空间中点的旋转，整体思路就是：首先根据三维空间中的一点 $P(a, b, c)$，通过 Stereographic Projection 找到对应的四维单位球上的四元数 $p(\cos\theta, \sin\theta(x\pmb i+y\pmb j+z\pmb k))$，然后通过计算 $p^\prime=qpq^{-1}$，得到新的四元数 $p^\prime(\cos\theta^\prime, \sin\theta^\prime(x^\prime\pmb i+y^\prime\pmb j+z^\prime\pmb k))$，再通过 Stereographic Projection，映射回三维空间中的 $P^\prime(a^\prime, b^\prime, c^\prime)$ 点，此时点 $P^\prime$ 就是点 $P$ 旋转后的结果；

## 简化后的过程
不难看出，上述过程中需要将三维空间和四维单位球上的点，来回通过 Stereographic Projection，进行映射运算，计算起来会相当麻烦，有没有办法能简化这种运算？
这里我们要借助于一个特殊的球面，也就是三维空间中的单位球面，这个球面上任意一点 $H(u, v, w)$，通过 Stereographic Projection，对应于四维单位球上的点 $h(\cos{90^\degree}, \sin{90^\degree}(u\pmb i+v\pmb j+w\pmb k))$，也就是 $h(0, u\pmb i, v\pmb j, w\pmb k)$，下面利用这个规律，简化上面的计算：
1. 找到与点 $P(a, b, c)$ 在同一方向上的三维单位球面上的点 $H(u, v, w)$，其中
$$
\|P\|=\sqrt{a^2+b^2+c^2},
u=\frac{a}{\|P\|},
v=\frac{b}{\|P\|},
w=\frac{c}{\|P\|},
H(u, v, w)=\frac{P(a, b, c)}{\|P\|}
$$
2. 点 $H(u, v, w)$ 对应的四元数为 $h(0, u\pmb i, v\pmb j, w\pmb k)$，进行四元数左右乘运算 $h^\prime=qhq^{-1}$，所得 $h^\prime=(0, u^\prime\pmb i, v^\prime\pmb j, w^\prime\pmb k)$；
3. $h^\prime$ 对应的三维空间上的点为 $H^\prime(u^\prime, v^\prime, w^\prime)$；
4. ，因为在三维空间中，只是进行旋转，并没与进行伸缩，所以点 $P(a, b, c)$ 旋转后的点 $P^\prime(a^\prime, b^\prime, c^\prime)=\|P\|\times{H^\prime(u^\prime, v^\prime, w^\prime)}$，所以
$$
a^\prime=\|P\|\times{u^\prime},
b^\prime=\|P\|\times{v^\prime},
c^\prime=\|P\|\times{w^\prime}
$$
5. 综上所述，根据点 $P(a, b, c)$ 构造四元数 $p(0, a\pmb i, b\pmb j, c\pmb k)=\|P\|\times{h(0, u\pmb i, v\pmb j, w\pmb k)}$，由于经过四元数变换后（其中，$q=(\cos\frac{\omega}{2}, \sin\frac{\omega}{2}(d\pmb i+e\pmb j+f\pmb k))$、$q^{-1}=(\cos\frac{-\omega}{2}, \sin\frac{-\omega}{2}(d\pmb i+e\pmb j+f\pmb k))$，四维单位球映射到三维空间上的点 $P$，会沿着三维空间上轴 $\vec{A}(d, e, f)$，按照右手定则顺时针旋转 $\omega$ 角度，最终旋转到点 $P^\prime$）
    $$  
    \begin{aligned}
    h^\prime&=qhq^{-1}\\
    &=(0, u^\prime\pmb i, v^\prime\pmb j, w^\prime\pmb k)
    \end{aligned}
    $$  
    所以
    $$
    \begin{aligned}
    \|P\|\times{h^\prime}&=\|P\|\times{qhq^{-1}}\\
    &=q(\|P\|\times{h})q^{-1}\\
    &=qpq^{-1}
    \end{aligned}
    $$
    $$
    \begin{aligned}
    \|P\|\times{h^\prime}&=\|P\|\times{(0, u^\prime\pmb i, v^\prime\pmb j, w^\prime\pmb k)}\\
    &=(0, a^\prime\pmb i, b^\prime\pmb j, c^\prime\pmb k)\\
    &=p^\prime
    \end{aligned}
    $$
    因此
    $$
    \begin{aligned}
    p^\prime=qpq^{-1}
    \end{aligned}
    $$
    通过构造四元数 $p(0, a\pmb i, b\pmb j, c\pmb k)$，直接通过 $p^\prime=qpq^{-1}$ 计算出新的四元数 $p^\prime=(0, a^\prime\pmb i, b^\prime\pmb j, c^\prime\pmb k)$，可以直接截取其中虚部，得到旋转后的点 $P^\prime(a^\prime, b^\prime, c^\prime)$。
