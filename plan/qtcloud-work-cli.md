# 重构方案：qtcloud-work CLI 分层归位

依据同一目录族里 `data/report/evaluation/qtcloud-work-cli.md` 的意图审查，日期 2026-10-08。

## 意图声明

- 核心目标：把 `apps/qtcloud-work/src/cli/` 的依赖按「定位—执行—落盘」落定，让聚合、领域服务、适配各只做自己的事。
- 边界：不改命令面、不改 `--json` 四字段（`ok` / `lines` / `columns` / `rows`）与 `data` 内容、不改报错文字与退出码；不新增领域概念。
- 约束：单文件不超过 250 行；聚合不依赖领域服务、不依赖入口层；只读动作不落盘；改动后 `cargo fmt --check`、`clippy --all-targets --locked -- -D warnings`、`cargo test --locked` 三绿。
- 已知假设：`CONTRIBUTING.md` 与 `docs/dev-guide` 的规矩即作者意图；行为由 `tests/` 与 `examples/` 钉住；本 crate 是子模块，改动要在子模块提交推送后回父仓更指针；`workspace` 是 `workorder` 的上级容器，容器与内件互认是常态。
- 不确定性：`order` 与 `prompts` 怎么彻底脱钩、规则执行归谁、只读命令遇凭证时开不开账本。

## 模块拆分

1. 装载（现 `src/workspace/local.rs`）
   目标：把启动参数与缺省规矩落成四处位置，管账本生命周期（首跑建身份、读回）。
   边界：不定义聚合的字段与校验；位置不进模型。与 `order` 的互认是容器与内件的常态（`LocalWorkspace::order_file` 收 `&WorkOrder`），不算越界。
   接口：`LocalWorkspace::resolve` / `workflows_dir` / `workorders_dir` / `identity_file` / `events_file` / `order_file` / `artifact_path` / `ensure` / `check_flows_dir` / `workspace_id`；游离函数 `root` / `workspace_key` / `account` / `repo_root` / `short` / `write_yaml`。
   一并要改：`CONTRIBUTING.md`、`docs/dev-guide/index.md`、`docs/dev-guide/workspace.md`、`STATUS.md` 仍写 `locate/`，改成 `workspace` 容器事实。

2. `order` 执行（`src/order/execute.rs`）
   目标：走一步——跑 rule 判据、记一笔；人记一笔另走 `record_by_human`。
   边界：不定义判据（在 `criterion`），不定义装载（在 `workspace::local`），不落盘，不引 `prompts`。
   接口：`walk(&mut Order, &Step, &str)`、`record_by_human(&mut Order, &Step, &str)`；把 `audit::check` / `run` / `items_of` 搬到这里，此后只依赖 `criterion` 与装载。

3. `audit` 领域服务（`src/audit/mod.rs`）
   目标：审计资产表与工作区——资产表有而工作区无、工作区有而未登记。
   边界：只此一件，不再承载「跑判据」。
   接口：`audit(root, make) -> Outcome`；撤掉 `Item` / `Kind` / `items_of` 的转出。

4. 话术与调 AI（`src/prompts.rs` 与 `src/order/ai.rs` 的调 AI 那半）
   目标：给智能体的两段话，以及造话术、调 `pi`、审这一趟。
   边界：只收纯数据（`Facts` 与 `&[Criterion]`），不引 `order` 的类型；`order` 也不得引它。
   接口：`prompt_for(&Facts, &[Criterion])`、`judge_prompt(...)`、`criteria_text`；`previous_records` 移出 `prompts`（收纯字符串或由调用方拼）；调 AI 那半搬来适配层，落哪个文件由用户定。

5. 路径显示 `short`
   目标：相对工作区根写短、不在根下原样，全库一份。
   边界：纯字符串处理，不碰文件系统。
   接口：`short(root, path) -> String`；`catalog` 改引这一份（现 `src/catalog/mod.rs:84` 另有一份，实现不同）。

6. 事件负载（`src/workflow/events.rs`）
   目标：每个聚合的事件带自己的全文与凭证。
   边界：公共三字段（`event` / `at` / `workspace_id`）由 `crate::events` 补，负载形状由聚合定。
   接口：工作流的 `created` 改用 `crate::events::yaml_to_json(&payload)`，与工单事件一致。

7. 契约测试（`tests/contract.rs`）
   目标：新增源码自动纳入「不依赖入口层」检查，不再靠手维护清单。
   边界：只扫 `src/`，排除入口件；排除名单的边界由用户定。
   接口：读目录收集 `.rs` 列表，逐件断言不含 `crate::cli`。

## 定义冻结守卫（方案）

2026-10-09 加。规格 `workflow.md` 已定两条：创建工作流撞上区内同名即拒绝，不覆盖；定义一经工作任务引用即不可变。两条现在都没守，而且不是「未校验」而是「静默改掉」——实测：建一条两步工作流「试一条」，开单引用它，再同名跟一条 `workflow create 试一条 --steps 丙`，定义被直接盖掉，工单照常打开（`workflow_id` 不变，凭证锚在名上），进度从 `0/2 下一步：甲` 变成 `0/1 下一步：丙`。

背景。写作面只有 `workflow create` 与 `workflow import` 两个口：`import` 撞同名即拒，`create` 不查 `exists()` 直接截断写。文件就在手边，绕过 CLI 直改 YAML 也拦不住——所以守卫要分成「挡 CLI 这两个口」与「抓文件这一口」两段。

选项。一、只补 `create` 的撞名检查：挡得住 CLI，挡不住手改。二、加装载核对，用流水当锚：工作记录的 `step_id` 派生自「工作流凭证 + 步骤名」，删步骤、改步骤名都会让旧流水的 `step_id` 对不上当前定义；次序则用「流水里的站名序列必须是定义次序的子序列」核——手改文件因此会在读的时候被抓住，不必另立登记处。三、再加内容摘要：工单封面存所引定义的内容指纹，装载时比对——能连「删文件再同名重建、步骤名不变而描述变了」也抓住，代价是改封面结构与事件负载，且要与工具箱、studio 两侧对表同步。

影响。选项一只堵住已知的那条路，事后仍会有人从文件侧进来；选项二把成本压在读路径（打开工单多一次比对，不扫目录），抓得住删改与次序，抓不住「同名重建且步骤名一字不差」；选项三最严，但它动的是跨端契约，不是本地守卫。

建议先做选项二，顺手带上选项一：

1. `workflow::create` 照 `import` 的写法先查 `flow.exists()`，撞同名即拒；
2. 打开工单时逐条核 `step_id`，与当前定义按名派生的凭证对不上就报「定义与流水失配」，不静默算成没走过；
3. 核次序：流水里的站名序列须是当前定义次序的子序列，乱序即报失配；
4. 三条各配一个测试，并同步 `docs/dev-guide/work-order.md` 的一条纪律。

内容摘要（选项三）单列一项待议：它要改工单封面字段与事件负载，跨端对表也得跟，等三端都稳了再说。

## 决策落地情况（2026-10-09 复核）

- 已落地：`order` 与 `prompts` 已脱钩——造话术、调 `pi`、审移进 `workers/` 与 `adapters/`，`prompts` 只认 `criterion`，`order` 一侧已无引用。
- 已落地：判据的执行搬进 `order/rules.rs`（依赖 `criterion`），`audit` 只留资产审计；原先 `order/execute.rs` 引 `crate::audit` 已除。
- 已落地（按 ADR-0001 收窄承诺）：四个只读动作（`search` / `catalog` / `audit` / `material`）不开账本，要打或核对凭证的读命令（`workflow show`、`order list` / `show`）开账本；文档与 `tests/run_context.rs` 都跟上。
- 已落地：工作流事件与工单事件都带定义全文（工作流那条含派生出的步骤凭证与判据字段）。
- 已落地：`short` 全库只留 `workspace::short` 一份，`catalog` 改引它。
- 部分落地：失效注释已删；模块首行的「判据聚合」还挂着——按 `CONTRIBUTING.md` 它归中立模型（无身份、无生命周期），`Step` 本就持有 `criteria` 字段，所以只该改这一句，不动目录（`order` / `prompts` / `artifact` / `workers` 都在 workflow 之外引它，挪进 `step` 会把单向依赖改成两条）。命名归你，等你点头。
- 已落地：`tests/contract.rs` 改扫 `src/` 自动收集，新增文件即入检。但它只做字符串匹配 `!text.contains("crate::cli")`，只守「不引入口层」一条，不守「聚合不引服务」；且匹配的是文本，注释里出现也会红。
