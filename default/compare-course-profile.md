# 报告：compare-course-profile

## 定位

个人草稿在 `iGuo/materials/course/`：3 篇 Markdown（`index.md` 量潮课堂目录、`knowledge-work.md` 知识工作、`production-internship.md` 生产实习），单文件口语叙述，无课时编号、无 JSON、无验收标准。

课程研发档案在 `../../../quanttide-course/data/profile/`，自述为课程内容的唯一创作源头：每课时一目录（`lessonN-<slug>/`，`index.md` 与 `index.json` 成对，场景文件 `0X-<scene>.*`）。

## 比对

**口径**：草稿以「量潮课堂」为验证组织的产品构思——用体系教体系，课程是验证产出质量是否可见提升的手段；档案以生产实习为中心的课程内容源头，口径是教学交付。

**重叠**：只在「生产实习」这个课名上。草稿给的是制度与验证思路（指导边界、壳即 baseline、观察不纠正）；档案的 production-internship 只有 `lesson1-second-brain`（用第二大脑开工作区、逐级提交），README 还是占位。知识工作在档案里没有对应课程；vibe-coding 档案三课时齐全（zed / feishu / github），草稿未提及；data-engineering 档案只有需求（理论 24 学时、实验 8 学时），无课时，草稿也未提及。内容层几乎不交叠。

**缺口**：草稿缺课时结构与编号、无验收标准（全是散文式理念）、无 JSON 不能直接上架，且只挂了两门课。档案缺 data-engineering 课时；生产实习只到课时 1，草稿里的「指导边界制度」「壳 baseline」没有落成课时或验收标准；没有「量潮课堂」这一层验证设计；vibe-coding 的 README 只有标题；AGENTS.md 索引里的 `index.md`（研发策略）实际不存在，链接悬空。

**格式差**：草稿是扁平的中文单文件，只有 Markdown；档案每课程一目录、每课时 `lessonN-<slug>/`，目录名英文小写连字符，`index.md` 与 `index.json` 成对，场景另有 md/json（vibe-coding 还带 mp4 与 html DAG 视图），结构里有场景表、mermaid DAG 与 `acceptance{criteria, method, on_fail}`。档案自己的 `CONTRIBUTING.md` 仍写旧结构（`lesson1.json` + `sense1.md`），与实际目录不一致，是一处待还的格式债。

## 结论

课程研发档案是课程内容的唯一创作源头；个人草稿是概念与制度层，不整篇搬过去，只把它能课程化的部分回流，构思与验证设计留在个人档案。

**回流**：「知识工作」方法论课（自举 / 打通 / 验证三层、作业即 PR）——在档案里新开一门 `knowledge-work/`，或并入生产实习作为方法论总纲；先写课程 README 与一课时 `lesson1-*` 的 `index.md`、`index.json`，别只留理念。生产实习的「壳 baseline」观察法——写进课时的 acceptance：验收通不过先修标准，不纠正学生。指导边界（群体免费 / 个体付费、反馈发生在提交时点）——收进 `production-internship/README.md` 的三两句话。

**不并**：「量潮课堂」验证组织本身——验证目标、风险、回报机制，属验证组织与主体层的设计，不是课程内容。

**不动**：`vibe-coding/` 三课时、`data-engineering/index.md` 需求、`production-internship/lesson1-second-brain`。

**顺手还的格式债**：`CONTRIBUTING.md` 的旧结构与实际目录不一致；`AGENTS.md` 里 `index.md` 链接悬空。

**待创始人拍板**：「知识工作」新开一门课还是并入生产实习作总纲；指导边界只作课程边界口径还是另立主体层文档；「量潮课堂」验证设计留在个人档案还是另开一份产品与验证文档。

## 执行记录

- ✓ 2026-09-10 22:31　开工　对比个人课程草稿与课程研发档案
- ✓ 22:32　locate　两边档案找齐，两条判据命中
- ✓ 22:32　compare　「比对」一节写进本报告
- ✓ 22:33　conclude　「结论」一节写进本报告，等创始人点头

## 闸门项

- ⧗ 创始人点头（回流与并法怎么定）（留给人 / 待判）
