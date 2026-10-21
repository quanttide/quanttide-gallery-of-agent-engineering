# 把技能里的三类内容分层

本案例记录一次 Hermes 技能整理：把一个装混了「判据、方法、一次性评审结果、未决问题」的技能拆回只剩判据与指针，把一次性结果推进报告仓、未决项移出正文。案例沉淀为八阶段流水线与五条不变式，并用逐行标性质的办法修正了「按小节估可删量」的误判。

## 需求

### 情境

Hermes 技能是给智能体按触发加载的判据集，正文越长，每次触发付的加载成本越高。技能用久了会自然演化成收件箱：新踩的坑往下追加一句，一次评审的结论整段贴进来，还没定下来的问题也顺手写进去。本次对象是一个规格写作技能（105 行 / 4,921 字符），此前被使用 14 次、打补丁 10 次。

三条背景约束：

- 技能只装「动手那一刻必须做出的判断」，其余按成熟度下沉（脚本／references／改环境默认值）。
- 报告与复盘属陈述型资产，家在该领域的报告仓，不在技能里。
- 报告仓是子模块，写进去要走资产仓 → 领域仓 → 根仓三层收指针。

### 难点

- **三种寿命混在一份正文里**：永久成立的判据、反推的方法、某轮评审的一次性结果、还没定的问题，看起来都是「有用的记录」。
- **过期方式不对称**：判据过期会自己露馅（照它做不出来），一次性结果与未决项过期后**伪装成事实**——第 24 行写着「用户仍在纠结」，读起来像档案，实际是一个未回答问题。
- **仓里查无正本**：那七条评审结论在全仓 grep 零命中，只活在技能正文里；直接删就真丢了。
- **跨技能互不指路**：同一领域 5 个写作技能各自正文只出现自己的名字，选错一次就白读 5k 字符。
- **可删量估不准**：按小节看，标称 25 行「一次性结果」，实际逐行标完只有 11 行是结果。

## 意图

### 自然语言规则

1. 正文只放判据：动手那一刻必须做出的判断才进正文，其余下沉。
2. 按行标性质、不按小节估：把候选内容逐行标成 判据／方法／一次性结果／未决项，只搬后两类。
3. 未决项出正文：正文只留一行指针，未决问题与「已成立的部分」分开记，并写明下次先问。
4. 一次性结果入仓且不留副本：结论搬进该领域报告仓后，技能正文只留方法与分级口径。
5. 数字不编：定量对比缺实测就留分母口径与「待实测」，不写估值。
6. 同类技能互相指一次路：各自写一句最能判的分工，不复述对方判据。
7. 先删双写再谈瘦身：同一判据在别的技能里已有正本的，本处改指针。
8. 以状态为证：技能可读、报告仓远端有该文件、三层指针各层 HEAD = 远端 main，才算完成。

### 形式化规则

```text
对象：S 技能正文，R 领域报告仓，K 同类技能集合，C 候选内容

操作
  classify(C) → {判据, 方法, 一次性结果, 未决项}   逐行标性质
  sink(一次性结果) → R                            搬进报告仓，写清对象与口径
  point(未决项) → references/open-questions.md     出正文，留一行指针
  dedup(S, K) → 指针                               已有正本的改指针
  crossref(K) → 每技能一句分工
  chain(R) → 资产仓 → 领域仓 → 根仓                 逐层收指针

不变式（完成后必须全部成立）
  I1 判据唯一：任一判据只在正文出现一次，重复处已改指针
  I2 事实不冒充：未决项不在正文，正文不出现「仍在纠结」这类状态句
  I3 正本入库：一次性结果在报告仓有正本，技能正文不含其全文
  I4 数字可核：定量项要么给实测值与分母，要么标明待实测
  I5 指针收敛：资产仓与各层父仓 HEAD = 远端 main
```

## 规格

八阶段流水线，每阶段以状态断言收尾：

1. **备份**：技能目录整份复制到备份目录；断言 `diff -r` 与原目录无差异。
2. **标性质**：正文逐行标 判据／方法／一次性结果／未决项；断言每行都有归类，且「一次性结果」与「未决项」的行号清单固定下来。
3. **查正本**：在仓里 grep 待搬内容的特征句；断言得出「仓里有无正本」这个事实（有则直接改指针，不新建）。
4. **入仓**：报告仓内按性质选 `evaluation`／`plan`／`decision` 落文，按该仓体例写（主线一句 + 三段 + 分级 + 怎么核）；断言文件已提交推送。
5. **登记**：报告仓 `README.md` 概述列出该文；断言概述不再出现「待补充」。
6. **改技能**：未决项搬 references 并留指针、一次性结果改指向仓、双写改指针、同类技能加分工一句；断言正文不再含被搬内容的特征句。
7. **收指针**：报告仓 → 领域仓 → 根仓，逐层提交推送；断言各层 `git ls-remote` 与本地 `rev-parse` 相等。
8. **复核**：`check-skills.py` 判体量、`check-refs.py` 判孤儿与断章、`skill_view` 抽读一次；断言三项无新增问题。

### YAML 描述

```yaml
meta:
  skill: <技能路径>
  domain: <所属领域>
  report_repo: <该领域报告仓>
  report_path: <报告落点>
  backup: <备份目录>

steps:
  - id: backup
    action: 整份复制技能目录
    assert: diff -r 与原目录无差异
  - id: classify
    action: 逐行标 判据/方法/一次性结果/未决项
    assert: 每行有归类，后两类行号清单固定
  - id: check-original
    action: 在仓里 grep 待搬内容的特征句
    assert: 得出仓里有无正本这一事实
  - id: sink
    action: 按报告仓体例写入（主线+三段+分级+怎么核）
    commands:
      - git add <report_path> README.md && git commit
      - git push <ssh-url> HEAD:main
    assert: 远端 main 含该文件
  - id: register
    action: 报告仓 README 概述列出该文
    assert: 概述不含「待补充」
  - id: rewrite-skill
    action: 未决项出正文、一次性结果改指向、双写改指针、同类技能加分工
    assert: 正文不含被搬内容的特征句
  - id: pointers
    action: 报告仓 → 领域仓 → 根仓逐层收指针
    assert: 各层 git ls-remote == 本地 rev-parse
  - id: audit
    action: 跑 check-skills.py / check-refs.py / skill_view
    assert: 无新增超线、孤儿、断章
```

> 提示：最容易白干的一处是**按小节估可删量**。同一节里往往「怎么反推、怎么分级、被说看不懂怎么办」全是判据，和那几条一次性的结论只隔一个列表；按段落标题算会高估两三倍，删下去就把方法一起删掉。先逐行标性质，再动手。

## 实现

可跑的骨架（`<...>` 为占位）：

```bash
# 0) 备份
cp -a "$SKILL" "$BACKUP/$(basename "$SKILL")"

# 1) 待搬内容的特征句在仓里有没有正本（有则只改指针，不新建）
grep -rn "$FEATURE_SENTENCE" "$REPO_ROOT" || echo "仓里无正本，需入仓"

# 2) 入仓并推送（报告仓）
cd "$REPORT_REPO"
git add "$REPORT_PATH" README.md
git commit -q -m "docs: 评估——<一句话>"
git push "$SSH_URL" HEAD:main
test "$(git ls-remote "$SSH_URL" refs/heads/main | cut -f1)" = "$(git rev-parse HEAD)" && echo "报告仓对齐"

# 3) 逐层收指针（领域仓 → 根仓），每层只暂存自己那一项
for R in "$DOMAIN_REPO" "$ROOT_REPO"; do
  cd "$R" && git add "$POINTER_PATH" \
    && git diff --cached --name-only \
    && git commit -q -m "chore: 同步 <子模块> 指针（<说明>）" \
    && git push origin HEAD:main \
    && test "$(git ls-remote origin refs/heads/main | cut -f1)" = "$(git rev-parse HEAD)" && echo "$R 对齐"
done

# 4) 复核
python3 "$AUTHORING/scripts/check-skills.py" --root "$SKILL_ROOT" | grep -c "$(basename "$SKILL")"   # 0 = 不在超线名单
python3 "$AUTHORING/scripts/check-refs.py" --root "$SKILL_ROOT" | grep -A3 "$(basename "$SKILL")"    # 无孤儿/断章
```

案例执行结果：技能正文 105 行 / 4,921 字符 → 102 行 / 4,805 字符（预估算的是 65 行，按小节估高估了）；七条评审结论落进该领域报告仓的 `evaluation/work-spec-drift.md`（含必改三条、要谈四条、四组待实测分母），三层指针 `b46e67b` → `2a1fc2a` → `49484b4` 逐层核过 sha；新增 `references/open-questions.md` 一份；同领域 5 个写作技能各加一句分工；`check-skills.py` 与 `check-refs.py` 对本技能无新增问题。
