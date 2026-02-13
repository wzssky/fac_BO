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
    subgraph Configuration ["配置层 (Driver)"]
        ConfigYAML[/"config.yaml <br/> (21D变量, 1D保真度, <br/> 边界/分组/硬约束)"/]:::config
    end

    %% --- 主控层 ---
    subgraph MainControl ["主控层 (Orchestrator)"]
        MainPY("main.py <br/> (迭代调度, Sobol初始化, <br/> 保真度选择, 可中断恢复)"):::mainController
    end

    %% --- 几何层 ---
    subgraph GeometryLayer ["几何层 (Constraint Handler)"]
        GeometryPY("geometry.py <br/> (参数空间映射, 约束校验, <br/> 约束感知采样)"):::geometryLayer
        Constraints{"五类约束校验 <br/> (结构平滑性, <br/> 模式匹配, <br/> 可制造性)"}:::geometryLayer
    end

    %% --- 优化层 ---
    subgraph OptimizationLayer ["优化层 (The Brain)"]
        OptimizerPY("optimizer.py <br/> (分组坐标式 <br/> 多目标贝叶斯优化)"):::optimizationLayer
        GPModel[/"代理模型 (GP) <br/> (SingleTaskGP / <br/> MultiFidelityGP)"/]:::optimizationLayer
        AcqFunc{"采集函数 (EHVI) <br/> (多保真成本加权权衡)"}:::optimizationLayer
    end

    %% --- 评估层 ---
    subgraph EvaluationLayer ["评估层 (Simulator Pipeline)"]
        LumFDTD("lum_fdtd.py <br/> (评估流水线)"):::evaluationLayer
        
        subgraph Pipeline ["评估流水线细节"]
            GDSGen["GDS 生成"]:::evaluationLayer
            LithoSim["DUV光刻/工艺近似 <br/> (调用 fac)"]:::externalTool
            FDTDSim["3D FDTD 仿真 <br/> (Lumerical lumapi)"]:::externalTool
            MetricCalc["指标提取与计算"]:::evaluationLayer
        end
        
        Objectives[/"多目标 (越大越优) <br/> FOM1 = TTE→3 + TTM→2 <br/> FOM2 = -(TTE→2 + TTM→3)"/]:::evaluationLayer
    end

    %% --- 输出数据 ---
    subgraph Outputs ["输出数据"]
        JSONLogs[/"JSONL 过程日志"/]:::outputData
        Checkpoints[/"NPZ 检查点/摘要"/]:::outputData
    end

    %% --- 流程连接 ---
    ConfigYAML ==> MainPY
    MainPY -.-> GeometryPY
    GeometryPY -.-> MainPY
    MainPY ==> OptimizerPY
    OptimizerPY --> GPModel
    GPModel --> AcqFunc
    AcqFunc --> OptimizerPY
    OptimizerPY -- "校验请求" --> GeometryPY
    GeometryPY -- "执行校验" --> Constraints
    Constraints -- "返回结果" --> GeometryPY
    GeometryPY ==> MainPY
    MainPY ==> LumFDTD
    LumFDTD --> GDSGen
    GDSGen --> LithoSim
    LithoSim --> FDTDSim
    FDTDSim --> MetricCalc
    MetricCalc --> Objectives
    Objectives ==> MainPY
    MainPY -- "更新 (Tell)" --> OptimizerPY
    MainPY -- "持久化" --> JSONLogs & Checkpoints
