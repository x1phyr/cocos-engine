图形设备抽象层（GFX）是 Cocos Creator 引擎渲染系统的核心基础设施，它提供了一套统一的图形 API 抽象接口，屏蔽了底层渲染后端（WebGL、WebGL2、WebGPU、OpenGL 等）的实现差异。通过 GFX，引擎上层渲染逻辑可以无需关心具体平台使用的图形 API，实现真正的跨平台渲染能力。

## 架构设计哲学

GFX 层采用**抽象工厂模式**与**桥接模式**相结合的设计，核心思想是将图形资源的生命周期管理、命令录制与执行、以及状态绑定完全解耦。这种设计使得渲染管线可以在不同图形后端之间无缝切换，同时保持高性能的命令批处理与状态缓存机制。

GFX 架构分为三个层次：**设备管理层**、**核心抽象层**和**后端实现层**。设备管理器负责根据运行环境自动选择最优的渲染后端；核心抽象层定义了所有图形对象的标准接口；后端实现层则针对具体图形 API 提供优化实现。

```mermaid
graph TB
    subgraph "应用层"
        A[渲染管线]
        B[场景渲染]
    end
    
    subgraph "GFX 核心抽象层"
        C[Device 设备]
        D[CommandBuffer 命令缓冲]
        E[PipelineState 管线状态]
        F[DescriptorSet 描述符集]
        G[Buffer/Texture 资源]
    end
    
    subgraph "后端实现层"
        H[WebGLDevice]
        I[WebGL2Device]
        J[WebGPUDevice]
        K[EmptyDevice]
    end
    
    subgraph "原生后端"
        L[OpenGL ES]
        M[Metal]
        N[Vulkan]
    end
    
    A --> C
    B --> D
    C --> E
    C --> F
    C --> G
    D --> E
    E --> F
    
    H --> L
    I --> L
    J --> M
    J --> N
    K -.-> L
    
    C -.-> H
    C -.-> I
    C -.-> J
    C -.-> K

Sources: [cocos/gfx/base/device.ts](cocos/gfx/base/device.ts#L1-L499)
Sources: [cocos/gfx/device-manager.ts](cocos/gfx/device-manager.ts#L1-L244)
```

## 核心对象体系

GFX 定义了完整的图形资源抽象体系，所有对象都继承自 `GFXObject` 基类，通过 `ObjectType` 枚举标识对象类型。这种设计使得引擎可以进行统一的资源追踪与生命周期管理。

### 设备与交换链

`Device` 是 GFX 层的核心入口，负责创建和管理所有图形资源。设备封装了底层图形上下文，提供统一的资源创建接口。`Swapchain` 则管理屏幕缓冲的呈现，处理双缓冲/三缓冲机制以及表面变换。

| 属性/方法 | 说明 | 来源 |
|---------|------|------|
| `gfxAPI` | 当前使用的渲染 API 类型 | [device.ts#L61-L65](cocos/gfx/base/device.ts#L61-L65) |
| `capabilities` | 设备能力查询 | [device.ts#L133-L137](cocos/gfx/base/device.ts#L133-L137) |
| `createShader()` | 创建着色器对象 | [device.ts#L238-L243](cocos/gfx/base/device.ts#L238-L243) |
| `createPipelineState()` | 创建渲染管线状态 | [device.ts#L278-L283](cocos/gfx/base/device.ts#L278-L283) |
| `acquire()` / `present()` | 交换链缓冲获取与呈现 | [device.ts#L208-L213](cocos/gfx/base/device.ts#L208-L213) |

`DeviceManager` 是设备初始化的单例管理器，它根据平台能力和配置设置自动选择渲染后端。初始化流程遵循**优先级降级策略**：WebGPU → WebGL2 → WebGL → Headless。

```mermaid
sequenceDiagram
    participant App as 应用程序
    participant DM as DeviceManager
    participant Device as GFX Device
    participant SC as Swapchain
    
    App->>DM: init(canvas, bindingMappingInfo)
    DM->>DM: 查询 renderMode 设置
    DM->>DM: _determineRenderType()
    
    alt 支持 WebGPU
        DM->>Device: 创建 WebGPUDevice
    else 支持 WebGL2
        DM->>Device: 创建 WebGL2Device
    else 支持 WebGL
        DM->>Device: 创建 WebGLDevice
    else 仅 Headless
        DM->>Device: 创建 EmptyDevice
    end
    
    Device->>Device: initialize(deviceInfo)
    DM->>SC: createSwapchain(swapchainInfo)
    DM-->>App: 初始化完成

Sources: [cocos/gfx/device-manager.ts](cocos/gfx/device-manager.ts#L125-L180)
```

Sources: [cocos/gfx/base/device.ts](cocos/gfx/base/device.ts#L54-L499)
Sources: [cocos/gfx/device-manager.ts](cocos/gfx/device-manager.ts#L88-L244)
Sources: [cocos/gfx/base/swapchain.ts](cocos/gfx/base/swapchain.ts#L1-L80)

### 命令缓冲与队列

命令缓冲（`CommandBuffer`）是 GFX 性能优化的关键机制。它将渲染命令录制到缓冲区中，支持延迟执行和批量提交。`Queue` 则负责管理命令缓冲的执行顺序和同步。

GFX 支持两种命令缓冲类型：
- **Primary CommandBuffer**：主命令缓冲，可直接提交到设备执行
- **Secondary CommandBuffer**：次级命令缓冲，需录制到主缓冲中

这种设计借鉴了现代图形 API（如 Vulkan、WebGPU）的命令录制模型，允许并行录制多个命令缓冲，然后在提交时合并执行，显著提升多线程渲染效率。

```mermaid
graph LR
    subgraph "命令录制阶段"
        A[应用逻辑] --> B[CommandBuffer.record()]
        B --> C[bindPipelineState]
        B --> D[bindDescriptorSet]
        B --> E[bindInputAssembler]
        B --> F[draw/ dispatch]
    end
    
    subgraph "命令提交阶段"
        G[Device.flushCommands] --> H[Queue.execute]
        H --> I[CommandBuffer.execute]
        I --> J[底层 API 调用]
    end
    
    subgraph "呈现阶段"
        K[Device.acquire] --> L[渲染到离屏缓冲]
        L --> M[Device.present]
        M --> N[上屏显示]
    end

Sources: [cocos/gfx/base/command-buffer.ts](cocos/gfx/base/command-buffer.ts#L1-L287)
```

命令缓冲内部维护了绘制统计信息（`numDrawCalls`、`numInstances`、`numTris`），这些数据可用于性能分析和性能计数器报告。

Sources: [cocos/gfx/base/command-buffer.ts](cocos/gfx/base/command-buffer.ts#L52-L89)
Sources: [cocos/gfx/base/queue.ts](cocos/gfx/base/queue.ts#L1-L100)

### 管线状态对象（PSO）

`PipelineState` 是 GFX 中最重要的渲染状态封装对象，它将渲染管线的所有配置信息聚合为一个不可变的整体。这种设计符合现代图形 API 的**管线状态对象（Pipeline State Object, PSO）**理念，相比传统 OpenGL 的状态机模式，PSO 可以预先验证所有状态兼容性，减少运行时状态切换开销。

`PipelineStateInfo` 包含以下核心配置：

| 组件 | 说明 | 配置项 |
|-----|------|--------|
| `shader` | 着色器程序 | 顶点/片段/计算着色器 |
| `pipelineLayout` | 管线布局 | 描述符集布局、推送常量范围 |
| `renderPass` | 渲染过程 | 颜色/深度模板附件配置 |
| `inputState` | 输入状态 | 顶点属性格式、绑定布局 |
| `rasterizerState` | 光栅化状态 | 剔除模式、填充模式、多采样 |
| `depthStencilState` | 深度模板状态 | 深度测试/写入、模板测试/操作 |
| `blendState` | 混合状态 | 混合因子、混合方程、颜色掩码 |
| `primitive` | 图元类型 | 点/线/三角形列表或扇形 |
| `dynamicStates` | 动态状态 | 可动态切换的状态位掩码 |

```mermaid
graph TB
    subgraph "PipelineState 组成"
        S[Shader]
        PL[PipelineLayout]
        RP[RenderPass]
        IS[InputState]
        RS[RasterizerState]
        DSS[DepthStencilState]
        BS[BlendState]
    end
    
    PSO[PipelineState] --> S
    PSO --> PL
    PSO --> RP
    PSO --> IS
    PSO --> RS
    PSO --> DSS
    PSO --> BS
    
    PL --> DSL[DescriptorSetLayout]
    DSL --> B1[Binding 0: UniformBuffer]
    DSL --> B2[Binding 1: Sampler + Texture]
    DSL --> B3[Binding 2: StorageBuffer]
    
    RS --> CM[CullMode]
    RS --> PM[PolygonMode]
    RS --> FO[FrontFace]
    
    DSS --> DT[DepthTest]
    DSS --> DW[DepthWrite]
    DSS --> SF[StencilFunc]
    DSS --> SO[StencilOp]
    
    BS --> BE[BlendEquation]
    BS --> BF[BlendFactor]
    BS --> CM2[ColorMask]

Sources: [cocos/gfx/base/pipeline-state.ts](cocos/gfx/base/pipeline-state.ts#L1-L162)
Sources: [cocos/gfx/base/define.ts](cocos/gfx/base/define.ts#L600-L800)
```

这种设计使得渲染状态切换变为**原子操作**：只需绑定一个新的 PipelineState，即可一次性切换所有相关状态，避免了传统 OpenGL 中多次状态调用的开销。

Sources: [cocos/gfx/base/pipeline-state.ts](cocos/gfx/base/pipeline-state.ts#L36-L162)

### 描述符系统

描述符系统是现代图形 API 中用于管理着色器资源绑定的核心机制。GFX 通过 `DescriptorSetLayout`、`PipelineLayout` 和 `DescriptorSet` 三个对象实现了这一系统。

**DescriptorSetLayout** 定义了描述符集的结构，指定每个绑定位置（binding）的资源类型和访问权限。**PipelineLayout** 则组织多个描述符集布局，并定义推送常量（Push Constant）的范围。**DescriptorSet** 是实际的资源绑定容器，在运行时将具体的 Buffer、Texture、Sampler 绑定到布局定义的槽位上。

```mermaid
graph TB
    subgraph "布局定义阶段（初始化时创建一次）"
        DSL1[DescriptorSetLayout 0]
        DSL2[DescriptorSetLayout 1]
        PL[PipelineLayout]
    end
    
    subgraph "资源绑定阶段（每帧/每材质更新）"
        DS1[DescriptorSet 0]
        DS2[DescriptorSet 1]
    end
    
    subgraph "着色器资源"
        UB[UniformBuffer]
        T[Texture]
        S[Sampler]
        SB[StorageBuffer]
    end
    
    DSL1 --> B0["Binding 0: UniformBuffer"]
    DSL1 --> B1["Binding 1: Sampler + Texture"]
    DSL2 --> B2["Binding 2: StorageBuffer"]
    
    PL --> DSL1
    PL --> DSL2
    
    DS1 -.bindBuffer.-> UB
    DS1 -.bindTexture.-> T
    DS1 -.bindSampler.-> S
    DS2 -.bindBuffer.-> SB
    
    DS1 --> DSL1
    DS2 --> DSL2

Sources: [cocos/gfx/base/descriptor-set.ts](cocos/gfx/base/descriptor-set.ts#L1-L145)
```

描述符类型通过位掩码区分：
- `DESCRIPTOR_BUFFER_TYPE`：UniformBuffer、StorageBuffer
- `DESCRIPTOR_SAMPLER_TYPE`：Sampler、Texture

这种设计允许同一个绑定位置支持多种资源类型的组合，提高了布局的灵活性。

Sources: [cocos/gfx/base/descriptor-set.ts](cocos/gfx/base/descriptor-set.ts#L67-L145)
Sources: [cocos/gfx/base/define.ts](cocos/gfx/base/define.ts#L1000-L1200)

### 资源管理：Buffer 与 Texture

`Buffer` 和 `Texture` 是 GFX 中最基础的图形资源对象，它们分别对应 GPU 内存中的缓冲区和纹理数据。

**BufferUsageBit** 和 **TextureUsageBit** 枚举定义了资源的用途，这些信息帮助底层驱动优化内存分配策略：

| BufferUsageBit | 说明 | 典型用途 |
|---------------|------|---------|
| `TRANSFER_SRC` | 可作为复制源 | 从 CPU 上传数据到 GPU |
| `TRANSFER_DST` | 可作为复制目标 | 从 GPU 下载数据到 CPU |
| `VERTEX` | 顶点缓冲 | 存储顶点属性数据 |
| `INDEX` | 索引缓冲 | 存储索引数据 |
| `UNIFORM` | 均匀缓冲 | 存储着色器均匀量 |
| `STORAGE` | 存储缓冲 | 着色器随机读写 |
| `INDIRECT` | 间接绘制缓冲 | 存储绘制命令参数 |

| TextureUsageBit | 说明 | 典型用途 |
|----------------|------|---------|
| `SAMPLED` | 可采样读取 | 纹理贴图、查找表 |
| `STORAGE` | 可存储读写 | 计算着色器输出 |
| `COLOR_ATTACHMENT` | 颜色附件 | 离屏渲染目标 |
| `DEPTH_STENCIL_ATTACHMENT` | 深度模板附件 | 深度/模板缓冲 |
| `TRANSFER_DST` | 可复制写入 | 纹理上传 |

**MemoryUsageBit** 进一步区分内存分配策略：
- `DEVICE`：仅 GPU 可访问，适合静态资源
- `HOST`：CPU 可访问，适合频繁更新的动态资源
- `DEVICE | HOST`：两者都可访问，但可能有性能开销

Sources: [cocos/gfx/base/define.ts](cocos/gfx/base/define.ts#L340-L420)
Sources: [cocos/gfx/base/buffer.ts](cocos/gfx/base/buffer.ts#L1-L200)
Sources: [cocos/gfx/base/texture.ts](cocos/gfx/base/texture.ts#L1-L300)

## 后端实现架构

GFX 支持多种渲染后端，每种后端都有独立的实现目录。后端实现遵循**继承 + 特化**的模式：继承基类的抽象接口，特化底层 API 调用。

### WebGL 后端

WebGL 后端（`cocos/gfx/webgl/`）是 Cocos Creator 在 Web 平台的主要渲染后端。`WebGLDevice` 继承自 `Device`，实现了所有抽象方法。

WebGL 后端的关键特性：
- **状态缓存**：`WebGLStateCache` 缓存当前绑定的状态，避免冗余的 WebGL API 调用
- **扩展管理**：通过 `IWebGLExtensions` 统一管理 WebGL 扩展的可用性检测
- **命令录制**：`WebGLCommandBuffer` 将命令录制为函数调用序列，在 `flushCommands` 时批量执行
- **交换链集成**：`WebGLSwapchain` 管理 canvas 上下文，处理上下文丢失与恢复

```mermaid
graph TB
    subgraph "WebGLDevice 初始化"
        A[Device.canvas] --> B[获取 WebGL 上下文]
        B --> C{WebGL2 支持？}
        C -->|是 | D[创建 WebGL2 上下文]
        C -->|否 | E[创建 WebGL 上下文]
        D --> F[检测扩展]
        E --> F
        F --> G[初始化状态缓存]
        G --> H[创建默认队列和命令缓冲]
        H --> I[初始化格式特性表]
    end
    
    subgraph "资源创建流程"
        J[createTexture] --> K[创建 WebGLTexture]
        J --> L[设置格式转换]
        J --> M[应用格式特性]
        
        N[createShader] --> O[编译着色器源码]
        N --> P[链接程序]
        N --> Q[查询 uniform 位置]
        N --> R[缓存反射信息]
    end

Sources: [cocos/gfx/webgl/webgl-device.ts](cocos/gfx/webgl/webgl-device.ts#L1-L100)
```

### WebGL2 后端

WebGL2 后端（`cocos/gfx/webgl2/`）与 WebGL 后端结构相似，但利用了 WebGL2 的新特性：
- **Uniform Buffer Object (UBO)**：支持块状均匀量，减少 uniform 更新开销
- **Transform Feedback**：支持顶点着色器输出到缓冲
- **Sampler Objects**：分离采样器状态与纹理绑定
- **Multiple Render Targets (MRT)**：支持多渲染目标同时输出

### WebGPU 后端

WebGPU 后端（`cocos/gfx/webgpu/`）是面向未来的渲染后端，采用了更接近现代图形 API 的设计：
- **异步资源创建**：大部分资源创建返回 Promise，适应 WebGPU 的异步特性
- **实例化命令缓冲**：`WebGPUCommandAllocator` 管理命令缓冲的复用
- **描述符池**：预分配描述符，减少运行时分配开销

### Empty 后端

Empty 后端（`cocos/gfx/empty/`）是一个空实现，所有方法都是空操作（no-op）。它主要用于：
- **服务器端渲染测试**：无需图形硬件的环境
- **功能降级**：当所有图形后端都不可用时的备用方案
- **单元测试**：隔离图形依赖的纯逻辑测试

Sources: [cocos/gfx/empty/empty-device.ts](cocos/gfx/empty/empty-device.ts#L1-L100)

## 渲染流程集成

GFX 层与上层渲染管线的集成通过以下流程实现：

```mermaid
sequenceDiagram
    participant App as 应用/游戏逻辑
    participant Root as Root 模块
    participant RP as RenderPipeline
    participant RS as RenderStage
    participant GFX as GFX Device
    participant Native as 底层图形 API
    
    App->>Root: frame start
    Root->>RP: render()
    
    loop 每个渲染阶段
        RP->>RS: execute()
        
        RS->>GFX: acquire()
        RS->>GFX: createCommandBuffer()
        
        loop 录制渲染命令
            RS->>GFX: cmdBuff.bindPipelineState()
            RS->>GFX: cmdBuff.bindDescriptorSet()
            RS->>GFX: cmdBuff.bindInputAssembler()
            RS->>GFX: cmdBuff.draw() / dispatch()
        end
        
        RS->>GFX: Device.flushCommands()
        RS->>GFX: Device.present()
    end
    
    GFX->>Native: 执行命令缓冲
    Native-->>GFX: 渲染完成
    GFX-->>RS: frame end

Sources: [cocos/gfx/device-manager.ts](cocos/gfx/device-manager.ts#L125-L180)
```

渲染管线在每个帧周期中：
1. 调用 `Device.acquire()` 获取下一个交换链缓冲
2. 创建或复用命令缓冲
3. 录制渲染命令（绑定管线状态、描述符集、输入汇编器，执行绘制）
4. 调用 `Device.flushCommands()` 提交命令缓冲
5. 调用 `Device.present()` 呈现结果到屏幕

这种设计将渲染命令的录制与执行分离，允许渲染管线在命令缓冲中预先录制多帧的命令，实现高效的流水线并行。

## 特性检测与降级策略

GFX 通过 `DeviceCaps` 和 `Feature` 枚举提供设备能力查询机制。渲染管线可以在运行时查询设备是否支持特定功能，并据此调整渲染策略。

```typescript
// 特性查询示例
if (device.capabilities.supportFeature(Feature.COMPUTE_SHADER)) {
    // 使用计算着色器进行粒子更新
} else {
    // 降级到 CPU 更新或顶点着色器方案
}

// 格式特性查询
const formatFeature = device.getFormatFeature(Format.RGBA16F);
if (formatFeature & FormatFeatureBit.STORAGE_TEXTURE) {
    // 可作为存储纹理使用
}
```

| Feature | 说明 | 降级策略 |
|---------|------|---------|
| `COMPUTE_SHADER` | 计算着色器支持 | 使用顶点着色器或 CPU 模拟 |
| `INSTANCED_ARRAYS` | 实例化数组 | 使用 CPU 批处理合并网格 |
| `MULTIPLE_RENDER_TARGETS` | 多渲染目标 | 使用多趟渲染分离输出 |
| `ELEMENT_INDEX_UINT` | 32 位索引 | 限制网格顶点数到 65535 |
| `BLEND_MINMAX` | 最小/最大混合 | 使用自定义着色器模拟 |

Sources: [cocos/gfx/base/define.ts](cocos/gfx/base/define.ts#L85-L105)
Sources: [cocos/gfx/base/device.ts](cocos/gfx/base/device.ts#L133-L137)

## 与渲染管线的协作

GFX 层为上层渲染管线（`cocos/rendering/`）提供基础图形原语，渲染管线则负责组织和调度这些原语完成场景渲染。

**关键协作点**：
- **渲染流程（RenderFlow）**：定义渲染阶段的执行顺序
- **渲染阶段（RenderStage）**：在特定阶段录制渲染命令到 GFX 命令缓冲
- **渲染队列（RenderQueue）**：收集并排序可渲染对象，批量提交到 GFX
- **全局描述符集管理**：`GlobalDescriptorSetManager` 管理相机、场景级别的均匀量绑定

这种分层设计使得渲染算法可以在不修改 GFX 层的前提下进行迭代和优化，同时也使得 GFX 层可以独立演进，支持新的图形 API。

Sources: [cocos/rendering/render-stage.ts](cocos/rendering/render-stage.ts#L1-L100)
Sources: [cocos/rendering/render-pipeline.ts](cocos/rendering/render-pipeline.ts#L1-L150)

## 最佳实践

### 1. 减少管线状态切换

PipelineState 的切换是渲染中开销较大的操作。应尽可能将使用相同管线状态的物体合并渲染，减少状态切换次数。

```typescript
// 不推荐：频繁切换管线状态
for (const obj of objects) {
    cmdBuff.bindPipelineState(obj.pipelineState);
    cmdBuff.bindDescriptorSet(obj.descriptorSet);
    cmdBuff.draw();
}

// 推荐：按管线状态分组
const groups = groupBy(objects, 'pipelineState');
for (const group of groups) {
    cmdBuff.bindPipelineState(group.pipelineState);
    for (const obj of group.objects) {
        cmdBuff.bindDescriptorSet(obj.descriptorSet);
        cmdBuff.draw();
    }
}
```

### 2. 描述符集复用

DescriptorSet 的创建和销毁开销较大。对于频繁更新的资源（如每帧更新的均匀量），应复用已有的 DescriptorSet，仅调用 `update()` 方法更新绑定。

### 3. 命令缓冲录制优化

对于静态场景或重复使用的渲染命令，可以预先录制命令缓冲并缓存，在后续帧中直接复用，减少 CPU 端的录制开销。

### 4. 资源生命周期管理

GFX 对象需要手动调用 `destroy()` 释放底层资源。应确保在对象不再使用时及时销毁，避免 GPU 内存泄漏。使用 `GCObject` 基类的对象可以利用引擎的垃圾回收机制自动管理。

## 下一步阅读

- **[渲染管线架构](17-xuan-ran-guan-xian-jia-gou)**：了解 GFX 如何被上层渲染管线使用
- **[着色器与效果系统](18-zhao-se-qi-yu-xiao-guo-xi-tong)**：深入学习着色器编译、效果资产与管线的集成
- **[3D 渲染管线](10-3d-xuan-ran-guan-xian)**：查看完整的 3D 渲染流程如何利用 GFX 进行场景渲染