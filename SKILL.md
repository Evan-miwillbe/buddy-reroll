---
name: buddy-reroll
description:
  修改 Claude Code 的 buddy 宠物。用户想要更换或自定义 /buddy 宠物时触发。
  支持指定物种、稀有度、闪光、帽子等属性，自动搜索匹配的 userID 并写入配置。
---

# Buddy Reroll Skill

修改 Claude Code 的 `/buddy` 伴侣宠物。

## 原理

Claude Code 的宠物属性由 `lS6()` 返回的 ID 经确定性算法生成：

```
lS6() = oauthAccount?.accountUuid ?? userID ?? "anon"
属性 = roll(mulberry32(hash(id + "friend-2026-401")))
```

通过修改 `~/.claude.json` 中的 `userID` 字段，可以控制宠物属性。
**必须同时删除** `oauthAccount.accountUuid`（否则会覆盖 userID）。

## 常量（从 claude.exe 二进制提取）

```
SPECIES = [duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk]
EYES    = [·, ✦, ×, ◉, @, °]
HATS    = [none, crown, tophat, propeller, halo, wizard, beanie, tinyduck]
RARITY_WEIGHTS = {common: 60, uncommon: 25, rare: 10, epic: 4, legendary: 1}
SHINY_CHANCE = 0.01 (1%)
STAT_NAMES = [DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK]
SALT = "friend-2026-401"
```

Common 稀有度的宠物**没有帽子**（固定 none）。

## Hash 函数

- **native 安装**（claude.exe）：Bun 编译，`typeof Bun < "u"` 为 true → 用 `Bun.hash`
- **npm 安装**（Node.js）：`typeof Bun` 为 "undefined" → 用 FNV-1a

检测方法：读取 `~/.claude.json` 的 `installMethod` 字段。

## 执行步骤

### 1. 读取配置 & 检测环境

```bash
# 读取当前 userID 和 installMethod
cat ~/.claude.json | grep -E '"userID"|"installMethod"'
```

检查 `bun` 是否可用（native 安装必需）：
```bash
which bun && bun --version
```

如果用户没有 bun，引导安装：`powershell -c "irm bun.sh/install.ps1 | iex"`

### 2. 询问用户需求

用 AskUserQuestion 一次性问三个问题：
1. **物种**（必选）：从 SPECIES 列表 18 种中选
2. **名字**（可选）：自定义宠物名字。不提供则默认使用物种名（如 turtle 就叫 "turtle"）
3. **高级选项**：稀有度、闪光、帽子、stats 需要调整吗？不需要则默认 legendary + shiny

### 3. 生成搜索脚本并运行

根据 installMethod 生成对应的搜索脚本，写入临时文件后用 `bun` 或 `node` 运行。

**Native 安装**（绝大多数情况）→ 用 Bun.hash：

```javascript
// 生成到 /tmp/buddy-search.mjs 然后用 bun 运行
function bunHash(str) {
  return Number(BigInt(Bun.hash(str)) & 0xffffffffn);
}
```

**npm 安装** → 用 FNV-1a：

```javascript
function fnv1a(str) {
  let h = 2166136261;
  for (let i = 0; i < str.length; i++) {
    h ^= str.charCodeAt(i);
    h = Math.imul(h, 16777619);
  }
  return h >>> 0;
}
```

共同部分：

```javascript
function mulberry32(seed) {
  return function () {
    seed |= 0; seed = seed + 0x6D2B79F5 | 0;
    let t = Math.imul(seed ^ seed >>> 15, 1 | seed);
    t = t + Math.imul(t ^ t >>> 7, 61 | t) ^ t;
    return ((t ^ t >>> 14) >>> 0) / 4294967296;
  };
}

const SPECIES  = ['duck','goose','blob','cat','dragon','octopus','owl','penguin','turtle','snail','ghost','axolotl','capybara','cactus','robot','rabbit','mushroom','chonk'];
const EYES     = ['\u00B7','\u2726','\u00D7','\u25C9','@','\u00B0'];
const HATS     = ['none','crown','tophat','propeller','halo','wizard','beanie','tinyduck'];
const RARITIES = ['common','uncommon','rare','epic','legendary'];
const RARITY_WEIGHTS = {common:60, uncommon:25, rare:10, epic:4, legendary:1};
const STAT_NAMES = ['DEBUGGING','PATIENCE','CHAOS','WISDOM','SNARK'];

function pick(rng, arr) { return arr[Math.floor(rng() * arr.length)]; }

function rollRarity(rng) {
  const total = Object.values(RARITY_WEIGHTS).reduce((a, b) => a + b, 0);
  let r = rng() * total;
  for (const rarity of RARITIES) { r -= RARITY_WEIGHTS[rarity]; if (r < 0) return rarity; }
  return 'common';
}

function predict(uid, hashFn) {
  const salted = uid + 'friend-2026-401';
  const rng = mulberry32(hashFn(salted));
  const rarity = rollRarity(rng);
  const species = pick(rng, SPECIES);
  const eye = pick(rng, EYES);
  const hat = rarity === 'common' ? 'none' : pick(rng, HATS);
  const shiny = rng() < 0.01;
  const stats = {};
  for (const s of STAT_NAMES) stats[s] = Math.floor(rng() * 100) + 1;
  return { species, rarity, eye, hat, shiny, stats };
}
```

搜索逻辑：遍历 `buddy-0` 到 `buddy-N`，筛选匹配用户需求的 UID。
如果有 stats 偏好，按偏好排序后取 Top 10。
搜索上限 2 亿，通常几秒到几十秒就能找到结果。

运行命令（native）：
```bash
bun /tmp/buddy-search.mjs
```

运行命令（npm）：
```bash
node /tmp/buddy-search.mjs
```

### 4. 展示结果让用户选择

格式化输出每个匹配的 UID 及其属性，让用户选择。

### 5. 写入配置

**修改前先备份**：
```bash
cp ~/.claude.json ~/.claude.json.bak-buddy
```

**修改三个字段**：
1. `userID` → 选定的 UID（如 `buddy-145297`）
2. `companion` → `{ "name": "用户指定的名字或物种名" }`（设名字，强制重新孵化）
3. 删除 `oauthAccount.accountUuid`（防止覆盖 userID）

用 Edit 工具精确修改这三个字段，不要重写整个文件。

### 6. 验证

告诉用户：
1. **退出当前 Claude 会话**
2. **重启 Claude Code**
3. **输入 `/buddy` 孵化新宠物**
4. 如果结果不对，用 `cp ~/.claude.json.bak-buddy ~/.claude.json` 恢复

## 常见问题

**Q: 重启后宠物没变？**
A: 服务器可能在运行时恢复了 accountUuid。确保 `oauthAccount` 中没有 `accountUuid` 字段。

**Q: 想恢复原来的宠物？**
A: `cp ~/.claude.json.bak-buddy ~/.claude.json`，重启即可。

**Q: 可以指定具体 stats 值吗？**
A: Stats 范围 1-100，不能精确指定，但可以搜索 WIS+DBG 总分最高的。

**Q: Common 稀有度能戴帽子吗？**
A: 不能。代码中 `hat: rarity === "common" ? "none" : pick(rng, HATS)`，common 固定无帽。
