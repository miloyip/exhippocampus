---
layout: page
title: 以体素建构三维游戏世界
categories:
    - graphics
header:
    image_fullwidth: voxel_header.jpg
    caption: Voxel Pac-Man Shader by Nrx
    caption_url: https://www.shadertoy.com/view/MlfGR4
---

体素除了能应用于像《我的世界（Minecraft）》形式的游戏，还可以有更细致的表现，成为下一代的三维游戏世界构成方式，本文从技术角度分析当中的原理及相关技术。

## 引言

现时，主流三维游戏一般都需要庞大、精致的游戏世界（game
world，或称为游戏关卡／game level、游戏舞台／game
stage）。所谓的游戏世界，不单指玩家可见的三维渲染环境，也包含游戏性系统（游戏机制、物理碰撞、人工智能等）所需的虚拟环境。

多数三维游戏使用三维三角形网格（3D triangle
mesh）建构大部分的游戏世界，包括地形、建筑、植被及其他静态物件。一些以自然户外环境为主的游戏，如大部分RPG、MOBA类型游戏，会使用到高度场（height
field）去表示地形；一些室内为主的游戏，如一些FPS、TPS、ACT等，会使用构造实体几何（constructive
solid geometry, CSG）技术去建构基本的室内环境（在许多游戏引擎中称为 BSP／binary space partitioning 笔刷）。

游戏世界的制作占总制作成本的一大部分，而随著游戏平台的性能提升，以及游戏内容需求的膨胀，游戏世界的制作成本也因应不断提高。以上的制作方法都各有优缺点。三维网格是合乎当代硬件的游戏世界表示方式，也是较自由的建模方式，而且有成熟的数字创作工具（digital
content creation tool）如 3ds
Max 和 Maya。但其缺点包括建模成本高、仅为表面表示方式（可能无法判断一个任意点在其外还是其内）、不容易修改（尤其是在展开UV之后）、不容易做连续或离散级数的细致程度（level
of detail,
LOD）等。高度场和 BSP 的制作成本较网格低，而且较容易修改和实现 LOD，但其适用场合就非常局限。

有没有更好的三维游戏界的制作方式？此问题一直是游戏制作的重要探索方向。

## 体素

自2009年《我的世界（Minecraft）》的空前成功，根据[IGN于2013年9月的报道](http://www.ign.com/articles/2013/09/03/minecraft-pc-sales-at-12-million-franchise-at-33-million)，Minecraft 在所有平台上已售出共三千三百万份。体素（voxel）进入开发者的眼球，成为另一种建构游戏世界的可行方式，见图1、2。

![《我的世界》游戏截屏。](/images/voxel_minecraft.jpg)

图1:《我的世界》游戏截屏。

![《冰与火之歌》中的君临城。](/images/voxel_minecraft-city.jpg)

图2: 《我的世界》游戏玩家合作制作的《冰与火之歌》中的君临城。

简单类比，体素就是像素的三维版本。在二维中，我们可使用颜色的二维数组表示一个影像（image）；在三维中也可以用体素的三维数组表示一个栅格化的三维空间，每个体素储存一个比特，表示该空间是实心还是空心的，如图3。这种二元体素（binary voxel）是最简单的体素形式，但体素还可以储存其他属性。例如《我的世界》的体素会储存让空间的材质（泥土、石、水等），而在医学上会把CT扫描得来的X射线不透光性（opacity）储存在体素中，如图4。

相对于高度场地形及BSP，体素可以同时制作一般地表、山洞、建筑物等固体。

|![体素](/images/voxel.png)|![CT瞄的体素数据](/images/voxel_ctscan.jpg)
|:---:|:---:|
|图3：把一些体素叠起来，灰色立方体是其中之一个体素。（维基百科图片）|图4：以体积渲染方法去可视化CT扫瞄的体素数据。（维基百科图片）|


## 破坏与建设

由于体素数据的结构简单而均匀，它相对于网格来说更容易修改。因此，游戏可以让玩家建设游戏世界（这是一种用户生成内容／user-generated content, UGC），也可以让游戏规则动态改变游戏世界，例如可破坏物件（destructible object）、地形变形（terrain morphing）等。不单止修改，无中生有也是可能的──《我的世界》和《魔方世界（Cube World）》（图5）等游戏都包含程序式生成内容（procedurally generated content），后者更可以生成一个大型的RPG地图及游戏性内容。当然，纯粹自动生成的内容不一定合乎游戏设计师的要求，但某程度的程序式生成功能可以大幅降低制作成本。

![魔方世界](/images/voxel_cubeworld.jpg)

图5：《魔方世界》完全以程序随机生成内容，让玩家可以不断探索无限的世界。整个游戏仅由一对夫妻制作。[官方預告片](https://www.youtube.com/watch?v=fuTT1TFgfoE)

## 更细致的表现

前述的游戏例子都是使用二元体素，其棱角非常突出，虽然可视为一种风格（如8位游戏机时代的象素风格），但是否有方法改善呢？其中一个方法，就如二维影像，我们可以提升体素数据的分辨率，例如从每立方米一个体素提升至每厘米一个体素，以增强细致程度。然而，体积数据是以立方级数增长的，所需的存储容量很高。例如一个 $256^3$ 大小的体素空间含一千六百多万个体素（16M个）；而 $1024^3$ 就会增长至过10亿个体素（1G 个）。即使每个体素只占1字节，也需要大量储存空间。当然，我们可以考虑在一般的应用场合中，大量相连的体素是全部空心或实心的，那么我们可以使用一些数据结构去压缩这些原始体素数据，如八叉树。2010年 NVidia 就曾发表研究[8]，展示如何高效地使用 GPU 去光线追踪以稀疏体素八叉树（sparse voxel octree, SVO）表示的场景。在该研究中，每体素还存储了颜色及法綫的压缩数据。一个大教堂模型（Sibenik）压缩后以 440MB 存储 $4096^3$ 的数据，被渲染时的光綫追踪能力达每秒1亿光綫（图6）。GigaVoxels[3][4]也是相似的技术。[Atomontage 引擎](http://www.atomontage.com/)也是使用含颜色的体素数据，但以独家的方式进行压缩，宣称可以极高帧率实时渲染大型体素场景（图7）。

然而，每体素储存颜色／法綫的方案，未必符合游戏开发所需。此方案的重点，是场景中每个细节（例如达 1cm 的精度）都可以独立编辑。对于在科学应用上通过扫瞄真实场景（例如地形、建筑、生物等）得到的体素数据，这种精确性是必须的。然而，游戏通常只需要虚拟的场景，玩家并不在乎大教堂中的某幅墙的凹凸是否和现实完全相同。从场景制作的角度看，虽然场景能任意雕刻，十分自由，但这种自由度反而难以管理及掌控。情况有如在二维游戏中，可任意绘画的非常巨大的地图。这个问题的争论点类似在 id Software 的 MegaTexture／虚拟纹理（virtual texturing）技术[1][6]所产生的制作及游戏容量问题。以下将描述利用等值面提取及数据扩大解决细致程度及容量问题。

![Sibenik](/images/voxel_sibenik.jpg)

图6: 把 Sibenik 模型体素化为$4096^3$的稀疏体素八叉树（SVO），每个体素含颜色及法线，以GPU光线追踪渲染。

![Atomontage](/images/voxel_atomontage.jpg)

图7: Atomontage 引擎实时渲染大型体素场景，场景可以被动态任意破坏。

# 等值面提取

对于上述谈到的棱角问题，以光綫追踪渲染体素时，可以加入一些插值方式及法綫数据令效果更为平滑。另一个渲染方法，是在体素中储存标量数据，然后提取该标量场的等值面（isosurface），生成三维网格，再以普通的光栅化方式渲染。如果类比二维的情况，就如同提取高度场的等高綫（contour line），再把这些等高綫渲染。在三维的情况，我们可以存储场景的物质密度，或是有符号距离（signed distance)，作为标量场。

给定一个标量场，我们可以使用经典的移动立方体（Marching Cubes, MC）算法[10]去提取等值面。MC 算法十分简单，只需扫瞄体素数据，若等值（isovalue）介乎两个相邻体素的值之间，便使用綫性插值求出中间的等值面顶点，最后使用查表去决定如何把这些顶点连接成等值面三角网格。由于 MC 只能生成圆滑的等值面，不能表示锐利的顶点及棱，后来的对偶轮廓（dual contouring, DC）[7]可以解决此问题，但就需要在体素中加入法綫数据，效果如图9所示。

![Marching Cubes](/images/voxel_marchingcubes.png)

图8: 移动立方体（Marching Cubes, MC）算法中的15个立方体组态。每条棱两端的值与等值比较后，若一个大于等值，一个小于等值，就在该棱上线性插值生成顶点，然后按这些组态生成三角面片。（维基百科图片）

![Dual Contouring](/images/voxel_dc.jpg)
图9: 对偶轮廓（dual contouring, DC）算法可以同时保持锐利的顶点及棱

CryEngine 3 和 C4 Engine 所支持的体素建构场景功能，都使用了这类等值面提取方式。这两个引擎都是以体素方式解决高度场地形的缺点（如无法表示山洞）。下一代的《无尽的任务》包括两个作品《EverQuest Next》及《EverQuest Next Landmark》整合了[Voxel Farm Engine](http://voxelfarm.com/vfweb/index.html)，全面使用了等值面提取方式的体素去建构大型的游戏世界，包括地形、洞穴、建筑物等（圖10），而且还加入破坏／改变游戏环境的技能设计。

![EverQuest Next](/images/voxel_eqnext1.jpg) 
![EverQuest Next](/images/voxel_eqnext2.jpg)

图10: 《EverQuest Next》的场景截屏，地形和建筑物都是用体素建模，生成圆滑或尖锐边缘的网格，再使用紋理映射加強细节及营造游戏独特的美术风格。体素编辑演示视频。

## 数据扩大

使用等值面提取方式后，不需要高分辨率的体素来产生圆滑的表面，可以说一方面解决了棱角的问题。但另一方面，我们如何在不大幅增加数据量的情况下，提升细致程度？其中一个通用技术就是数据扩大（data amplification）──用小量数据生成大量数据。一般高度图地形简单地采用多层重复密铺（tiling）的纹理，也可算是一种简单的数据扩大。这种方法也可以应用到等值面提取上，不过它需要更复杂的纹理映射方式，例如 C4 Engine 可混合三个平面投影去混合纹理采样（tri-planar mapping）[9]。要避免视觉上的重复问题，还可以考虑 Wang Tile [10] 或者其他纹理生成技术，这可能涉及如何参数化（parameterize）等值面的问题。可以参看图11《EverQuest Next》的例子。

![EverQuest Next纹理](/images/voxel_texturing.jpg)

图11：《EverQuest Next》混合两个投影纹理，以映射至任何拓扑结构的生成网格。

除了表面的纹理外，还可以加强几何方面的复杂度，例如我们可以使用位移贴图技術（displacement mapping）去为表面加上更多细节。

## 数字雕刻与其他建模方式

之前谈及许多体素的表示方式，以下再談可如何建立、编辑这些数据，用图形学的述语，就是建模（modeling）。体素非常适合数字雕刻（digital scupting），即是把体素当作粘土，提供贴土、推入、拉出、磨平、捏造等工具去制作模型。Mudbox（图12）和 ZBrush 是现成的数字雕刻工具，但这类工具通常是基于对表面网格几何作出修改，而不能改变模型的拓扑结构。采用体素就不会出现这种拓扑问题，但相对地其编辑工具还未成熟。

![Mudbox](/images/voxel_mudbox.jpg)

图12：使用 Mudbox 在模型网格上进行数字雕刻。现时这类技术通常用于角色建模及纹理，通过法线／位移贴图去增加细节。[基本㓮刻教程](http://www.youtube.com/watch?v=z5X8E8IY9wU)。

体素的一个特点，是可以简单地实现构造实体几何（CSG）。为了更容易建模，可以使用CSG去为建筑等人工物建模，在运行时才体素化。这样比直接雕刻省时，容易修改，而且更能节省容量，并更适合一些自动生成算法。以二维来类比，就像在Photoshop中使用矢量形状编辑，需要时才光删化成像素。另外，对于普通的三维网格，也可以进行体素化，但并不容易保存原来的纹理映射。

## 高级渲染技术

除了等值面提取及纹理贴图等技术外，体素在渲染上还有很多可发展的技术。由于体素适合光线追踪，近年有一些研究使用了体素为基础的实时全局光照，例如体素圆锥追综（voxel cone tracing）[5]。如果游戏世界本身已经是由体素组成，就省却体素化（voxelization）的步骤。而这种全局光照，除了常见的慢反射表面（diffuse surface）的二次反射，还可以实现面积光源、光泽反射／折射（glossy reflection/refraction）材质等效果，如图13。

![Voxel Cone Tracing](/images/voxel_voxelconetracing.jpg)

图13：利用体素圆锥追综制造全局光照。光源从上而下，在地面反射至拱门天花，红布也有渗色（color bleeding）至天花。另外留意地面材质含有光泽反射。（此场景本身是以三角形网格渲染。）[示范视频](https://www.youtube.com/watch?v=fAsg_xNzhcQ)。

## 游戏性技术

当然，游戏并不仅是一个实时渲染器，游戏的重点是游戏性（gameplay）。许多二元体素游戏已经展示出一些与别不同的游戏性元素，在此不作详细分析。但在技术层次上，我们还需要解决一些动态体素世界的问题，主要有物理和人工智能方面。在物理方面，最基本的要考虑等值面与刚体形撞的碰撞检测、支撑分析（切割后悬空的部分可能需要掉落，甚至因结构问题而断裂），此外也可研究相关的流体模拟、燃烧模拟。人工智能方面，最基本是视线查测及路径搜寻。体素也可利于进行地理上的推理分析。

## 结语

传统的游戏世界在游戏性、制作上都有一定限制，基于体素的制作方式可带来各种创新，提高游戏品质并控制制作成本。然而，颠覆传统需要各方面的配合，游戏设计、关卡设计、美术制作、游戏性编程都要注入新的思维，也必须辅以扎实的工具及制作流程。在引擎技术上，需要解决大规模世界的多分辨率建模、依 LOD 作资源串流、实时渲染、物理模拟、人工智能等各个方面的需求。

或许这也是一个机遇，可以开拓另一条游戏技术道路，制作未来具技术壁垒的创新游戏。

## 参考文献
[1] Sean Barrett. [Sparse virtual textures](http://silverspaceship.com/src/svt/). GDC 2008 presentation, 2008.

[2] Michael F Cohen, Jonathan Shade, Stefan Hiller, and Oliver Deussen. [Wang tiles for image and texture generation](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.8.9049&rep=rep1&type=pdf), volume 22. ACM, 2003.

[3] Cyril Crassin, Fabrice Neyret, Sylvain Lefebvre, and Elmar Eisemann. [Gigavoxels: Ray-guided streaming for efficient and detailed voxel rendering](http://hal.inria.fr/docs/00/34/58/99/PDF/CNLE09.pdf). In Proceedings of the 2009 symposium on Interactive 3D graphics and games, pages 15–22. ACM, 2009.

[4] Cyril Crassin, Fabrice Neyret, Miguel Sainz, Elmar Eisemann, et al. Efficient rendering of highly detailed volumetric scenes with gigavoxels. GPU Pro, pages 643–676, 2010.

[5] Cyril Crassin, Fabrice Neyret, Miguel Sainz, Simon Green, and Elmar Eisemann. [Interactive indirect illumination using voxel cone tracing](http://hal.archives-ouvertes.fr/docs/00/65/01/73/PDF/GIVoxels-pg2011-authors.pdf). In Computer Graphics Forum, volume 30, pages 1921–1930. Wiley Online Library, 2011.

[6] Charles Hollemeersch, Bart Pieters, Peter Lambert, and Rik Van de Walle. [Accelerating virtual texturing using cuda](http://aaronm.nuclearglory.com/vt/AcceleratingVirtualTexturingUsingCUDA-b10648-49.pdf). Gpu Pro: Advanced Rendering Techniques, pages 623–641, 2010.

[7] Tao Ju, Frank Losasso, Scott Schaefer, and Joe Warren. [Dual contouring of hermite data](http://faculty.cs.tamu.edu/schaefer/research/dualcontour.pdf). In ACM Transactions on Graphics (TOG), volume 21, pages 339–346. ACM, 2002.

[8] Samuli Laine and Tero Karras. [Efficient sparse voxel octrees](http://www.nvidia.com/docs/IO/88889/laine2010i3d_paper.pdf). Visualization and Computer Graphics, IEEE Transactions on, 17(8):1048–1059, 2011.

[9] Eric Stephen Lengyel. [Voxel-Based Terrain for Real-Time Virtual Simulations](http://www.terathon.com/lengyel/Lengyel-VoxelTerrain.pdf). PhD thesis, UNIVERSITY OF CALIFORNIA, 2010.

[10] William E Lorensen and Harvey E Cline. [Marching cubes: A high resolution 3d surface construction algorithm](http://fab.cba.mit.edu/classes/S62.12/docs/Lorensen_marching_cubes.pdf). In ACM Siggraph Computer Graphics, volume 21, pages 163–169. ACM, 1987.

本文原于2013年11月14日在腾讯内部揭载，获授权公开。
