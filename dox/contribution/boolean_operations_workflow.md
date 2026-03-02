# OCCT 布尔运算流程

本文档以 Mermaid 图表的形式，完整描述 Open CASCADE Technology 布尔运算的内部执行流程。

---

## 1. 整体架构

```mermaid
graph TB
    subgraph "Layer 1: API 层"
        API_Fuse["BRepAlgoAPI_Fuse"]
        API_Cut["BRepAlgoAPI_Cut"]
        API_Common["BRepAlgoAPI_Common"]
        API_Section["BRepAlgoAPI_Section"]
        API_Bool["BRepAlgoAPI_BooleanOperation"]
    end

    subgraph "Layer 2: 算法层"
        PF["BOPAlgo_PaveFiller<br/>(求交引擎)"]
        BOP["BOPAlgo_BOP<br/>(构建器)"]
        SEC["BOPAlgo_Section<br/>(截面构建器)"]
        BS["BOPAlgo_BuilderSolid<br/>(实体重建)"]
    end

    subgraph "Layer 3: 数据结构层"
        DS["BOPDS_DS<br/>(主数据仓库)"]
        PB["BOPDS_PaveBlock<br/>(棱边分割)"]
        IF["BOPDS_Interf*<br/>(干涉记录)"]
        FI["BOPDS_FaceInfo<br/>(面状态)"]
    end

    API_Fuse --> API_Bool
    API_Cut --> API_Bool
    API_Common --> API_Bool
    API_Section --> API_Bool

    API_Bool -->|"Phase 1: 求交"| PF
    API_Bool -->|"Phase 2: 构建"| BOP
    API_Bool -->|"Phase 2: 截面"| SEC
    BOP -->|"FUSE 3D"| BS

    PF --> DS
    PF --> PB
    PF --> IF
    PF --> FI

    BOP --> DS
    SEC --> DS
```

---

## 2. 类继承体系

```mermaid
classDiagram
    class BRepBuilderAPI_Command {
        +Done() bool
    }
    class BRepBuilderAPI_MakeShape {
        +Build()
        +Shape() TopoDS_Shape
        +Modified()
        +Generated()
        +IsDeleted()
    }
    class BOPAlgo_Options {
        +SetFuzzyValue()
        +SetRunParallel()
        +SetUseOBB()
        +SetGlue()
    }
    class BRepAlgoAPI_Algo {
    }
    class BRepAlgoAPI_BuilderAlgo {
        #myDSFiller: BOPAlgo_PaveFiller*
        #myBuilder: BOPAlgo_Builder*
        +IntersectShapes()
        +BuildResult()
    }
    class BRepAlgoAPI_BooleanOperation {
        #myTools: ListOfShape
        #myOperation: BOPAlgo_Operation
        +Build()
        +SetTools()
        +SetOperation()
    }
    class BRepAlgoAPI_Fuse {
        +BRepAlgoAPI_Fuse(S1, S2)
    }
    class BRepAlgoAPI_Cut {
        +BRepAlgoAPI_Cut(S1, S2)
    }
    class BRepAlgoAPI_Common {
        +BRepAlgoAPI_Common(S1, S2)
    }
    class BRepAlgoAPI_Section {
        +BRepAlgoAPI_Section(S1, S2)
    }

    BRepBuilderAPI_Command <|-- BRepBuilderAPI_MakeShape
    BRepBuilderAPI_MakeShape <|-- BRepAlgoAPI_Algo
    BOPAlgo_Options <|-- BRepAlgoAPI_Algo
    BRepAlgoAPI_Algo <|-- BRepAlgoAPI_BuilderAlgo
    BRepAlgoAPI_BuilderAlgo <|-- BRepAlgoAPI_BooleanOperation
    BRepAlgoAPI_BooleanOperation <|-- BRepAlgoAPI_Fuse
    BRepAlgoAPI_BooleanOperation <|-- BRepAlgoAPI_Cut
    BRepAlgoAPI_BooleanOperation <|-- BRepAlgoAPI_Common
    BRepAlgoAPI_BooleanOperation <|-- BRepAlgoAPI_Section

    class BOPAlgo_Algo {
        +Perform()
        +CheckData()
    }
    class BOPAlgo_PaveFiller {
        #myDS: BOPDS_DS*
        #myIterator: BOPDS_Iterator*
        #myContext: IntTools_Context
        +PerformVV()
        +PerformVE()
        +PerformEE()
        +PerformVF()
        +PerformEF()
        +PerformFF()
        +MakeSplitEdges()
        +MakeBlocks()
    }
    class BOPAlgo_BuilderShape {
        #myShape: TopoDS_Shape
        #myImages: DataMap
    }
    class BOPAlgo_Builder {
        +FillImagesVertices()
        +FillImagesEdges()
        +FillImagesFaces()
        +FillImagesSolids()
    }
    class BOPAlgo_ToolsProvider {
        #myTools: ListOfShape
        +SetTools()
    }
    class BOPAlgo_BOP {
        #myOperation: BOPAlgo_Operation
        #myRC: TopoDS_Shape
        +BuildRC()
        +BuildShape()
        +BuildSolid()
    }

    BOPAlgo_Options <|-- BOPAlgo_Algo
    BOPAlgo_Algo <|-- BOPAlgo_PaveFiller
    BOPAlgo_Algo <|-- BOPAlgo_BuilderShape
    BOPAlgo_BuilderShape <|-- BOPAlgo_Builder
    BOPAlgo_Builder <|-- BOPAlgo_ToolsProvider
    BOPAlgo_ToolsProvider <|-- BOPAlgo_BOP
```

---

## 3. 主流程：BRepAlgoAPI_BooleanOperation::Build()

```mermaid
flowchart TD
    START(["用户调用 Build()"])
    START --> VALIDATE

    subgraph "参数校验"
        VALIDATE{"Arguments 和<br/>Tools 非空?"}
        VALIDATE -->|否| ERR1["AlertTooFewArguments"]
        VALIDATE -->|是| CHK_OP{"Operation<br/>已设置?"}
        CHK_OP -->|否| ERR2["AlertBOPNotSet"]
    end

    CHK_OP -->|是| PHASE1

    subgraph "Phase 1: 求交 (70%)"
        PHASE1["合并 Arguments + Tools"]
        PHASE1 --> CREATE_PF["创建 BOPAlgo_PaveFiller"]
        CREATE_PF --> SET_OPT["设置选项:<br/>Parallel / Fuzzy / Glue / OBB"]
        SET_OPT --> PF_PERFORM["PaveFiller.Perform()"]
    end

    PF_PERFORM --> PF_OK{"求交<br/>成功?"}
    PF_OK -->|否| ERR3["返回错误"]
    PF_OK -->|是| PHASE2

    subgraph "Phase 2: 构建 (30%)"
        PHASE2{"Operation<br/>类型?"}
        PHASE2 -->|"FUSE / CUT / COMMON"| CREATE_BOP["创建 BOPAlgo_BOP<br/>设置 Arguments, Tools, Operation"]
        PHASE2 -->|"SECTION"| CREATE_SEC["创建 BOPAlgo_Section"]
        CREATE_BOP --> BUILD["Builder.PerformWithFiller()"]
        CREATE_SEC --> BUILD
    end

    BUILD --> HISTORY["构建修改历史"]
    HISTORY --> RESULT(["返回 myShape"])

    style PHASE1 fill:#e1f5fe
    style PHASE2 fill:#fff3e0
    style ERR1 fill:#ffcdd2
    style ERR2 fill:#ffcdd2
    style ERR3 fill:#ffcdd2
```

---

## 4. PaveFiller 求交引擎：14 步详解

```mermaid
flowchart TD
    INIT["Init()<br/>创建 BOPDS_DS + Iterator + Context"]
    INIT --> PREPARE["Prepare()<br/>构建平面面的 p-curve"]

    PREPARE --> VV["① PerformVV()<br/>顶点-顶点求交"]
    VV --> VE["② PerformVE()<br/>顶点-棱边求交"]
    VE --> UPD1["UpdatePaveBlocksWithSDVertices()"]

    UPD1 --> EE["③ PerformEE()<br/>棱边-棱边求交"]
    EE --> UPD2["UpdatePaveBlocksWithSDVertices()"]

    UPD2 --> VF["④ PerformVF()<br/>顶点-面求交"]
    VF --> UPD3["UpdatePaveBlocksWithSDVertices()"]

    UPD3 --> EF["⑤ PerformEF()<br/>棱边-面求交"]
    EF --> UPD4["Update SD Vertices + Interferences"]

    UPD4 --> REPEAT["⑥ RepeatIntersection()<br/>容差增大后重新 VV/VE/VF"]
    REPEAT --> FORCE_EE["⑦ ForceInterfEE()<br/>强制棱边-棱边检查"]
    FORCE_EE --> FORCE_EF["⑧ ForceInterfEF()<br/>强制棱边-面检查"]

    FORCE_EF --> FF["⑨ PerformFF()<br/>面-面求交 (最耗时)"]
    FF --> UPD5["UpdateBlocksWithSharedVertices()<br/>RefineFaceInfoIn()"]

    UPD5 --> SPLIT["⑩ MakeSplitEdges()<br/>从 PaveBlock 创建分割棱边"]
    SPLIT --> BLOCKS["⑪ MakeBlocks()<br/>从 FF 交线构建截面棱边"]
    BLOCKS --> CHECK["CheckSelfInterference()"]

    CHECK --> PCURVES["⑫ MakePCurves()<br/>构建面上的 2D 曲线"]
    PCURVES --> DE["⑬ ProcessDE()<br/>处理退化棱边"]
    DE --> MICRO["⑭ RemoveMicroEdges()"]
    MICRO --> DONE(["BOPDS_DS 完成:<br/>包含完整求交数据"])

    style VV fill:#c8e6c9
    style VE fill:#c8e6c9
    style EE fill:#c8e6c9
    style VF fill:#c8e6c9
    style EF fill:#c8e6c9
    style FF fill:#ffcc80
    style SPLIT fill:#b3e5fc
    style BLOCKS fill:#b3e5fc
    style PCURVES fill:#b3e5fc
    style DONE fill:#a5d6a7
```

---

## 5. 求交引擎各步骤的输入输出

```mermaid
flowchart LR
    subgraph "PerformVV"
        VV_IN["两个顶点<br/>距离 ≤ 容差"]
        VV_OUT["Same-Domain 映射<br/>InterfVV 记录"]
        VV_IN --> VV_OUT
    end

    subgraph "PerformVE"
        VE_IN["顶点 + 棱边"]
        VE_OUT["额外 Pave 点<br/>SplitPaveBlocks"]
        VE_IN --> VE_OUT
    end

    subgraph "PerformEE"
        EE_IN["棱边 + 棱边"]
        EE_OUT["交叉点: 新顶点<br/>重合段: CommonBlock"]
        EE_IN --> EE_OUT
    end

    subgraph "PerformVF"
        VF_IN["顶点 + 面"]
        VF_OUT["FaceInfo.VerticesIn<br/>InterfVF 记录"]
        VF_IN --> VF_OUT
    end

    subgraph "PerformEF"
        EF_IN["棱边 + 面"]
        EF_OUT["交叉点: 新顶点<br/>贴面段: CommonBlock"]
        EF_IN --> EF_OUT
    end

    subgraph "PerformFF"
        FF_IN["面 + 面"]
        FF_OUT["交线: BOPDS_Curve<br/>交点: BOPDS_Point"]
        FF_IN --> FF_OUT
    end
```

---

## 6. PaveBlock 棱边分割过程

```mermaid
flowchart TD
    subgraph "原始棱边 E"
        ORIG["V1 ─────────────────── V2<br/>PaveBlock(V1, V2)"]
    end

    ORIG -->|"VE/EE/EF 求交<br/>产生额外 Pave"| PAVES

    subgraph "添加交点"
        PAVES["V1 ──── Vn1 ──── Vn2 ──── V2<br/>ExtPaves: {Vn1, Vn2}"]
    end

    PAVES -->|"PaveBlock.Update()"| SPLIT

    subgraph "分割为多段"
        PB1["PB1: V1 → Vn1"]
        PB2["PB2: Vn1 → Vn2"]
        PB3["PB3: Vn2 → V2"]
    end
    SPLIT --> PB1
    SPLIT --> PB2
    SPLIT --> PB3

    PB1 -->|"MakeSplitEdges()"| E1["新棱边 E1"]
    PB2 -->|"MakeSplitEdges()"| E2["新棱边 E2"]
    PB3 -->|"MakeSplitEdges()"| E3["新棱边 E3"]

    style ORIG fill:#fff9c4
    style PAVES fill:#ffe0b2
    style PB1 fill:#c8e6c9
    style PB2 fill:#c8e6c9
    style PB3 fill:#c8e6c9
```

---

## 7. BOPAlgo_BOP 构建阶段

```mermaid
flowchart TD
    START_BOP(["PerformInternal1()"])
    START_BOP --> CHECK["CheckData()<br/>校验操作类型和维度约束"]

    CHECK --> DIM_RULES

    subgraph DIM_RULES["维度规则"]
        FUSE_R["FUSE: Objects维度 == Tools维度"]
        CUT_R["CUT: max(Objects) ≤ min(Tools)"]
        CUT21_R["CUT21: min(Objects) ≥ max(Tools)"]
        COMMON_R["COMMON: 任意维度"]
    end

    DIM_RULES --> PREPARE_BOP["Prepare()<br/>创建空的结果 Compound"]
    PREPARE_BOP --> EMPTY{"存在空形体?"}
    EMPTY -->|是| TREAT["TreatEmptyShape()<br/>快速路径处理"]
    TREAT -->|"已完成"| HIST
    TREAT -->|"需继续"| FILL
    EMPTY -->|否| FILL

    subgraph FILL["FillImages: 逐维度构建映射"]
        F_V["FillImagesVertices + BuildResult(VERTEX)"]
        F_E["FillImagesEdges + BuildResult(EDGE)"]
        F_W["FillImagesContainers(WIRE)"]
        F_F["FillImagesFaces + BuildResult(FACE)"]
        F_SH["FillImagesContainers(SHELL)"]
        F_S["FillImagesSolids + BuildResult(SOLID)"]
        F_CS["FillImagesContainers(COMPSOLID)"]
        F_C["FillImagesCompounds + BuildResult(COMPOUND)"]

        F_V --> F_E --> F_W --> F_F --> F_SH --> F_S --> F_CS --> F_C
    end

    FILL --> BUILD_SHAPE["BuildShape()"]
    BUILD_SHAPE --> HIST["PrepareHistory()"]
    HIST --> POST["PostTreat()"]
    POST --> RESULT_BOP(["myShape: 最终结果"])

    style FILL fill:#e8f5e9
    style DIM_RULES fill:#f3e5f5
```

---

## 8. BuildShape() 按操作类型的分支

```mermaid
flowchart TD
    BS(["BuildShape()"])
    BS --> IS_3D{"维度 == 3<br/>且实体操作?"}

    IS_3D -->|是| CHK_OPEN["CheckArgsForOpenSolid()<br/>检查是否有开放实体"]
    CHK_OPEN -->|"有开放实体"| BUILD_BOP_ALT["BuildBOP()<br/>基于分类的替代路径"]
    BUILD_BOP_ALT -->|成功| DONE_ALT(["结果"])
    BUILD_BOP_ALT -->|失败| BUILD_RC

    CHK_OPEN -->|"全闭合"| BUILD_RC
    IS_3D -->|否| BUILD_RC

    BUILD_RC["BuildRC()<br/>构建结果复合体"]

    BUILD_RC --> OP_TYPE{"Operation?"}

    OP_TYPE -->|FUSE| FUSE_RC["收集所有分割体"]
    OP_TYPE -->|COMMON| COMMON_RC["ArgsIm ∩ ToolsIm<br/>保留同时存在于两组的"]
    OP_TYPE -->|CUT| CUT_RC["ArgsIm \\ ToolsIm<br/>保留仅在 Objects 中的"]
    OP_TYPE -->|CUT21| CUT21_RC["ToolsIm \\ ArgsIm<br/>保留仅在 Tools 中的"]

    FUSE_RC --> IS_FUSE_3D{"FUSE 且<br/>维度 == 3?"}
    IS_FUSE_3D -->|是| BUILD_SOLID
    IS_FUSE_3D -->|否| ASSEMBLE

    COMMON_RC --> ASSEMBLE
    CUT_RC --> ASSEMBLE
    CUT21_RC --> ASSEMBLE

    subgraph BUILD_SOLID["BuildSolid()"]
        BS1["收集非 INTERNAL 面"]
        BS2["筛选只被 1 个实体引用的面"]
        BS3["BOPAlgo_BuilderSolid 重建实体"]
        BS4["添加未被修改的实体"]
        BS1 --> BS2 --> BS3 --> BS4
    end

    BUILD_SOLID --> DONE(["myShape"])

    subgraph ASSEMBLE["组装容器"]
        ASM1["构建 Wire / Shell"]
        ASM2["OrientEdgesOnWire / OrientFacesOnShell"]
        ASM3["RemoveDuplicates"]
        ASM1 --> ASM2 --> ASM3
    end

    ASSEMBLE --> DONE

    style FUSE_RC fill:#c8e6c9
    style COMMON_RC fill:#bbdefb
    style CUT_RC fill:#ffe0b2
    style CUT21_RC fill:#f8bbd0
    style BUILD_SOLID fill:#fff9c4
```

---

## 9. BuildSolid: FUSE 实体重建

```mermaid
flowchart TD
    INPUT["输入: myRC 中的所有分割体"]
    INPUT --> MAP["MapFacesToBuildSolids()<br/>面 → 引用该面的实体列表"]

    MAP --> FILTER["筛选: 只保留被 1 个实体引用的面<br/>(消除内部共享面)"]

    FILTER --> BUILDER["BOPAlgo_BuilderSolid"]

    subgraph BUILDER["BOPAlgo_BuilderSolid.Perform()"]
        AVOID["PerformShapesToAvoid()<br/>标记 INTERNAL 面"]
        LOOPS["PerformLoops()<br/>构建闭合壳体 (ShellSplitter)"]
        AREAS["PerformAreas()<br/>壳体分类: Growth / Hole"]
        INTERNAL["PerformInternalShapes()<br/>处理内部面"]

        AVOID --> LOOPS --> AREAS --> INTERNAL
    end

    subgraph CLASSIFY["壳体分类"]
        IS_HOLE{"IsHole(Shell)?<br/>无穷远点分类 == IN?"}
        IS_HOLE -->|是| HOLE["标记为 Hole"]
        IS_HOLE -->|否| GROWTH["标记为 Growth<br/>创建实体"]
        GROWTH --> PUT_HOLE["将 Hole 放入<br/>最近的 Growth 中"]
    end

    AREAS --> CLASSIFY
    INTERNAL --> UNTOUCHED["添加未修改的实体"]
    UNTOUCHED --> RESULT(["结果实体列表"])

    style BUILDER fill:#e8f5e9
    style CLASSIFY fill:#fff3e0
```

---

## 10. 完整数据流

```mermaid
flowchart TD
    INPUT["输入:<br/>Objects {S1, S2, ...}<br/>Tools {T1, T2, ...}<br/>Operation: FUSE/CUT/COMMON"]

    INPUT --> INIT

    subgraph INIT["Phase 0: 初始化"]
        DS_INIT["BOPDS_DS.Init()<br/>索引化所有子形体"]
        ITER_INIT["BOPDS_Iterator.Prepare()<br/>包围盒预过滤"]
        CTX_INIT["IntTools_Context<br/>几何计算上下文"]
        DS_INIT --- ITER_INIT --- CTX_INIT
    end

    INIT --> INTERSECT

    subgraph INTERSECT["Phase 1: 求交 (PaveFiller)"]
        direction TB
        VV["VV: 重合顶点"]
        VE["VE: 顶点落在棱边"]
        EE["EE: 棱边交叉/重合"]
        VF["VF: 顶点在面内"]
        EF["EF: 棱边穿面"]
        FF["FF: 曲面求交"]
        SPLIT_E["MakeSplitEdges"]
        MAKE_BLK["MakeBlocks: 截面边"]
        MAKE_PC["MakePCurves: 2D曲线"]

        VV --> VE --> EE --> VF --> EF --> FF --> SPLIT_E --> MAKE_BLK --> MAKE_PC
    end

    INTERSECT --> INTERF_DATA

    subgraph INTERF_DATA["求交结果"]
        INT_VV["InterfVV: 重合顶点对"]
        INT_EE["InterfEE: 棱边交叉点"]
        INT_FF["InterfFF: 交线 + 交点"]
        PB_DATA["PaveBlocks: 分割后的棱边"]
        FI_DATA["FaceInfo: IN/ON/Sc 状态"]
        SD_DATA["ShapesSD: Same-Domain 映射"]
    end

    INTERF_DATA --> BUILD

    subgraph BUILD["Phase 2: 构建 (BOPAlgo_BOP)"]
        FILL_IMG["FillImages:<br/>原始形体 → 分割后形体"]
        BUILD_RC_2["BuildRC: 按操作筛选"]
        BUILD_SHAPE_2["BuildShape: 组装结果"]
        FILL_IMG --> BUILD_RC_2 --> BUILD_SHAPE_2
    end

    BUILD --> OUTPUT

    subgraph OUTPUT["最终输出"]
        SHAPE["myShape: TopoDS_Shape"]
        HISTORY["History: Modified / Generated / Deleted"]
    end

    style INTERSECT fill:#e3f2fd
    style BUILD fill:#fff8e1
    style OUTPUT fill:#e8f5e9
```

---

## 11. 操作类型枚举

```mermaid
graph LR
    subgraph BOPAlgo_Operation
        FUSE["BOPAlgo_FUSE<br/>并集: A ∪ B"]
        COMMON["BOPAlgo_COMMON<br/>交集: A ∩ B"]
        CUT["BOPAlgo_CUT<br/>差集: A \\ B"]
        CUT21["BOPAlgo_CUT21<br/>反差: B \\ A"]
        SECTION["BOPAlgo_SECTION<br/>截面: A ∩ B 的边界"]
    end

    subgraph "结果示意 (2D)"
        F_IMG["FUSE<br/>🟦🟦🟦🟦<br/>🟦🟦🟦🟦"]
        CO_IMG["COMMON<br/>　　🟩🟩<br/>　　🟩🟩"]
        CU_IMG["CUT<br/>🟧🟧　　<br/>🟧🟧　　"]
        CU21_IMG["CUT21<br/>　　　🟪<br/>　　　🟪"]
        SEC_IMG["SECTION<br/>　　┃　<br/>　　┃　"]
    end

    FUSE --- F_IMG
    COMMON --- CO_IMG
    CUT --- CU_IMG
    CUT21 --- CU21_IMG
    SECTION --- SEC_IMG
```

---

## 12. 关键源码文件索引

```mermaid
mindmap
  root((布尔运算<br/>源码))
    API 层
      BRepAlgoAPI_BooleanOperation.hxx/.cxx
      BRepAlgoAPI_Fuse.hxx/.cxx
      BRepAlgoAPI_Cut.hxx/.cxx
      BRepAlgoAPI_Common.hxx/.cxx
      BRepAlgoAPI_Section.hxx/.cxx
      BRepAlgoAPI_BuilderAlgo.hxx/.cxx
    算法层
      BOPAlgo_BOP.hxx/.cxx
      BOPAlgo_Builder.hxx/.cxx
      BOPAlgo_Builder_1~4.cxx
      BOPAlgo_PaveFiller.hxx/.cxx
      BOPAlgo_PaveFiller_1~9.cxx
      BOPAlgo_BuilderSolid.hxx/.cxx
      BOPAlgo_Section.hxx/.cxx
    数据结构层
      BOPDS_DS.hxx/.cxx
      BOPDS_ShapeInfo.hxx
      BOPDS_PaveBlock.hxx/.cxx
      BOPDS_CommonBlock.hxx/.cxx
      BOPDS_FaceInfo.hxx
      BOPDS_Interf.hxx
      BOPDS_Iterator.hxx/.cxx
      BOPDS_Curve.hxx
    工具类
      BOPTools_AlgoTools.hxx/.cxx
      BOPTools_AlgoTools3D.hxx/.cxx
      BOPTools_Set.hxx
      IntTools_EdgeEdge.hxx
      IntTools_FaceFace.hxx
      IntTools_Context.hxx
    测试
      GTests/BOPAlgo_BOP_Test.cxx
      GTests/BRepAlgoAPI_Cut_Test.cxx
      GTests/BRepAlgoAPI_Fuse_Test.cxx
      GTests/BOPAlgo_PaveFiller_Test.cxx
      tests/boolean/ ~4100 Tcl 脚本
```

---

## 13. 测试覆盖矩阵

```mermaid
quadrantChart
    title 布尔运算测试覆盖 (GTest vs Tcl)
    x-axis "GTest 数量少" --> "GTest 数量多"
    y-axis "Tcl 数量少" --> "Tcl 数量多"
    Cut: [0.85, 0.70]
    Fuse: [0.75, 0.75]
    Common: [0.05, 0.65]
    Section: [0.05, 0.35]
    TUC: [0.03, 0.60]
    PaveFiller: [0.03, 0.02]
```

---

## 附录: 源文件路径

所有文件位于 `src/ModelingAlgorithms/TKBO/` 下：

| 包 | 路径 |
|---|------|
| BRepAlgoAPI | `src/ModelingAlgorithms/TKBO/BRepAlgoAPI/` |
| BOPAlgo | `src/ModelingAlgorithms/TKBO/BOPAlgo/` |
| BOPDS | `src/ModelingAlgorithms/TKBO/BOPDS/` |
| BOPTools | `src/ModelingAlgorithms/TKBO/BOPTools/` |
| GTests | `src/ModelingAlgorithms/TKBO/GTests/` |
| Tcl Tests | `tests/boolean/` (35 个测试组, ~4100 个脚本) |
| 官方文档 | `dox/specification/boolean_operations/boolean_operations.md` |
