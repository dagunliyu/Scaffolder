# 方案：Scaffolder & Blender MCP 结合设计规约

本方案将组织工程的多孔轻量化结构/三周期无界极小曲面（TPMS）支架生成工具（Scaffolder）与大语言模型连接框架（Model Context Protocol / MCP）融合。其核心目标在于将参数化的极小曲面网格生成转变为**自然语言驱动的生成式设计与属性优化回路（AI-Driven Generative Design & Parameter Tuning Loop）**。

---

## 1. 核心结合场景

### 场景 A：无感参数计算（Natural Language to Parameters）

- **背景**：Scaffolder 生成结构的周期系数 `coff`、等值面高度 `isolevel` 和网格尺寸 `grid_size` 等参数数学定义较深（例如 $coff = \frac{2\pi N}{L}$），用户很难按直觉设准配置。
- **方案**：AI 通过 MCP 获取目标对象的 Bounding Box 物理几何边界 $L$，用户只需要提及"设定网格周期宽度为 5mm"，AI 即可自动计算出精确的 `coff` 数值、设定最适 `grid_size` 偏移，完成自动拟合。

### 场景 B：闭环多物理属性优化驱动（Property-driven Mesh Generation）

- **背景**：工业支架与医疗植入性骨架需要满足预定的空隙率（Porosity）或特定的表面积比（Surface Area Ratio）。
- **方案**：AI 启动一次尝试生成 → 读取 Scaffolder 的计算物理结果 → 判断结果偏大/偏小 → 结合等值面与空隙率的物理关联，迭代微调 `isolevel` 再次生成，直到空隙率落在指定区间（如 $(60 \pm 1)\%$），实现全自动寻优。

---

## 2. 架构设计与改动路径

基于 `blender-mcp` 现有的"通过 TCP 套接字（Socket）连接底层 Blender，并在服务端暴露 Python 命令执行"的链路，整合主要分为三个子工作项：

```
┌──────────────────────────────────────────┐
│              AI 客户端                    │
│  (Claude Desktop / Cursor / VS Code)      │
└──────────────┬───────────────────────────┘
               │  MCP (JSON-RPC over stdio)
               ▼
┌──────────────────────────────────────────┐
│          Blender MCP Server              │
│  server.py  ─  新增 Scaffolder Tools    │
│  - generate_tpms_scaffold               │
│  - get_scaffolder_presets               │
│  - analyze_scaffold_properties          │
└──────────────┬───────────────────────────┘
               │  TCP Socket (localhost:9876)
               ▼
┌──────────────────────────────────────────┐
│             Blender 进程                 │
│  addon.py ─ 执行 Python 脚本            │
│  ├── bpy.ops.object.scaffolder_generate  │
│  ├── PyScaffolder.generate_scaffold(...)│
│  └── 回传 porosity / surface_area 等   │
└──────────────────────────────────────────┘
```

### 核心改动点说明

1. **参数注册与外部读写**
   - `blender/SCAFFOLDER_settings.py` 中声明的状态属性（`props.progress1`、`props.result1` 等）可通过传入脚本直接读写，无需额外包装。

2. **静默生成支持**
   - `blender/SCAFFOLDER_OP_generate_mesh.py` 的 modal 事件监听线程模型与 GUI 耦合较深，需提供一个可以在 MCP 脚本中**阻塞调用**的底层批处理入口，直接驱动 `PyScaffolder.generate_scaffold(v, f, params, callback)` 并捕获其输出。

3. **MCP 端专属工具封装**（在 `blender-mcp` 的 tools 列表中扩展）
   - `get_scaffolder_presets`：返回支持的曲面列表（gyroid, schwarzp, neovius 等）。
   - `generate_scaffold`：传入参数与目标几何网格名，生成极小曲面网格并输出评估数据。
   - `analyze_scaffold`：对已生成的网格重新计算 porosity / surface_area_ratio。

---

## 3. 详细落地步骤（Phases）

### Phase 1：API 兼容与暴露（可实现自动化 Python 控制）

**步骤 1**：在 `blender/utils.py` 中，提供无 UI 环境下的 `PyScaffolder` 依赖静默加载入口，避免 `ImportError` 打断 MCP 脚本执行。

**步骤 2**：在 `blender/SCAFFOLDER_OP_generate_mesh.py` 中，重构生成入口，提取核心生成逻辑为独立函数 `run_scaffolder_headless(target_obj, params)`，允许通过 Python 脚本获取包含 `porosity`、`surface_area`、`surface_area_ratio` 的返回字典，而不依赖窗口管理器 Timer。

```python
# 预期的独立函数签名（Phase 1 目标）
def run_scaffolder_headless(target_obj, params) -> dict:
    """
    在无 GUI 事件循环的环境下直接调用 PyScaffolder，
    返回 {'object_name': str, 'porosity': float,
           'surface_area': float, 'surface_area_ratio': float}
    """
    ...
```

### Phase 2：MCP 工具层配置扩展（Tool Definition）

**步骤 3**：在 `blender-mcp` 的 server 配置文件中（`src/blender_mcp/server.py`），新增 Scaffolder 专有 Tools 定义：

```json
{
  "name": "generate_tpms_scaffold",
  "description": "基于场景中已有的几何模型生成极小曲面支架结构网格，并获取其内部空隙率与表面积指标。",
  "input_schema": {
    "type": "object",
    "properties": {
      "target_object_name": {
        "type": "string",
        "description": "Blender 场景中目标 Mesh 对象的名称"
      },
      "surface_name": {
        "type": "string",
        "enum": ["gyroid", "schwarzp", "schwarzd", "double-gyroid",
                 "double-p", "double-d", "lidinoid", "neovius",
                 "bcc", "schoen_iwp", "tubular_g_ab", "tubular_g_c"],
        "description": "TPMS 曲面类型"
      },
      "unit_cell_size": {
        "type": "number",
        "description": "每个周期晶胞的物理大小（与模型单位一致，通常为毫米）"
      },
      "target_porosity": {
        "type": "number",
        "description": "预期目标孔隙率，介于 0.0 到 1.0 之间。AI 将自动迭代 isolevel 直至收敛"
      },
      "grid_size": {
        "type": "integer",
        "description": "采样网格分辨率，默认为 100，越高越精细但运算更慢"
      }
    },
    "required": ["target_object_name", "surface_name"]
  }
}
```

### Phase 3：AI 参数寻优闭环（Optimization Loop）

**步骤 4**：编写系统提示词模板（Meta Prompt），引导大语言模型在使用该 MCP 工具时遵循以下推理流程：

```
1. 调用 get_scene_info 获取 target_object 的 Bounding Box，记最小维度为 L。
2. 计算初始 coff = 2π × (L / unit_cell_size)。
3. 以 isolevel = 0.0 为起点调用 generate_tpms_scaffold，记录返回的 porosity_actual。
4. 若 |porosity_actual - target_porosity| > 0.01：
      - porosity 偏低（结构太厚） → 增大 isolevel（向正向偏移）
      - porosity 偏高（结构太薄） → 减小 isolevel
   调整幅度使用二分法（bisection），最多迭代 8 次。
5. 收敛后输出最终参数组合与物理评估报告。
```

---

## 4. 验证用例

### 用例 1：基础 API 测试（无 GUI）

在 Blender Python Console 中，不点击任何 UI 按钮，直接运行以下脚本，验证静默生成是否正常工作：

```python
import bpy
import PyScaffolder

target = bpy.data.objects['Cube']
result = run_scaffolder_headless(target, {
    'surface_name': 'gyroid',
    'coff': 3.14159,
    'grid_size': 60
})
print(result)
# 预期输出: {'object_name': 'gyroid_3.142_0_60', 'porosity': 0.497, ...}
```

### 用例 2：AI 闭环调优验证（完整 MCP 链路）

向支持该 MCP 的对话终端发出指令：

> "在场景中的 BonePlant 对象上，生成一个孔隙率精确达到 55% 的 Gyroid TPMS 支架，晶胞尺寸 3mm，并在完成后输出迭代次数与最终参数。"

**验收标准**：
- 生成的模型最终 porosity 落在 $[54\%, 56\%]$ 范围内。
- AI 自动找到最优 `isolevel` 而不需要用户手工介入。
- MCP 对话中输出的迭代记录清晰可读，每轮列出当前 `isolevel` 与实测 `porosity`。

---

## 5. 后续讨论议题（Backlog）

| 优先级 | 议题 | 说明 |
|--------|------|------|
| P0 | `PyScaffolder` 二进制兼容性 | Blender 内嵌 Python 版本与 MCP Server 使用的外部 Python 版本可能不同，需确认 `.pyd` / `.so` 加载路径 |
| P1 | 实时进度回传 | `props.progress1` 在 MCP 执行期间如何异步推送到 AI 客户端（考虑 SSE 或轮询） |
| P1 | 多目标优化 | 同时满足孔隙率与最小管径（`minimum_diameter`）两个约束的联合优化策略 |
| P2 | Lua 自定义曲面支持 | `lua_file` 字段允许传入自定义方程，与 MCP 的文件上传能力结合 |
| P2 | 批量生成与对比报告 | 一次指令生成多种曲面类型的方案，AI 汇总对比孔隙率与表面积，辅助工程师选型 |
