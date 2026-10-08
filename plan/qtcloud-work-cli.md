# 重构方案：qtcloud-work CLI 分层归位

依据同一目录族里 `data/report/evaluation/qtcloud-work-cli.md` 的意图审查，日期 2026-10-08。

## 意图声明

- 核心目标：把 `apps/qtcloud-work/src/cli/` 的依赖按「定位—执行—落盘」落定，让聚合、领域服务、适配各只做自己的事，装载独立于聚合。
- 边界：不改命令面、不改 `--json` 四字段（`ok` / `lines` / `columns` / `rows`）与 `data` 内容、不改报错文字与退出码；不新增领域概念。
- 约束：单文件不超过 250 行；聚合不依赖领域服务、不依赖入口层；装载独立于聚合；只读动作不落盘；改动后 `cargo fmt --check`、`clippy --all-targets --locked -- -D warnings`、`cargo test --locked` 三绿。
- 已知假设：`CONTRIBUTING.md` 与 `docs/dev-guide` 的规矩即作者意图；行为由 `tests/` 与 `examples/` 钉住；本 crate 是子模块，改动要在子模块提交推送后回父仓更指针。
- 不确定性：装载该住哪、`order` 与 `prompts` 朝哪边单向、规则执行归谁、只读命令遇凭证时开不开账本。

## 模块拆分

1. 装载（现 `src/workspace/local.rs`）
   目标：把启动参数与缺省规矩落成四处位置，管账本生命周期（首跑建身份、读回）。
   边界：不定义聚合的字段与校验；位置不进模型。
   接口：`LocalWorkspace::resolve` / `workflows_dir` / `workorders_dir` / `identity_file` / `events_file` / `order_file` / `artifact_path` / `ensure` / `check_flows_dir` / `workspace_id`；游离函数 `root` / `workspace_key` / `account` / `repo_root` / `short` / `write_yaml`。

2. `order` 执行（`src/order/execute.rs`）
   目标：走一步——交 AI、跑 rule 判据、记一笔；人记一笔另走 `record_by_human`。
   边界：不定义判据（在 `criterion`），不定义装载（在 `workspace::local`），不落盘。
   接口：`walk(&mut Order, &Step, &str)`、`record_by_human(&mut Order, &Step, &str)`；把 `audit::check` / `run` / `items_of` 搬到这里，此后只依赖 `criterion` 与装载。

3. `audit` 领域服务（`src/audit/mod.rs`）
   目标：审计资产表与工作区——资产表有而工作区无、工作区有而未登记。
   边界：只此一件，不再承载「跑判据」。
   接口：`audit(root, make) -> Outcome`；撤掉 `Item` / `Kind` / `items_of` 的转出。

4. `prompts` 适配（`src/prompts.rs`）
   目标：给智能体的两段话（走一步、审一遍）。
   边界：只收纯数据 `Facts` 与 `&[Criterion]`，不引 `order` 的类型。
   接口：`prompt_for(&Facts, &[Criterion])`、`judge_prompt(...)`、`criteria_text`；`previous_records` 搬进 `order/ai.rs`，`Facts.records` 仍是拼好的字符串。

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

## 待确认的决策

- [ ] 装载住在哪：接受 `workspace/` 承载装载，把 `CONTRIBUTING.md`、`docs/dev-guide/index.md`、`docs/dev-guide/workspace.md`、`STATUS.md` 从 `locate/` 改到现状，并在 `CONTRIBUTING.md` 注明「`local` 是装载、不属于聚合，聚合件不引 `order`，装载件与 `order` 是互认」（我的倾向，改动最小）；还是把装载立回独立层，恢复「workspace 不引 order」的旧声明。命名归你。
- [ ] 规则执行归谁：把 `check` / `run` / `items_of` 搬进 `order`，`audit` 只留资产审计（我的倾向，符合「领域服务只做一件事」）；还是保留在 `audit`，给「聚合可调服务跑判据」开明文例外。配套要同步 `docs/dev-guide/audit.md` 的「两件相关的事」。
- [ ] `order` 与 `prompts` 的方向：`previous_records` 搬进 `order/ai.rs`，`prompts` 不引 `order`（我的倾向）；还是保留现状。
- [ ] 只读命令开账本：收窄承诺——四个工作区只读动作（`search` / `catalog` / `audit` / `material`）不落盘，凡要打印凭证的读命令（`workflow show`、`order show` / `list`）可开账本，改文档并补测试（我的倾向，与 ADR-0001 的 uuid4 锚定一致）；还是让这些命令真不落盘（须先改凭证方案）。
- [ ] 事件负载：工作流事件与工单一致带全文（我的倾向）；还是保留减配并改注释。
- [ ] `short` 只留一份：留 `workspace::short` 作唯一、`catalog` 改引它（我的倾向）；还是各留一份。
- [ ] `criterion` 自称「判据聚合」与 `CONTRIBUTING.md` 的中立模型不符，命名归你；失效注释 `src/criterion/model.rs:96`（指向已无的 `crate::task`）删除（我的倾向）。
- [ ] 契约测试改扫目录自动收集（我的倾向）；还是维持手维护清单。
