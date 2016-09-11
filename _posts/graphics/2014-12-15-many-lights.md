---
layout: page
title: SIGGRAPH Asia 2014 见闻之《高效地使用大量光源于实时着色》课程
categories:
    - graphics
header:
    image_fullwidth: manylights_header.jpg
    caption: Photograph by Ryan Pouncy
    caption_url: https://unsplash.com/pixelperfect
---

在 SIGGRAPH Asia 2014 中，《[高效地使用大量光源于实时着色（Efficient Real-TimeShading with Many Lights）](https://newq.net/publications/more/sa2014-many-lights-course)》是和游戏技术最接近的课程之一。三位主讲分别是 Ola Olsson（瑞典查尔姆斯理工大学博士）、Emil Persson（Avalanche Studios研究部负责人）、Markus Billeter（瑞典查尔姆斯理工大学博士候选人）。课程内容比较前向、延迟、分块、群组和相关技术实现大量动态光源的渲染管道。由于现时未获演讲稿的电子版本，笔者尽量以记忆及相关文献简单介绍一下本课程的重点。

近年，实时渲染管道经历了一连串变革，全动态光源对这些变革有重要影响。传统的前向渲染（forward rendering）管道在渲染$n$个物体在$m$个光源下着色，需要绘制$\mathrm{O}(nm)$次。当需要越来越复杂的游戏场景，更多光源，这种方式成为CPU及GPU的性能瓶颈。在2004年 GDC 出现了延迟渲染（deferred rendering）管道的讨论[4]，所有物体都先绘制在一组屏幕空间的缓冲（称为几何缓冲区／G-buffer，见图1），再逐光源对该缓冲着色，复杂度变成$\mathrm{O}(n + m)$。另一种相关的技术在同一届 GDC 中发表，被称为延迟光照（deferred lighing）[3]，几年后也被称为前期光照通道（light pre-pass）[2]。通过这类“延迟”方式，已可以大量增加物体和光源的数量，渲染出更复杂的场景。然而，这种延迟方式有几个问题。首先，只能使用统一的材质着色器，而且其参数受限（每个参数需要相应的G-buffer），这对于游戏的画面风格造成很大的限制。第二，在性能上虽然复杂度降低了，但读写G-buffer的内存带宽用量成为了性能瓶颈。由于GPU的计算性能不断提升但显存带宽却提升缓慢，使延迟渲染越来越不适合近代的GPU。第三，不能处理半透明及多重采样抗锯齿（MultiSampling Anti-Aliasing, MSAA）。

| ![漫反射缓冲区](/images/manylights_deferred_rendering_pass_col.jpg) | ![深度缓冲区](/images/manylights_deferred_rendering_pass_dep.jpg) | ![法线缓冲区](/images/manylights_deferred_rendering_pass_nor.jpg) | ![最终结果](/images/manylights_deferred_rendering_pass_res.jpg) |
|:---:|:---:|:---:|:---:|
| (a) 漫反射缓冲区 | (b) 深度缓冲区 | (c) 法线缓冲区 | (d) 最终结果 |

图1：延迟渲染的几何缓冲区／G-buffer及渲染结果（维基百科图片）

因为这些因素，近几年催生出另一族渲染管道的方式，称为分块渲染（tiled rendering）[5]。分块渲染有多个变种，先说分块延迟渲染（tiled deferred rendering）。在前面提及，延迟渲染的瓶颈在于读写 G-buffer，在大量光源下，具体瓶颈将位于每个光源对 G-buffer的读取及与颜色缓冲区（color buffer）混合。这里的问题是，每个光源，即使它们的影响范围在屏幕空间上有重疉，因为每个光源是在不同的绘制中进行，所以会重复读取G-buffer中相同位置的数据，计算后以相加混合方式写入颜色缓冲。光源越多，内存带宽用量越大。而分块延迟渲染则是把屏幕分拆成细小的栅格，例如每 32 × 32 象素作为一个分块（tile）。然后，计算每个分块会受到哪些光源影响，把那些光源的索引储存在分块的光源列表里。最后，逐个分块进行着色，对每像素读取 G-buffer 和光源列表及相关的光源信息。因此，G-buffer的数据只会被读取1次且仅1次，写入 color buffer 也是1次且仅1次，大幅降低内存带宽用量。不过，这种方法需要计算光源会影响哪些分块，这个计算又称为光源剔除（light culling），可以在 CPU 或 GPU（通常以 compute shader 实现）中进行。用GPU计算的好处是，GPU 计算这类工作比 CPU 更快，也减少 CPU／GPU 数据传输。而且，可以计算每个分块的深度范围（depth range），作更有效的剔除。

| ![渲染结果](/images/manylights_tiled_shading_shot.jpg) | ![光源分块](/images/manylights_tiled_shading_grid.jpg) |
|:---:|:---:|
| (a) 渲染结果 | (b) 光源分块 |

图2：分块渲染（图片来自 Olaolss 的[网页](http://www.cse.chalmers.se/~olaolss/main_frame.php?contents=publication&id=tiled_shading)）

有趣的是，这种分块方式也可以用于前向渲染，这称为分块前向渲染（tiled forward rendering）[1]。当计算好光源列表，前向渲染时就按屏幕位置读取相关的光源信息去着色。那么，就可以同时解决延迟渲染的各种问题，又不需要像传统前向渲染对物体进行多次绘制。这种分块渲染似乎已经很理想，但如果增加更多光源，而场景又比较空扩，每个分块的光源数量就会变得更多。虽然做了深度范围的优化可缓解这个问题，但如果在视野里有很多深度不连续区域（depth discontinuities），如图3，那么该优化也无能为力了。为了解决这个问题，研究者想出了不同的新方法。

|![渲染结果](/images/manylights_depthdiscon1.jpg) | ![深度不连续的区域](/images/manylights_depthdiscon2.jpg) |
|:---:|:---:|
| (a) 渲染结果 | (b) 深度不连续的区域 |

图3：深度不连续的场景（图片来自[8]）

Olsson、Billeter 及他们的导师 Ulf Assarsson 在2012年发表了一种新的改善方式，名为分群组渲染（clustered rendering）[6][7]。这里的群组（cluster）是指分块（tile）在深度上进一步划分，这种体积数据使每个单位储存更少量的光源（图4）。在渲染时，除了考虑着色像素的屏幕坐标，还考虑到深度的坐标，去索引群组并取得光源信息。这个简单的概念优化了光源剔除，使光源数目能进一步提升。

| ![屏幕空间上仍是栅格式划分](/images/manylights_clusters0.jpg) | ![但从侧面可看到深度被划分成多个群组](/images/manylights_clusters1.jpg) |
|:---:|:---:|
| (a) 屏幕空间上仍是栅格式划分 | (b) 但从侧面可看到深度被划分成多个群组 |

图4：群组渲染（图片来自Olaolss的网页 3）

据游戏业界的Persson称，以他所知，暂时未有已发行游戏使用了群组渲染技术。而他在Avalanche Studios开发中的《正当防卫3（Just Cause 3）》则使用了此技术[8]。对于实际在游戏中应用群组渲染技术，他表示非常乐观。在开发本作中，他从实际的场景中发现，如果简单地使用指数方式划分深度，较近的范围可能有太多群组，而使效率变差。所以他把最近的群组加深，有效改善此问题。另外，由于该作主要是室外大型场景，远景就不使用真正的动态光照，而仅使用公告板作渲染。

| ![缺省的指数深度分布](/images/manylights_justcluster1.png) | ![特殊的近景深度分布](/images/manylights_justcluster2.png) |
|:---:|:---:|
| (a) 缺省的指数深度分布 | (b) 特殊的近景深度分布 |

图5：Avalanche 工作室对群组深度分布的优化（图片来自[8]）

不过，在光照上还有一个未有完善解决的问题，就是阴影。笔者记得，以前在解决虚拟点光源模拟全局光照的阴影问题时，曾出现一种名为不完美阴影贴图（imperfect shadow map, ISM）的技术[9]。刚好在这课程的前一天 Square Enix 的德吉雄介也谈到使用ISM于他们的全局渲染管道中[10]。然而，本课程中 Olsson 就介绍了他和 Billeter 及其他同事合作发明的一个方案，
称为虚拟阴影贴图（virtual shadow map），采用类似 id Tech 5 的虚拟纹理（virtual texturing）技术，去动态按阴影采样需求动态分配阴影贴图的纹理空间。其结果也是不错的，虽然使用了较大量的显存，但似乎在这一代游戏机平台上是有可能应用到的。

最后 Billeter 讲述在移动平台上实现多光源的尝试，由于移动设备的内存带宽比PC的问题更大，使用分块或群组渲染都更有竞争力。可望在近一两年内在高端手机游戏中实际应用。

## 参考文献

[1] Markus Billeter, Ola Olsson, and Ulf Assarsson. Tiled forward shading. GPU Pro 4: Advanced Rendering Techniques, 4:99, 2013.

[2] Wolfgang Engel. Designing a renderer for multiple lights–the light pre-pass renderer. ShaderX7: Advanced rendering techniques, 2009.

[3] Rich Geldreich, Matt Pritchard, and John Brooks. [Deferred lighting and shading](http://www.tenacioussoftware.com/gdc_2004_deferred_shading.ppt). In Game Developers Conference, volume2, 2004. 

[4] Shawn Hargreaves and Mark Harris. [Deferred shading](http://www.shawnhargreaves.com/DeferredShading.pdf). In Game Developers Conference, volume 2, 2004. 

[5] Ola Olsson and Ulf Assarsson. [Tiled shading](http://www.cse.chalmers.se/~olaolss/get_file.php?filename=papers/Improved%20Ray%20Hierarchy%20Alias%20Free%20Shadows.pdf). Journal of Graphics, GPU, and Game Tools, 15(4):235–251, 2011. 

[6] Ola Olsson, Markus Billeter, and Ulf Assarsson. [Clustered deferred and forward shading](http://www.cse.chalmers.se/~olaolss/get_file.php?filename=papers/clustered_shading_preprint.pdf). In HPG ’12: Proceedings of the Conference on High Performance Graphics 2012, 2012.

[7] Ola Olsson, Markus Billeter, and Ulf Assarsson. [Tiled and clustered forward shading](http://www.cse.chalmers.se/~olaolss/get_file.php?filename=papers/tiled_shading_siggraph_2012.pdf). In SIGGRAPH ’12: ACM SIGGRAPH 2012 Talks, New York, NY, US-
A, 2012. ACM.

[8] Emil Persson and Ola Olsson. [Practical clustered deferred and forward shading](http://www.cse.chalmers.se/~olaolss/get_file.php?filename=papers/siggraph_2013.pdf). SIGGRAPH Course: Advances in Real-Time Rendering in Games, 2013. 

[9] Tobias Ritschel, Thorsten Grosch, Min H Kim, H-P Seidel, Carsten Dachsbacher, and Jan Kautz. [Imperfect shadow maps for efficient computation of indirect illumination](http://people.mpi-inf.mpg.de/~ritschel/Papers/ISM.pdf). In ACM Transactions on Graphics (TOG), volume 27, page 129. ACM, 2008. 

[10] Yusuke Tokuyoshi. [Virtual spherical gaussian lights for real-time glossy indirect illumination](http://www.jp.square-enix.com/info/library/pdf/Virtual%20Spherical%20Gaussian%20Lights%20for%20Real-Time%20Glossy%20Indirect%20Illumination.pdf). In SIGGRAPH Asia 2014 Technical Briefs, page 17. ACM, 2014. 

本文原于2014年12月15日在腾讯内部揭载，获授权公开。
