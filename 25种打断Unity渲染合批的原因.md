# 25种打断 Unity 渲染合批的原因

1. **Additional Vertex Streams** — 对象使用MeshRenderer.additionalVertexStreams设定了额外的顶点信息流。
2. **Deferred Objects on Different Lighting Layers** — 该物件位于另一不同的光照层中。
3. **Deferred Objects Split by Shadow Distance** — 两个物体中有一个在阴影距离范围内而另一个不是。
4. **Different Combined Meshes** — 该对象属于另一个已合并的静态网格。
5. **Different Custom Properties** — 该对象设定了不同的MaterialProperyBlock。
6. **Different Lights** — 该物件受不同的前向光照（Forward Light）影响
7. **Different Materials** — 该对象使用不同的材质。
8. **Different Reflection Probes** — 该对象受不同的反射探头（Reflection Probe）影响。
9. **Different Shadow Caster** **Hash** — 该对象使用其他的阴影投射着色器，或是设定了不同的着色器参数/关键词，而这些参数/关键词会影响阴影投射Pass的输出。
10. **Different Shadow Receiving Settings** — 该对象设定了不同的“Receive Shadows”参数，或是一些对象在阴影距离内，而另一些在距离之外。
11. **Different Static Batching Flags** — 该对象使用不同的静态批处理设定。
12. **Dynamic Batching Disabled to Avoid Z-Fighting** — Player Settings中关闭了动态批处理，或在当前环境中为避免深度冲突而被临时关闭。
13. **Instancing Different Geometries** — 使用GPU Instancing渲染不同的网格或子网格。
14. **Lightmapped Objects** — 对象使用了不同的光照贴图，或在相同的光照贴图中有不同的光照贴图UV转换关系。
15. **Lightprobe Affected Objects** — 对象受其他光照探头（Light Probe）影响。
16. **Mixed Sided Mode Shadow Casters** — 对象的“Cast Shadows”设定不同。
17. **Multipass** — 对象使用了带多个Pass的着色器。
18. **Multiple Forward Lights** — 该物件受多个前向光渲染影响。
19. **Non-instanceable Property Set** — 为instanced着色器设定来non-instanced属性。
20. **Odd Negative Scaling** — 该对象的缩放为很奇怪的负值，例如(1,-1,1)。
21. **Shader Disables Batching** — 着色器使用“DisableBatching”标签显式关闭了批处理。
22. **Too Many Indices in Dynamic Batch** — 动态批处理索引过多（超过32k）。
23. **Too Many Indices in Static Batch** — 静态批处理中的组合网格索引过多。对于OpenGL ES来说是48k，OSX是32k，其他平台是64k。
24. **Too Many Vertex Attributes for Dynamic Batching** — 欲进行动态批处理的子网格拥有超过900个顶点属性。
25. **Too Many Vertices for Dynamic Batching** — 欲进行动态批处理的子网格顶点数量超过300个。