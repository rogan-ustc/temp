# temp
# fig1

flowchart LR
  %% Direction
  %% LR: left-to-right for wide pipelines
  %% TD is also fine; choose per page width
  %% Core pipeline from inputs to evaluations

  subgraph Inputs["输入与目标"]
    X["特征 X"]:::inp
    T["处理 T"]:::inp
    Y["结局 Y"]:::inp
    delta["对抗半径 δ"]:::param
    goal["输出：学得工具变量 Z 与反事实预测 Ŷ"]:::goal
  end

  X --> A1
  delta --> A1

  subgraph Stage1["阶段一：对抗样本（放大混杂影响）"]
    A1["求解 ε = arg max_{||ε|| ≤ δ}  L_CFP(f(X+ε))"]:::proc
    A2["生成对抗样本 X_adv = X + ε"]:::proc
  end

  A1 --> A2

  subgraph Stage2["阶段二：混杂生成（对抗式代理）"]
    B1["生成网络 G"]:::block
    B2["U = G(X_adv, X)（或同构形式）"]:::proc
    note2["不需对 U 施加高斯等分布假设"]:::note
  end

  A2 --> B1 --> B2
  B2 --> note2

  subgraph Stage3["阶段三：IV 学习与两阶段回归（端到端）"]
    C1["学习工具变量编码 Z = f_Z(X)"]:::block
    C2["学习混杂表征 C = f_C(X)"]:::block
    C3["第一阶段：拟合处理模型  T̂ = g1(Z, C)"]:::proc
    C4["第二阶段：拟合结局模型  Ŷ = g2(T̂, C)"]:::proc
  end

  X --> C1
  X --> C2
  C1 --> C3
  C2 --> C3 --> C4

  subgraph TrainGame["训练与对抗博弈（交替优化）"]
    D1["更新生成器 G"]:::train
    D2["更新 IV 编码器 f_Z"]:::train
    D3["更新混杂编码器 f_C"]:::train
    D4["更新两阶段回归 g1, g2"]:::train
    D5["目标1：最小化 Z 与 U 的依赖度量/判别损失"]:::loss
    D6["目标2：联合最小化反事实预测损失"]:::loss
  end

  %% Couplings
  B2 -. U .- D5
  C1 -. Z .- D5
  C4 --> D6

  %% Alternating updates loop-back arrows
  D1 --> B2
  D2 --> C1
  D3 --> C2
  D4 --> C3
  D5 --> D2
  D5 --> D3
  D6 --> D4

  subgraph Eval["下游集成与评测"]
    E1["将学得的 Z 接入多类 IV 反事实骨干：Poly2SLS / NN2SLS / KernelIV / DualIV / DeepIV / OneSIV / DFIV / DeepGMM / AGMM"]:::eval
    E2["覆盖多种因果结构与噪声分布，评估稳健性与泛化性"]:::eval
  end

  C4 --> Eval
  goal --- Eval

  classDef inp fill:#e6f2ff,stroke:#4a90e2,stroke-width:1px,color:#0b3d91;
  classDef param fill:#fff3e6,stroke:#ff8c00,stroke-width:1px,color:#8a4b00;
  classDef goal fill:#e8fff2,stroke:#00a85a,stroke-width:1px,color:#00663a;
  classDef block fill:#eef,stroke:#66c,stroke-width:1px;
  classDef proc fill:#f7f7ff,stroke:#889,stroke-width:1px;
  classDef note fill:#ffffe6,stroke:#cc9,stroke-dasharray: 4 2,color:#665;
  classDef train fill:#f0f9ff,stroke:#38a,stroke-dasharray: 5 3;
  classDef loss fill:#ffeef0,stroke:#e5536b,stroke-width:1px,color:#8a2030;
  classDef eval fill:#f2fff8,stroke:#38a169,stroke-width:1px,color:#185f3e;

# fig2

flowchart TB
  %% Focus on Stage-1 and Stage-2 architectural innovations

  subgraph S1["阶段一：对抗驱动的混杂放大"]
    X1["输入特征 X"]:::inp
    delta1["对抗半径 δ"]:::param
    fnet["判别/任务网络 f(·)（用于 L_CFP）"]:::block
    eps["ε = arg max_{||ε|| ≤ δ}  L_CFP(f(X+ε))"]:::proc
    Xadv["对抗样本  X_adv = X + ε"]:::proc
    X1 --> fnet
    X1 --> eps
    delta1 --> eps
    fnet --> eps
    eps --> Xadv
    noteS1["通过 X_adv 放大潜在混杂对下游学习的不利影响"]:::note
    Xadv --> noteS1
  end

  subgraph S2["阶段二：生成式混杂代理（对抗式外生性约束）"]
    G["生成器 G"]:::block
    U["混杂代理  U = G(X_adv, X)  或同构形式"]:::proc
    judge["外生性判别器/依赖度量  D(Z, U)"]:::block
    claim["不需对 U 做高斯等分布假设"]:::note
  end

  %% Stage-2 data flow
  X1 --> G
  Xadv --> G
  G --> U
  U --> claim

  %% Bridge to Stage-3 (for context)
  Zenc["IV 编码器 f_Z(X) → Z"]:::block
  Cenc["混杂编码器 f_C(X) → C"]:::block

  X1 --> Zenc
  X1 --> Cenc
  Zenc -. Z .- judge
  U -. U .- judge

  %% Adversarial game signals
  adv1["对抗信号：最大化 D 对 (Z, U) 的可分性"]:::loss
  adv2["对抗信号：最小化 Z 与 U 的依赖/判别损失（促外生性）"]:::loss

  judge --> adv1
  Zenc --> adv2
  G --> adv2

  classDef inp fill:#e6f2ff,stroke:#4a90e2,stroke-width:1px,color:#0b3d91;
  classDef param fill:#fff3e6,stroke:#ff8c00,stroke-width:1px,color:#8a4b00;
  classDef block fill:#eef,stroke:#66c,stroke-width:1px;
  classDef proc fill:#f7f7ff,stroke:#889,stroke-width:1px;
  classDef note fill:#ffffe6,stroke:#cc9,stroke-dasharray: 4 2,color:#665;
  classDef loss fill:#ffeef0,stroke:#e5536b,stroke-width:1px,color:#8a2030;



 
