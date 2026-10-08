# 意图审查：qtcloud-work CLI（apps/qtcloud-work/src/cli/）

审查日 2026-10-08，crate `qtcloud-work-cli` 0.1.0-beta.2。

## 结论

最可能的原始意图是把一次知识工作拆成「定位—执行—落盘」三段，并让每件事只住一处。整体偏移中等：主流程成立，但装载的落位一次反复把已经拆掉的聚合环重新带回，仓库文档未跟；最该先处理的是定下装载落位（它决定 `order ↔ workspace` 算不算环、文档怎么写），随后把「读路径不写盘」这条纪律在代码与测试上落实。

## 意图假设

- 核心目标：一条命令进来，先定位（工作区根 / 账本 / 工作流目录 / 产物落点），再交给管这件事的那件处理，结果包成统一信封印出。
- 边界：管内容的不碰盘、碰盘的不定内容；跨聚合只做一件事的单独放；接口与外边界归适配层；位置不进模型，工单文件里不写路径。
- 约束：单文件不超过 250 行；聚合不依赖领域服务，聚合与服务不依赖入口层；只读动作不落盘。
- 已知假设：`CONTRIBUTING.md`、`docs/dev-guide`、`README.md` 里声明的规矩即作者意图，代码应照着它走。
- 不确定性：`locate` 并回 `workspace` 是「装载不算聚合、住在 workspace 目录里」的重新落位，还是把 `workspace ↔ order` 的环带了回来——作者未表态，见偏移 2。

## 偏移清单（按严重度排序）

1. [严重] 只读命令会开账本，「读路径不写盘」被证伪 — src/workspace/local.rs:146
   证据：`workflow show` 调 `WorkflowFile::credentialed()`（src/workflow/mod.rs:78），它调 `workspace_id()`（src/workspace/local.rs:146），后者调 `ensure()`（src/workspace/local.rs:111）建目录、写 `workspace.yaml`、发 `WorkspaceCreated`。实测：`--workflows` 指一个外部目录、`--data` 指一个空目录，跑 `workflow show 试` 后空目录里出现 `workspace.yaml`、`events.jsonl`、`workorders/`。测试 `tests/run_context.rs:87` 只跑空账本上的 `order list`，触发不到这条路径，空过。
   与意图的关系：与 `src/workspace/local.rs:145`「读路径不会触发首跑写盘」及 `docs/dev-guide/workspace.md`「只读动作不开账本，不在任何根上建文件」矛盾。

2. [严重] 装载并回 workspace，order 与 workspace 重新互相引用 — src/order/mod.rs:29
   证据：`order` 引 `crate::workspace::LocalWorkspace`，`workspace` 引 `crate::order::WorkOrder`（src/workspace/local.rs:23）。`5aeaf54` 曾把装载搬进 `locate/`，专为消 `workspace ↔ order` 这条环；`4457a2e`、`79d0204` 又并回 `workspace/`。`CONTRIBUTING.md`、`docs/dev-guide/index.md`、`docs/dev-guide/workspace.md` 通篇仍是 `locate/`，并断言「workspace 一件也不引 order」，`STATUS.md` 也按 `locate` 记账。
   与意图的关系：与「装载独立、聚合之间不转圈」矛盾；也可能是重新落位而文档未更，需与作者确认。

3. [中] 聚合依赖领域服务 order → audit — src/order/execute.rs:47
   证据：`:47`、`:48`、`:136`、`:137` 调 `crate::audit::items_of` 与 `crate::audit::run`；`items_of` 只是 `criterion` 的转出（src/audit/mod.rs:23）。`tests/contract.rs` 只钉入口方向，无断言拦。
   与意图的关系：与「聚合不得依赖服务」矛盾；规则执行做成服务、聚合又要调它，属设计张力。

4. [中] 聚合与适配互相引用 order ↔ prompts — src/order/ai.rs:5
   证据：`order/ai.rs` 用 `crate::prompts` 的 `Facts` / `prompt_for` / `judge_prompt` / `previous_records`（`:5`、`:12`、`:43`、`:157`）；`src/prompts.rs:94` 又用 `crate::order::WorkRecord`。适配与聚合的依赖方向无断言。
   与意图的关系：与「聚合件不与适配件同层」矛盾。

5. [其他] 两处 `short` 各写一份，实现还不一样 — src/workspace/local.rs:214
   证据：`catalog` 另有一份 `short`（src/catalog/mod.rs:84，用于 `:78`、`:204`）；`workspace` 按字符串前缀剥，`catalog` 按 `Path::strip_prefix` 剥。
   与意图的关系：无依据的重复，与「每件事只住一处」相左。

6. [其他] 事件负载两套形状 — src/workflow/events.rs:26
   证据：工单事件把整份 YAML 转 JSON 带上（`src/order/events.rs`），工作流事件的 `to_json` 只留每条判据的 `executor` 与 `description`，`path` / `absent` / `file` / `contains` / `run` 丢弃，注释却写「带声明全文与派生出的凭证」。
   与意图的关系：同一意图两处做法与说法不一致。

7. [其他] 失效注释 — src/criterion/model.rs:96
   证据：指向 `crate::task::execute`，全库已无 `crate::task`。
   与意图的关系：无依据，属重构残留。

8. [其他] `criterion` 自称聚合，与分类不符 — src/criterion/mod.rs:1
   证据：模块注释写「判据聚合」；`CONTRIBUTING.md` 把 `criterion` 列为中立领域模型（无身份、无生命周期）。
   与意图的关系：名与类不符，命名归用户。

9. [其他] 契约测试清单重复且手维护 — tests/contract.rs:169
   证据：`workspace/events.rs`、`workspace/mod.rs`、`workspace/model.rs` 各重复一次（42 个唯一项列成 45 条）；新增文件不会自动纳入检查。
   与意图的关系：规则覆盖范围收窄，与「有测试钉住」的声明不符。

## 建议

先与作者确认两条落位：装载住在 `workspace/` 里算不算环，`order ↔ prompts` 该朝哪边单向。定下后把 `CONTRIBUTING.md`、`docs/dev-guide`、`STATUS.md` 从 `locate/` 改成现状，这是偏移 2 的收尾。

接着处理偏移 1：要么让凭证派生不再落盘地取工作区 id，要么把「只读动作不开账本」改成事实，并补一条 `workflow show` 的只读测试，别只测空账本上的 `order list`。

偏移 3、4 属设计取舍：给「聚合可调某些服务」开口子并把方向写进规范，或者把规则执行与话术各自挪到能被单向依赖的层。

偏移 5 到 9 为清理项：合并两处 `short`、统一事件负载形状、删失效注释、把 `criterion` 的名与类对齐、把契约测试清单改成扫目录自动收集。命名部分由用户定去留。
