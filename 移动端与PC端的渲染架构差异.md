# 移动端与PC端的渲染架构差异

## IMR架构（PC端）

Immediate Mode Rendering(立即模式渲染)是常用于PC端的渲染架构，它的特点是整个管线是连续执行的，执行完上一个任务后，将立即执行下一个任务，无需相互等待。

当接收到一个绘制指令后，这一绘制会立即开始执行，并且将依次顺序经过如下步骤：

1. 顶点处理（Vertex Processing）：从内存读取顶点索引，并根据索引查找相关顶点缓冲区，加载顶点数据。顶点着色器加载到SM中并执行。
2. 裁剪和剔除（Clip & Cull）：在这一过程中，图元引擎将剔除裁剪空间（clip space)外的三角形，并且进行背面剔除操作。
3. 光栅化（Raster）：执行光栅化，从几何转化为像素，像素打包成warp，重新流入SM，并根据重心坐标插值顶点属性。
4. 提前可见性测试（Early Visibility Test ）：对于没有Alpha Test的像素执行early-Z test，通过后进入下一环节。
5. 纹理和着色（Texture & Shade）：执行像素着色器。
6. Alpha测试（Alpha Test）
7. 可见性测试（Late Visibility Test)：对于有Alpha Test的像素，由ZROP执行late-Z test，并根据结果决定是否更新帧缓冲的颜色和深度。
8. Alpha混合（Alpha Blend）：对于通过测试的像素，根据blend计算并更新颜色缓冲区。

## TBR架构 

TBR，即Tile-Based Rendering，是常用于移动端的渲染管线。它的核心思想是牺牲执行效率，优化带宽消耗，以更好地适配移动端硬件。

###  不连续的TBR

对于TBR而言，它的整个过程是不连续的，具体体现在：

1. 提交DrawCall阶段。得到绘制指令后，不会立即开始绘制操作，而是将所有绘制指令缓存起来，到最后才进行绘制；
2. 顶点着色阶段。对所有绘制指令执行vs，并将绘制结果保存起来；
3. 像素着色阶段。绘制所有图元后，再将结果拷贝到frameBuffer对应位置。

概括而言：IMR就是单个指令依次连续执行，TBR则是所有指令完成一个步骤后再进入下一步骤。

###  基于tile的TBR

TBR还有一个重要的特性，那就是它会将frameBuffer划分为多个tile，以tile为单位进行渲染，这也正是它名字的来源。

当每个三角形都执行完vs阶段后，会进入binning pass阶段，此时framebuffer被划为多个tile，并会去计算每个三角形所关联的tile。最终，每个tile记录要渲染的三角形列表。

像素着色阶段，会以tile为单位依次进行绘制。根据primitive list判断当前tile包含哪些三角形以及对应的顶点属性，然后再绘制tile中每个三角形。绘制完成后，拷贝回framebuffer对应位置。

### 为什么说TBR/TDBR优化了带宽消耗

1. **批量读取/写入：** 对于IMR而言，依次连续执行指令，就类似于每次只读取一个数据，这样的操作执行n次；而对于TBR/TDBR而言，将所有指令执行完成后才进入下一阶段，就类似于一次性读取所有数据。这一设计是对带宽友好的。

2. **Tile中的片上内存：** 读写深度缓冲/颜色缓冲是非常消耗带宽的操作，对于IMR架构而言，在做深度测试时和Blending时都会存在读写FrameBuffer的情况，使用TBR后，因为渲染被切分为tile，而tile比较小，因此可以设计一种较快的内存，称为on chip memory。可以先将数据存储在tile上的on chip memory上，提升了读写性能。等所有操作完成后再写入FrameBuffer；
