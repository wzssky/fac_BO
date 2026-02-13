graph TD
    %% --- 样式定义 ---
    classDef config fill:#f9f,stroke:#333,stroke-width:2px,color:black;
    classDef mainController fill:#ff9,stroke:#f66,stroke-width:4px,color:black;
    classDef geometryLayer fill:#dbeafe,stroke:#3b82f6,stroke-width:2px,color:black;
    classDef optimizationLayer fill:#d1fae5,stroke:#10b981,stroke-width:2px,color:black;
    classDef evaluationLayer fill:#f3e8ff,stroke:#a855f7,stroke-width:2px,color:black;
    classDef externalTool fill:#e5e7eb,stroke:#9ca3af,stroke-dasharray: 5 5,color:black;
    classDef outputData fill:#fef08a,stroke:#eab308,stroke-width:2px,color:black;

    %% --- 配置层 ---
    subgraph Configuration["配置层 (Driver)"]
        ConfigYAML[/"config.yaml\n(定义21D变量, 1D保真度,\n边界/默认值, 分组策略,\n制造/几何硬约束)"/]:::config
    end

    %% --- 主控层 ---
    subgraph MainControl["主控层 (Orchestrator)"]
        MainPY("main.py\n(迭代调度, Sobol初始化,\n保真度选择/分组轮换,\n超体积统计, 可中断恢复)"):::mainController
    end

    %% --- 几何层 ---
    subgraph GeometryLayer["几何层 (Constraint Handler)"]
        GeometryPY("geometry.py\n(参数空间映射, 约束校验,\n约束感知采样)"):::geometryLayer
        Constraints{"五类约束校验\n(结构平滑性,\n模式匹配,\n可制造性等)"}:::geometryLayer
    end

    %% --- 优化层 ---
    subgraph OptimizationLayer["优化层 (The Brain)"]
        OptimizerPY("optimizer.py\n(分组坐标式\n多目标贝叶斯优化)"):::optimizationLayer
        GPModel[/"代理模型 (GP)\n(SingleTaskGP / \nSingleTaskMultiFidelityGP)\n拟合双目标响应"/]:::optimizationLayer
        AcqFunc{"采集函数 (EHVI)\n(多保真模式下引入\n成本加权权衡)"}:::optimizationLayer
    end

    %% --- 评估层 ---
    subgraph EvaluationLayer["评估层 (Simulator Pipeline)"]
        LumFDTD("lum_fdtd.py\n(流水线构建器)"):::evaluationLayer
        
        subgraph Pipeline ["评估流水线"]
            GDSGen["GDS 生成"]:::evaluationLayer
            LithoSim["DUV光刻/工艺近似\n(调用 fac)"]:::externalTool
            FDTDSim["3D FDTD 电磁仿真\n(Lumerical lumapi)\n(高/低精度网格 .fsp)"]:::externalTool
            MetricCalc["指标提取与计算\n(双偏振透射/串扰)"]:::evaluationLayer
        end
        
        Objectives[/"多目标 (越大越优)\nFOM1 = TTE→3 + TTM→2\nFOM2 = -(TTE→2 + TTM→3)"/]:::evaluationLayer
    end

    %% --- 输出数据 ---
    subgraph Outputs["输出数据"]
        JSONLogs[/"JSONL 过程日志"/]:::outputData
        Checkpoints[/"NPZ 检查点/摘要"/]:::outputData
    end

    %% --- 核心流程连接 ---

    %% 1. 配置加载
    ConfigYAML ==>|读取定义与约束| MainPY

    %% 2. 初始化与循环
    MainPY -.->|1. Sobol初始化/采样请求| GeometryPY
    GeometryPY -.->|返回可行初始解| MainPY
    
    %% 循环主体
    MainPY ==>"2. 迭代调度循环 (Start Loop)"==> OptimizerPY
    
    %% 优化层内部
    OptimizerPY -->|利用| GPModel
    GPModel -->|提供预测与不确定性| AcqFunc
    AcqFunc -->|推荐候选解| OptimizerPY
    
    %% 校验候选解
    OptimizerPY --"3. 候选解校验请求"--> GeometryPY
    GeometryPY --"执行五类约束校验"--> Constraints
    Constraints --"返回校验结果 (可行性)"--> GeometryPY
    GeometryPY --"返回经过校验的候选参数"--> MainPY

    %% 评估流程
    MainPY ==>"4. 发送参数与保真度指令"==> LumFDTD
    LumFDTD -->|参数→版图| GDSGen
    GDSGen -->|调用| LithoSim
    LithoSim -->|制造感知版图| FDTDSim
    FDTDSim -->|仿真结果提取| MetricCalc
    MetricCalc -->|计算| Objectives
    Objectives ==>"5. 返回 FOM1, FOM2"==> MainPY

    %% 更新模型
    MainPY --"6. 更新观测数据 (Tell)"--> OptimizerPY
    OptimizerPY -->|重新拟合| GPModel

    %% 日志记录
    MainPY --"实时记录/周期性保存"--> JSONLogs & Checkpoints

    %% 样式调整连接线
    style MainControl fill:none,stroke:none
