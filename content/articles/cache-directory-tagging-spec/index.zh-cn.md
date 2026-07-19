---
title: "保护备份的 43 字节约定:缓存目录标记实战"
date: 2026-07-19T20:25:08Z
draft: false
tags: ["Backup", "CACHEDIR.TAG", "互操作性"]
categories: ["技术实践"]
author: "X 实验室"
description: "Cache Directory Tagging Specification (CACHEDIR.TAG) 的工程实战拆解:43 字节签名、应用语义、tar/Borg/restic 三家采纳差异,以及盲目排除的安全代价。"
summary: "一个 43 字节的魔数头部 —— Signature: 8a477f597d28d172789f06886806bc55 —— 让应用把自己的缓存目录标记出来,tar、Borg、restic 默认跳过;本文拆解规范的确切语义、各采纳方差异、以及让盲目排除变得危险的那个安全权衡。"
toc: true
---

## 背景与动机

2004 年 Bryan Ford 提出了一个故意微小的约定,目的在于解决困扰备份软件作者多年的一个问题:各种应用都会暗中创建**缓存目录** —— 浏览器的网页缓存、缩略图树、Gradle 的 `~/.gradle/caches`、Mozilla 的 `~/.mozilla/.../Cache`、ccache 的编译产物仓。这类目录的共同性质是:它们没有长期归档价值、动辄膨胀到数 GB、每次浏览都会变动(从而毁掉增量备份的差量算法)、文件名是基于哈希的乱码(用户在浏览自己 home 目录时看见一堆怪异名字会被吓到)([Ford 2004](https://bford.info/cachedir/))。当年的位置标准朝着两个互斥方向拉扯:*Filesystem Hierarchy Standard* 规定的是系统级 `/var/cache`(只 root 可写,对每用户应用无用),*XDG Base Directory* 推荐 `$XDG_CACHE_HOME` —— 但每个用户都可能在登录脚本里改掉这个环境变量,于是备份软件在 `$HOME` 下扫描时无法在不同机器间可靠地枚举每用户缓存。

Ford 的提案绕开了争论位置这场死结:与其争论**缓存该放哪儿**,不如约定**一个目录如何声明自己是缓存**。结果就是 **Cache Directory Tagging Specification**([`bford.info/cachedir/`](https://bford.info/cachedir/)) —— 4 行规则,20 年后仍被 GNU tar、BorgBackup、restic、Gradle、ccache、Git 对象缓存以及多款浏览器普遍遵守。规范小到能写在一张便利贴上 —— 也正因为如此,它在那些宏大而权威的"把缓存放在这里"标准都失败的地方取得了成功。

本文拆解这条约定背后的工程: `CACHEDIR.TAG` 的精确字节布局、目录级语义(标记**最顶层**的缓存目录、标签覆盖整个子树)、各采纳方的行为差异(restic 排除的是**内容**但保留 `CACHEDIR.TAG` 本身;tar 提供三种粒度的排除),以及那条颇为重要却少被讨论的安全权衡 —— 备份软件必须**校验**那 43 字节的头部才能信任标签,否则攻击者随手扔一个 `CACHEDIR.TAG` 就能让你的备份对任意数据视而不见。

## 前置条件

- 类 Unix 环境,装 `tar` 1.28+(支持 `--exclude-caches-all`),或装 Go 生态里 restic,或用 Python 3 装 Borg(`pip install borgbackup`)。
- 一个含至少一处缓存形子目录的测试树,**第一次别在你 home 目录上跑破坏性变体**。
- 可选:`xxd` 或者任一 hex editor 用来检视 43 字节头。
- 读权限访问规范主页 [`https://bford.info/cachedir/`](https://bford.info/cachedir/)(版本 0.6,自 2004 年起未变)。Bryan Ford 个人网站上挂着维护副本,`Changes` 段落跟踪历年叉改。

## 步骤一:准备工作

先想清楚要标记的对象。规范对"缓存目录"这个名词定义得很硬:它的**全部内容(包括所有子目录,递归)** 应当能从位于别处的源数据**重生成**([Ford 2004,"Application Semantics"](https://bford.info/cachedir/))。如果该目录下任何一件产物是**原始**输出 —— 一个 Git 仓的工作文件、一棵 `~/Documents`、一份数据库 dump —— 那它就**不是**缓存,不能打标签。把目录打错标签正是那 43 字节签名要防的故障模式。

搭一个 scratch 树:

```bash
mkdir -p ~/taggedir-demo/{real-data,~/cache/gradle-build,~/cache/thumbs}
echo "primary content"          > ~/taggedir-demo/real-data/notes.txt
echo "blob A"                   > ~/taggedir-demo/cache/gradle-build/aux.json
echo "thumbnail bytes go here" > ~/taggedir-demo/cache/thumbs/cover.png.json

# 先确认一切可见
find ~/taggedir-demo -type f | sort
```

挑一个丢了也不痛的目标。本演练中 `cache/` 子树是待牺牲的缓存;`real-data/` 是备份绝不许丢的对象。同一组 `tar`/`borg`/`restic` 命令你要跑两遍 —— 第一次不打标签(全部进入归档),第二次在 `cache/` 放一个 `CACHEDIR.TAG`(缓存从归档消失但 `real-data/` 原样幸存)。

## 步骤二:核心实现

### 2.1 写标签

规范只钉死两件事:文件名(`CACHEDIR.TAG`,全大写,无扩展名)和文件的**前 43 字节**。其余 —— 注释、应用名、品牌 —— 都是信息性的。

```bash
write_cachedir_tag() {
  local dir="$1" app="$2"
  [ -d "$dir" ] || { echo "no such dir: $dir" >&2; return 1; }
  printf 'Signature: 8a477f597d28d172789f06886806bc55\n# This file is a cache directory tag created by %s.\n# For information about cache directory tags, see https://bford.info/cachedir/\n' "$app" \
    > "$dir/CACHEDIR.TAG"
}

write_cachedir_tag ~/taggedir-demo/cache "demo-script"
```

三个值得背下的实现要点:

- 那 43 字节的头部是一个**字面**字符串 `Signature: 8a477f597d28d172789f06886806bc55`。规范要求 ASCII、强制大小写(`Signature` 大写 S、十六进制小写)、强制冒号后**恰好一个**空格、禁止 `S` 之前出现任何字节 —— 不许 BOM、不许前导空格、不许 shebang([Ford 2004](https://bford.info/cachedir/))。Hacker News 讨论里提的 BOM 建议正是因此被驳回:"一段 Unicode 空白"会强迫每个消费端都带上完整 Unicode 字符数据库 —— 从而把 43 字节契约的轻量性质毁掉([HN 讨论帖,2024 年 11 月](https://news.ycombinator.com/item?id=42093541))。
- 十六进制那串值就是字符串 `.IsCacheDirectory` 的 MD5。Ford 选 magic 值而非赤裸哨兵词是为了让一个被误改的无关 `CACHEDIR.TAG`(比如 README 被脚本误 rename 成这个名)**不会**悄悄把备份软件对某个目录给排除掉。32 位十六进制摘要意外撞名的概率天文级低。
- 这个文件**必须是普通文件**、不能是符号链接。否则攻击者可以用一个名为 `CACHEDIR.TAG` 的符号链接指到 `/etc`,把任意树从备份中抽掉 —— 规范在协议层就预先把这个 foot-gun 拆掉了。

### 2.2 头部验证

各采纳方严格度不一。规范只是*推荐*核对 43 字节头部;HN 帖子里提到 GNU tar 的 `--exclude-caches` 会做核对,BorgBackup 的 `--exclude-caches` 则是"在任何 `CACHEDIR.TAG` 出现的地方都先验签再决定排除目录及其子目录"([Unix.SE 上 CACHEDIR.TAG 与 .nobackup 比较的回答](https://unix.stackexchange.com/questions/724711/are-cachedir-tag-and-nobackup-evaluated-differently))。默认开启验签就对了 —— 详见安全段落。

只依赖 `xxd` 的自检:

```bash
verify_cachedir_tag() {
  local f="$1/CACHEDIR.TAG"
  [ -f "$f" ] || { echo "missing or not regular file: $f"; return 1; }
  local first43
  first43=$(head -c 43 "$f")
  if [ "$first43" = 'Signature: 8a477f597d28d172789f06886806bc55' ]; then
    echo "OK: $f recognized as cache directory tag"
    return 0
  else
    echo "BAD header: '$first43'"; xxd "$f" | head -3; return 1
  fi
}

verify_cachedir_tag ~/taggedir-demo/cache
```

### 2.3 应用侧的三条不变式

为真实应用写标签的人必须守规范"Application Semantics"段落钉下的三条不变式:

1. **缓存子树只写一个标签。** 把单个 `CACHEDIR.TAG` 写在**最顶层**"整目录内容代表缓存"那个目录里。**别**把标签往每个叶子缓存目录里乱撒 —— 既占空间,也会在某些消费端那边触发"一旦下钻就关闭排除"的困惑。
2. **标签覆盖整个子树。** 遵守标签的备份工具会把目录**以及所有子目录**一并排除、不再下钻。如果你在同一个打了标签的目录里混杂缓存内容和原始内容,原始内容也会被排除 —— 而且是悄悄的。目录布局本身就是契约。
3. **标签不见就重建。** 若用户(`rm -rf`)清空了缓存但没删目录本身,应用下次启动就该把 `CACHEDIR.TAG` 重写一遍,否则在下次全量以前这个目录不再被识别为缓存。Gradle、ccache 与 Borg 自己的缓存目录都这么做;从零开始的小脚本最容易漏掉这一点,是常见 bug。

### 2.4 备份工具调用

最常见的三家采纳方,以及各家咬人的差异:

```bash
# GNU tar —— 三个关系很近的 flag,各自对"是否保留 tag 文件"行为不同:
tar -czf ~/backup.tar.gz -C ~/taggedir-demo .                       # 基线:全打包
tar -czf ~/backup.tar.gz --exclude-caches-all -C ~/taggedir-demo .  # 排除缓存目录本身和 CACHEDIR.TAG 文件
tar -czf ~/backup.tar.gz --exclude-caches      -C ~/taggedir-demo .  # 排除缓存内容、保留 CACHEDIR.TAG(默认稳妥)
tar -czf ~/backup.tar.gz --exclude-caches-under -C ~/taggedir-demo . # 同上但不递归子树

# restic —— 单 flag,"保留 tag" 语义:
restic -r /srv/restic-repo backup ~/taggedir-demo --exclude-caches
# restic 文档原文:"exclude a folder's content if it contains the special CACHEDIR.TAG file, but keep CACHEDIR.TAG."
# (https://restic.readthedocs.io/en/latest/040_backup.html)

# BorgBackup —— 对应 flag,带签名校验:
borg create --exclude-caches ~/borg-repo::'{now}' ~/taggedir-demo
```

tar 三个 flag(`--exclude-caches`、`--exclude-caches-all`、`--exclude-caches-under`)只在"缓存目录本身"与"`CACHEDIR.TAG` 文件"是否进入归档这个问题上分化。对开发者树的复现性备份,推荐选 `--exclude-caches`(居中的默认稳妥档),它保留了足够元数据,restore 时缓存目录依旧存在,物主应用能透明地重填。

Unix.SE 上那段比较也厘清了 `--exclude-caches` 与 Borg 通用 `--exclude-if-present NAME` 的差异:后者匹配任意给定名的文件 —— 包括 `.nobackup` —— **不**核验 43 字节头部;它是个自由形式的 hint 机制,不是规范治下的缓存标记。`--exclude-caches` 是严格、带签名核验的那一支;`--exclude-if-present=NAME` 是宽松的逃生口([Unix.SE](https://unix.stackexchange.com/questions/724711/are-cachedir-tag-and-nobackup-evaluated-differently))。

## 步骤三:验证与调优

完整往返一次,确认标签在做你以为的事:

```bash
# 1. 基线(任何处都没 tag):整树入归档
( cd ~/taggedir-demo && rm -f cache/CACHEDIR.TAG; \
  tar -czf /tmp/full.tar.gz -C ~/taggedir-demo . )
tar -tzf /tmp/full.tar.gz | grep -E 'cache/(gradle-build|thumbs|CACHEDIR.TAG)' | sort
# 期望:./cache/gradle-build/aux.json、./cache/thumbs/cover.png.json(缓存内容在)

# 2. 仅在 cache/ 加 tag:
write_cachedir_tag ~/taggedir-demo/cache "demo-script"

# 3. 用 --exclude-caches 归档:
tar -czf /tmp/skip.tar.gz --exclude-caches -C ~/taggedir-demo .
tar -tzf /tmp/skip.tar.gz | grep -E 'cache/(gradle-build|thumbs|CACHEDIR.TAG|.*\.txt)' | sort
# 期望:只有 ./cache/CACHEDIR.TAG(内容消失、标签存活、real-data 原样)
tar -tzf /tmp/skip.tar.gz | grep real-data
# 期望:./real-data/notes.txt(那里没有 CACHEDIR.TAG 故永不被排除)

# 4. 试 --exclude-caches-all(缓存子树里连 tag 也消失):
tar -czf /tmp/all.tar.gz --exclude-caches-all -C ~/taggedir-demo .
tar -tzf /tmp/all.tar.gz | grep -E 'cache/' || echo "no cache entries — tag itself excluded"
```

### 调优坑

- **嵌套标签纯属浪费。** 若 `cache/gradle-build/` 自己也放一个 `CACHEDIR.TAG`,内层不会改变任何行为(外层已把子树排除),但会污染其他归档器的树并扰乱 `du --exclude-caches` 计账。规范说得很明:顶层一只。
- **"缓存"是 hint,不是"安全删除许可"。** 系统软件"**不应**周期性地自动按'陈旧'删掉任意缓存目录中的旧文件"([Ford 2004,"Application Semantics"](https://bford.info/cachedir/))。各应用对陈旧的定义天差地别;一个把打了标签的树当作 tmp 来扫的 cron,迟早会干掉一种缓存 —— 它的内容每文件重生要耗好几分钟。
- **不要懒验签。** 如果你写自定义 `find` 来过滤排除、只看 `CACHEDIR.TAG` 名字但不核验前 43 字节 —— 攻击者能往 `~/.config/CACHEDIR.TAG` 写一个,让你的备份对 `~/.config` 静默失明。永远"按名找到"再"按头核验"后再排除。
- **对 `.nobackup` 用户做测试。** 历史上某些工作流用空文件 `.nobackup` 作同义哨兵。**它们不是同一物**:`.nobackup` 仅在 `--exclude-if-present .nobackup` 下被采纳、无头部可验、无规范可言。把团队从 `.nobackup` 迁到 `CACHEDIR.TAG`,既要写标签、也要把备份命令从 `--exclude-if-present .nobackup` 切到 `--exclude-caches`,runbook 写清楚。

## 最佳实践小结

- **只标顶层缓存目录、只写一次。** 一只 `CACHEDIR.TAG` 在缓存根,所有合规采纳方就会排除整个子树;嵌套标签零增益还搅乱空间计账。
- **永远核验 43 字节头部。** 跳过此步的采纳者(自定义脚本、原生 `find` 过滤)给数据藏匿攻击留了门道。匹配 tar 与 Borg 已出厂的严格度。
- **优先 `--exclude-caches` 而非 `--exclude-caches-all`** —— 为恢复保真,缓存目录与标签都活下来,下次启动应用能透明重填。
- **把标签当备份提示、不是删除许可。** 别仅仅凭标签自动删过期的缓存文件;各应用对"陈旧"的定义不同、重生成代价差几个数量级。
- **把团队从 `.nobackup` 迁出去。** 凡备份工具链支持,把基于文件名的哨兵换成带签名核验的规范版,并把切换中的 flag 变更记进 runbook。

## 参考资料

- Bryan Ford, *Cache Directory Tagging Specification* (v0.6, 2004) —— [`https://bford.info/cachedir/`](https://bford.info/cachedir/)
- Hacker News 讨论(2024 年 11 月 9 日,`networked`)—— [`https://news.ycombinator.com/item?id=42093541`](https://news.ycombinator.com/item?id=42093541)
- `telcoM`,《Are CACHEDIR.TAG and .nobackup evaluated differently?》Stack Exchange 回答 —— [`https://unix.stackexchange.com/questions/724711/are-cachedir-tag-and-nobackup-evaluated-differently`](https://unix.stackexchange.com/questions/724711/are-cachedir-tag-and-nobackup-evaluated-differently)
- restic,《Backing up: excluding files》(`--exclude-caches`)—— [`https://restic.readthedocs.io/en/latest/040_backup.html`](https://restic.readthedocs.io/en/latest/040_backup.html)
- Gradle,`MarkingStrategy.CACHEDIR_TAG` 参考页 —— [`https://docs.gradle.org/current/kotlin-dsl/gradle/org.gradle.api.cache/-marking-strategy/-c-a-c-h-e-d-i-r_-t-a-g.html`](https://docs.gradle.org/current/kotlin-dsl/gradle/org.gradle.api.cache/-marking-strategy/-c-a-c-h-e-d-i-r_-t-a-g.html)
- GNU tar 手册,《Excluding Some Files》(`--exclude-caches`、`--exclude-caches-all`、`--exclude-caches-under`)—— [`https://www.gnu.org/software/tar/manual/html_node/exclude.html`](https://www.gnu.org/software/tar/manual/html_node/exclude.html)
