# 新加坡日均 USEP 预测（AE6202 小组项目）

一日超前预测新加坡日均 USEP。六个模型（ETS、SARIMA、TSLM、动态回归、XGBoost、LSTM）加两个基准（前一日价格 Naive、上周同日 SeasonalNaive）。训练期 2021–2024，验证期 2025，测试期 2026-01-01 至 2026-08-31。

样本外的实际预测分两层（Rmd 第 6 节），数据截止 2026-08-31，不使用任何 9 月实际值：

- 6.1 九月一日一日预测：全部模型加两个基准，区间为经验误差分位数。
- 6.2 九月整月 30 天路径：SARIMA 和 ETS 沿用冻结设定、用全部数据重估参数，区间为模型自带区间；XGBoost 只有 9 月 1 日一个点。同一节还有 2025 年和 2026 年的 30 天回测。

## 文件结构

```
USEP_forecasting.Rmd     全部代码：数据处理 → 模型 → 检验 → 表格与图（唯一的代码文件）
data/                    8 个原始 Excel，只读
output/
  model_runs.rds         完整模型运行结果：冻结设置、验证期与测试期逐日预测（缓存，勿删）
  model_forecasts.xlsx   M0–M4 逐日预测、9月1日预测、调参日志、TPC 结果（缓存，勿删）
  ancillary_robustness.rds  辅助服务价格稳健性检验：M2–M4 四种处理的验证期与测试期预测（缓存，勿删）
  processed_data.xlsx    knit 时生成：日度数据与特征
  report_tables.xlsx     knit 时生成：报告用的全部表格
  figures/               knit 时生成：图1–图6
report/                  报告修订稿（红字为修改处）
.vscode/                 VS Code 设置、推荐扩展、一键任务
```

## 两种运行方式

由 Rmd 开头的参数 `run_models` 控制。两种模式下，每次 knit 都会重新生成 `output/processed_data.xlsx`、`output/report_tables.xlsx` 和 `output/figures/`。

| 代码块（章节） | `run_models: false`（默认，几分钟） | `run_models: true`（数小时；另需 xgboost、keras3 和 TensorFlow） |
|---|---|---|
| `packages` 至 `candidates`（第 1–2 节：数据处理、特征、描述统计、平稳性检验、函数定义） | 运行；不加载 xgboost 和 keras3，`rolling_predict()` 不会被调用 | 运行 |
| `tuning`（第 3 节） | 跳过 | 运行：在 2025 年验证期对全部模型调参，写入 `output/model_runs.rds` |
| `test-forecasts`（第 4 节） | 跳过 | 运行：测试期、M0–M4 和 9 月 1 日预测；覆盖 `output/model_runs.rds` 和 `output/model_forecasts.xlsx` |
| `load-models`（第 4.1 节） | 读取 `output/model_runs.rds`（冻结设置、验证期与测试期预测）和 `output/model_forecasts.xlsx`（M0–M4 预测、9 月 1 日预测、调参日志、TPC 结果） | 不读缓存，直接使用刚算出的结果 |
| `ancillary-robustness`（第 5.5 节） | 读取 `output/ancillary_robustness.rds` | 重跑 24 个变体，覆盖 `output/ancillary_robustness.rds` |
| 第 5–9 节的其余代码块（检验、表格、图、9 月预测、导出、sessionInfo） | 运行，基于缓存的预测重算 | 运行，基于新的预测重算 |

- 默认模式不调参，也不做任何滚动预测：除 Naive（前一日价格）外，所有一日点预测（包括 SeasonalNaive）都来自缓存；`insample-residuals` 只是为残差诊断在训练期把四个传统模型各拟合一次。
- 9 月整月预测（第 6.2 节）在两种模式下都重新计算，预测本身只从缓存取 SARIMA 和 ETS 的冻结设定：用全部样本重估后做 9 月路径，并在 30 天回测的每个起点重估一次，共需一到两分钟。
- `run_models: true` 会覆盖三个文件：`output/model_runs.rds`、`output/model_forecasts.xlsx`、`output/ancillary_robustness.rds`。原内容无法恢复，完整重跑前请先备份。
- 缓存预测的核实情况：XGBoost 的缓存预测已用本代码重跑核实，结果完全一致；LSTM 的缓存预测来自此前一次运行，模型代码相同，但未在本环境重跑，因 TensorFlow 版本和硬件不同，重跑可能有小幅差异。
- 软件版本见 knit 输出文末的 `sessionInfo()`；`run_models: true` 时另外打印 TensorFlow 和 Keras 的版本。

## 在 VS Code 里像 RStudio 一样运行 R（macOS，一次性设置）

1. 安装 R：<https://cloud.r-project.org/>。再安装 pandoc（`brew install pandoc`，或从 <https://pandoc.org/installing.html> 下载）。RStudio 自带 pandoc，VS Code 没有，缺了它就不能 knit。
2. 用 VS Code 打开本文件夹，右下角提示安装推荐扩展时点 Install。推荐的扩展有两个：R（REditorSupport）和 Claude Code。
3. 菜单 Terminal → Run Task → **R: install packages**，会安装 rmarkdown、fpp3、forecast、tseries、readxl、writexl、languageserver 和 httpgd。只有完整重跑才需要另外装：`install.packages(c("xgboost", "keras3")); keras3::install_keras(backend = "tensorflow")`。
4. 日常使用：
   - 打开 `USEP_forecasting.Rmd`，每个代码块上方有 **Run Chunk / Run Above** 按钮。第一次运行会自动开一个 R 终端，相当于 RStudio 的 Console。
   - `Cmd+Enter` 运行当前行或选中的代码。
   - 图显示在右侧的 plot 窗口（httpgd），相当于 RStudio 的 Plots。左侧栏 R 图标里的 Workspace 相当于 Environment，可以查看变量。
   - 生成 HTML 报告：`Cmd+Shift+B`（默认任务 **R: knit (saved model results)**），或点编辑器右上角的 Knit 按钮。

## 让 Claude 直接改代码

- **VS Code 里的 Claude Code 扩展**：需要 VS Code 1.94 以上和付费 Claude 订阅。安装后打开任意文件，点编辑器右上角的 Spark 图标，第一次使用要登录。改代码时以 diff 显示，由你接受或拒绝；在终端运行 `Rscript` 检查代码前会先征求你同意。
- **Claude 桌面 App**：把这个文件夹连接到对话后，Claude 可以直接修改这里的文件，VS Code 中会同步显示改动。

## 尚未完成（来自修订稿附录 E）

- 对数变换稳健性检验：ETS、SARIMA 加 `lambda = 0, biasadj = TRUE`。
- TPC 实际触发指示变量：需要 RUSEP 数据。
- 修订稿中带 * 的数值是用 Python 复算的。knit 之后，以 `output/report_tables.xlsx` 中的 R 结果为准替换。
