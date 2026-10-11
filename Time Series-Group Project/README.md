# 新加坡日均 USEP 预测（AE6202 小组项目）

一日超前预测新加坡日均 USEP。六个模型（ETS、SARIMA、TSLM、动态回归、XGBoost、LSTM）加两个基准（前一日价格 Naive、上周同日 SeasonalNaive）。训练期 2021–2024，验证期 2025，测试期 2026-01-01 至 2026-08-31。

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
  figures/               knit 时生成：图1–图5
report/                  报告修订稿（红字为修改处）
.vscode/                 VS Code 设置、推荐扩展、一键任务
```

## 两种运行方式

| 设置 | 做什么 | 时间 |
|---|---|---|
| `run_models: false`（默认） | 读取 output/ 里已保存的模型预测，重算全部检验、表格和图 | 几分钟 |
| `run_models: true` | 从头调参并重跑六个模型，覆盖三个缓存文件 | 数小时；另需 xgboost、keras3 和 TensorFlow |

保存的完整运行环境：R 4.6.1、TensorFlow 2.21.0、Keras 3.15.1。

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

- 第 6 节实际预测：需要下载 2026 年 9 月五个站点的日温度，把 `end_date` 改为 2026-09-30 后重跑，再做 10 月预测。
- 对数变换稳健性检验：ETS、SARIMA 加 `lambda = 0, biasadj = TRUE`。
- TPC 实际触发指示变量：需要 RUSEP 数据。
- 修订稿中带 * 的数值是用 Python 复算的。knit 之后，以 `output/report_tables.xlsx` 中的 R 结果为准替换。
