## Unity Profiler 数据解析工具可用类 

###  一、核心数据视图类（Data View）

1. FrameDataView (Modules/ProfilerEditor/Public/FrameDataView.bindings.cs)

  所有帧数据视图的基类，提供：
  - 帧基本信息：frameIndex, frameTimeMs, frameFps, sampleCount, maxDepth
  - 线程信息：threadIndex, threadName, threadId
  - Marker 查询：GetMarkerId(), GetMarkerName(), GetMarkerFlags(), GetMarkers()
  - 类别信息：GetCategoryInfo(), GetAllCategories(), GetMarkerCategoryColor()
  - 元数据：GetFrameMetaData<T>(), GetSessionMetaData<T>()
  - 对象信息：GetUnityObjectInfo(), GetGfxResourceInfo()

2. RawFrameDataView (Modules/ProfilerEditor/Public/RawFrameDataView.bindings.cs)

  原始帧数据视图，继承自 FrameDataView，提供：
  - 样本详细信息：GetSampleStartTimeMs(), GetSampleTimeMs(), GetSampleTimeNs()
  - 样本标识：GetSampleMarkerId(), GetSampleFlags(), GetSampleCategoryIndex(), GetSampleName()
  - 层级关系：GetSampleChildrenCount(), GetSampleChildrenCountRecursive()
  - 元数据读取：GetSampleMetadataAsInt/Long/Float/Double/String<T>(), GetSampleMetadataAsSpan<T>()
  - 调用栈：GetSampleCallstack()
  - 流事件：GetSampleFlowEvents(), GetFlowEvents()

3. HierarchyFrameDataView (Modules/ProfilerEditor/Public/HierarchyFrameDataView.bindings.cs)

  层次化帧数据视图，继承自 FrameDataView，提供：
  - 视图模式：ViewModes (合并同名样本、隐藏编辑器样本等)
  - 层级导航：GetRootItemID(), GetItemMarkerID(), GetItemDepth(), HasItemChildren(), GetItemChildren(), GetItemAncestors(), GetItemDescendantsThatHaveChildren()
  - 项目信息：GetItemName(), GetItemInstanceID(), GetItemColumnData(), GetItemColumnDataAsFloat/Double()
  - 排序：Sort(), sortColumn, sortColumnAscending
  - 路径信息：GetItemPath(), GetItemMarkerIDPath()
  - 元数据：GetItemMetadataCount(), GetItemMetadata(), GetItemMetadataAsLong/Float()
  - 调用栈：GetItemCallstack(), ResolveItemCallstack()

4. ProfilerFrameDataIterator (Modules/ProfilerEditor/Public/ProfilerFrameDataIterator.bindings.cs)

  帧数据迭代器，提供：
  - 迭代功能：Next(enterChildren)
  - 线程信息：GetThreadCount(), GetThreadName(), SetRoot()
  - 组信息：GetGroupCount(), GetGroupName()
  - 时间信息：GetFrameStartS(), frameTimeMS, startTimeMS, durationMS
  - 样本信息：name, path, sampleId, instanceId, depth, maxDepth

###   二、Profiler 驱动与控制类

1. ProfilerDriver (Modules/ProfilerEditor/Public/ProfilerAPI.bindings.cs)

  Profiler 驱动器，提供：
  - 帧管理：ClearAllFrames(), GetNextFrameIndex(), GetPreviousFrameIndex(), firstFrameIndex, lastFrameIndex
  - 加载/保存：LoadProfile(), SaveProfile(), AddFramesFromFile()
  - 数据视图：GetHierarchyFrameDataView(), GetRawFrameDataView()
  - 计数器数据：GetCounterValuesBatch(), GetFormattedCounterValue()
  - 区域控制：SetAreaEnabled(), IsAreaEnabled()
  - 事件：NewProfilerFrameRecorded, profileLoaded, profileCleared

2. ProfilerProperty (Modules/ProfilerEditor/Public/ProfilerProperty.bindings.cs)

  属性查看器，提供：
  - 迭代：Next(), SetRoot(), ResetToRoot()
  - 属性信息：propertyName, propertyPath, depth, HasChildren
  - 列数据：GetColumn(), GetColumnAsSingle()
  - 帧信息：frameFPS, frameTime, frameGpuTime, frameDataReady
  - 音频 Profiler：GetAudioProfilerGroupInfo/DSPInfo/ClipInfo()
  - UI 系统 Profiler：GetUISystemProfilerInfo(), GetUISystemEventMarkers()

###   三、运行时 Profiler API

1. Profiler (Runtime/Profiler/ScriptBindings/Profiler.bindings.cs)

  运行时 Profiler 控制，提供：
  - 控制：enabled, enableBinaryLog, logFile, BeginSample(), EndSample()
  - 内存：usedHeapSizeLong, GetTotalAllocatedMemoryLong(), GetRuntimeMemorySizeLong()
  - 类别：SetAreaEnabled(), GetAreaEnabled(), GetCategoriesCount(), GetAllCategories()
  - 元数据：EmitFrameMetaData<T>(), EmitSessionMetaData<T>()

2. ProfilerRecorder (Runtime/Profiler/ScriptBindings/ProfilerRecorder.bindings.cs)

  记录器，提供：
  - 创建：构造函数支持 ProfilerMarker/ProfilerCategory/statName
  - 控制：Start(), Stop(), Reset(), Dispose()
  - 读取：CurrentValue, LastValue, Count, Capacity, GetSample(), CopyTo(), ToArray()
  - 类型信息：DataType, UnitType

3. ProfilerUnsafeUtility (Runtime/Profiler/ScriptBindings/ProfilerUnsafeUtility.bindings.cs)

  底层工具类，提供：
  - 类别管理：CreateCategory(), GetCategoryDescription(), GetCategoryColor()
  - Marker 管理：CreateMarker(), GetMarker(), SetMarkerMetadata()
  - 采样控制：BeginSample(), EndSample(), BeginSampleWithMetadata(), SingleSampleWithMetadata()
  - 计数器：CreateCounterValue(), FlushCounterValue()
  - 流事件：CreateFlow(), FlowEvent()
  - 时间戳：Timestamp, TimestampToNanosecondsConversionRatio

4. ProfilerMarker (Runtime/Profiler/ScriptBindings/ProfilerMarker.bindings.cs)

  Marker 结构体，提供：
  - 创建：支持 name, category, flags 参数
  - 控制：Begin(), End(), Auto() (自动作用域)
  - 句柄：Handle (IntPtr)

###   四、内存分析

MemoryProfiler (Runtime/Profiler/ScriptBindings/MemoryProfiling.bindings.cs)

  内存快照，提供：
  - 快照：TakeSnapshot(), TakeTempSnapshot()
  - 标志：CaptureFlags (ManagedObjects, NativeObjects, NativeAllocations 等)
  - 元数据事件：CreatingMetadata

###   五、时间线可视化

NativeProfilerTimeline (Modules/ProfilerEditor/Public/NativeProfilerTimeline.bindings.cs)

  时间线渲染，提供：
  - 初始化：Initialize()
  - 绘制：Draw()
  - 查询：GetEntryAtPosition(), GetEntryInstanceInfo(), GetEntryTimingInfo(), GetEntryPositionInfo()

###   六、数据结构

1. ProfilerCategoryInfo - 类别信息

2. MarkerInfo - Marker 信息

3. ProfilerRecorderSample - 记录样本

4. ProfilerCategoryDescription - 类别描述

5. ProfilerRecorderDescription - 记录器描述

### 示例

  如果要解析 .raw Profiler 文件：

  1. 加载文件: 使用 Profiler.AddFramesFromFile() 或 ProfilerDriver.LoadProfile()
  2. 遍历帧: 使用 ProfilerDriver.firstFrameIndex / lastFrameIndex
  3. 获取视图: 使用 ProfilerDriver.GetRawFrameDataView() 或 GetHierarchyFrameDataView()
  4. 读取样本: 使用 RawFrameDataView.GetSample*() 系列方法
  5. 获取元数据: 使用 GetSampleMetadata*() 系列方法
  6. 调用栈解析: 使用 GetSampleCallstack() 和 FrameDataView.ResolveMethodInfo()