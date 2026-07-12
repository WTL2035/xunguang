<div align="center">

# 寻光 V2.3

### A darkness-navigation game powered entirely by sound

![HTML5](https://img.shields.io/badge/HTML5-E34F26?style=flat-square&logo=html5&logoColor=white)
![CSS3](https://img.shields.io/badge/CSS3-1572B6?style=flat-square&logo=css3&logoColor=white)
![JavaScript](https://img.shields.io/badge/ES6_Class-000?style=flat-square&logo=javascript&logoColor=F7DF1E)
![Web Audio](https://img.shields.io/badge/Web_Audio_API-000?style=flat-square&logo=googlechrome&logoColor=white)
![Zero Dependencies](https://img.shields.io/badge/Zero_Dependencies-000?style=flat-square)

**一个 3419 行的单文件 HTML 项目，没有一行外部依赖，却实现了一整套程序化音频引擎、BFS 保证可达的地图生成、状态机驱动的游戏架构和多模态反馈系统。**

*"所有的感知，都是代码实时计算出来的——这是黑暗中最纯粹的光。"*

</div>

---

## 概述

「寻光」是一款**无画面**的声音导航游戏。玩家扮演一位失去视觉的行者，在完全黑暗中仅凭声音回响、触觉震动和文字描述找到出口。

> 没有一帧渲染，没有一张贴图，没有一个外部资源文件——所有的感知，都是代码实时计算出来的。
>
> *当视觉缺席，听觉便成为眼睛。每一次回响，都是黑暗在为你让路。*

---

## 架构总览

```
┌─────────────────────────────────────────────────────────┐
│                      Game (状态机)                        │
│  ┌─────────┐   ┌──────────┐   ┌────────────────────┐   │
│  │ start   │──▶│ playing  │──▶│   found (光明)      │   │
│  └─────────┘   └──────────┘   └─────────┬──────────┘   │
│       ▲              │                    │              │
│       └──────────────┴────────────────────┘              │
│                     nextChapter()                        │
├─────────────────────────────────────────────────────────┤
│  MapGenerator        AudioEngine         EventSystem     │
│  ┌──────────┐   ┌──────────────────┐   ┌────────────┐  │
│  │ Grid     │   │ Oscillator Nodes │   │ 30+ Events │  │
│  │ BFS      │   │ Convolver Reverb │   │ 10 Effects │  │
│  │ Walls[]  │   │ Stereo Panner    │   │ Weighted   │  │
│  └──────────┘   │ LFO Modulation   │   └────────────┘  │
│                  │ Noise Generator  │                    │
│                  │ Speech Synthesis │                    │
│                  └──────────────────┘                    │
├─────────────────────────────────────────────────────────┤
│  AchievementSystem  │  SaveSystem  │  VibrationManager   │
│  18 Achievements    │  5 Slots     │  Haptic Patterns    │
│  Stats Tracking     │  LocalStorage│  Device API         │
└─────────────────────────────────────────────────────────┘
```

---

## 技术亮点

> *一个没有画面的游戏，需要怎样的技术，才能让玩家"看见"黑暗？*

### 1. 程序化音频引擎 — 没有一个外部音频文件

整个游戏的音效全部由 `AudioEngine` 类通过 Web Audio API **实时合成**，零音频资源。这不仅节省了带宽，更重要的是让每个音效都可以根据游戏状态动态调整参数。

*在「寻光」里，声音不是录好的文件，而是代码在实时呼吸。每一脚步的沙沙声、每一次碰撞的回响、每一缕找到光明时升起的和弦，都是数学公式在扬声器中绽放。*

#### 信号链路架构

每个音效都经过精心设计的 DSP 节点链路：

```
Oscillator / BufferSource
        │
        ▼
  BiquadFilter (bandpass / lowpass)
        │
        ▼
  StereoPanner (方向感知)
        │
        ▼
     GainNode (包络塑形)
        │
        ▼
   ConvolverNode (混响空间)
        │
        ▼
  MasterGain → Destination
```

#### 回声定位 — 用声音代替视觉

碰撞墙壁时，系统根据**碰撞方向**和**距离目标的远近**实时调整音效参数——就像蝙蝠用回声丈量洞穴，玩家用声音丈量黑暗：

```javascript
playEcho(distance, direction) {
    // 频率随距离衰减：越远越低沉
    const freq = 1000 - (distance * 150);
    osc.frequency.setValueAtTime(Math.max(400, freq), now);
    osc.frequency.exponentialRampToValueAtTime(
        Math.max(200, freq * 0.6), now + 0.15
    );

    // 立体声声像：左/右/中心
    if (direction === 'left')  panner.pan.value = -0.8;
    if (direction === 'right') panner.pan.value = 0.8;

    // 滤波器 Q 值随距离变化：近处更尖锐
    filter.Q.value = 5 + (3 - distance);

    // 混响干湿比：距离越远混响越多
    reverbGain.gain.value = vol * 0.3;
}
```

#### 环境氛围音 — LFO 调制的持续低频无人机

三层不同频率的正弦波叠加，每层都带有独立的低频振荡器（LFO）调制频率，营造出不安定的黑暗氛围——像夜风穿过空旷走廊的低吟：

```javascript
createDrone = (freq, gainVal, panVal) => {
    const osc = ctx.createOscillator();      // 基础音
    const lfo = ctx.createOscillator();      // 低频调制器
    const lfoGain = ctx.createGain();

    lfo.frequency.value = 0.1 + Math.random() * 0.2; // 0.1~0.3Hz
    lfoGain.gain.value = 2;                           // ±2Hz 偏移
    lfo.connect(lfoGain);
    lfoGain.connect(osc.frequency);                    // 调制音高
};

createDrone(60, 0.08, -0.5);   // 左声道 60Hz
createDrone(62, 0.06, 0.5);    // 右声道 62Hz
createDrone(55, 0.05, 0);      // 中心 55Hz
```

叠加一层 **200Hz 低通滤波的白噪声**（`noiseGain = 0.02`），模拟黑暗中的微弱环境底噪。

#### 步行声 — 带通滤波噪声

每一步都从随机白噪声开始，经过带通滤波器（150Hz 中心频率）和指数衰减包络，模拟真实的脚步声——那是黑暗中唯一属于自己的节奏：

```javascript
playStep() {
    const buffer = ctx.createBuffer(1, sampleRate * 0.1, sampleRate);
    const data = buffer.getChannelData(0);
    for (let i = 0; i < bufferSize; i++) {
        const white = Math.random() * 2 - 1;
        data[i] = (white + data[i-1] || 0) * 0.5;  // 简单低通
    }
    // 带通 150Hz → 指数衰减 80ms
    gain.gain.setValueAtTime(0.2, now);
    gain.gain.exponentialRampToValueTime(0.01, now + 0.08);
}
```

#### 碰撞声 — 锯齿波 + 方波金属质感

```javascript
playBump() {
    osc.type = 'sawtooth';              // 低频冲击
    osc.frequency.exponentialRampToValueAtTime(40, now + 0.2);

    metalOsc.type = 'square';           // 高频金属泛音
    metalOsc.frequency.value = 800;
    metalGain.gain.exponentialRampToValueAtTime(0.001, now + 0.1);
}
```

#### 找到光明 — 三和弦 + 频率上扫

*当玩家抵达终点，440Hz 的正弦波缓缓滑向 880Hz，C-E-G 三和弦依次亮起——这是代码写给光明的赞美诗：*

```javascript
playLightFound() {
    // 主音 440Hz → 880Hz 频率上扫（1.5秒）
    osc.frequency.setValueAtTime(440, now);
    osc.frequency.exponentialRampToValueAtTime(880, now + 1.5);

    // C-E-G 三和弦依次奏响
    const chordFreqs = [523.25, 659.25, 783.99];
    chordFreqs.forEach((freq, i) => {
        // 每个音延迟 200ms 进入，营造"渐亮"效果
    });

    // 2000Hz → 4000Hz 三角波扫频（光芒闪烁感）
}
```

#### 语音合成 — Web Speech API

```javascript
speak(text) {
    const u = new SpeechSynthesisUtterance(text);
    u.lang = 'zh-CN';
    u.rate = 0.9;   // 稍慢于正常语速
    u.pitch = 0.95;  // 略低沉
    window.speechSynthesis.speak(u);
}
```

---

### 2. BFS 保证可达的地图生成

每一张地图都经过 **广度优先搜索（BFS）** 验证，确保起点 `(1,1)` 到终点 `(size-2, size-2)` 之间**一定存在通路**——*算法替玩家许下承诺：无论黑暗多深，光明永远可达。*

```javascript
generateMap() {
    const size = Math.max(5, 6 + this.level + this.config.mapSizeBonus);
    const count = Math.max(0, 3 + this.level * 2 + this.config.wallCountBonus);

    while (this.walls.length < 4 * size + count && attempts < maxAttempts) {
        // 尝试放置一面墙
        this.walls.push({x, y});

        // BFS 检查：如果不可达，立即回滚
        if (!this.isPathExists(1, 1, size-2, size-2)) {
            this.walls.splice(idx, 1);  // 移除这面墙
        }
    }
}

// BFS 实现
isPathExists(sx, sy, gx, gy) {
    const queue = [{x: sx, y: sy}];
    const visited = new Set();
    visited.add(`${sx},${sy}`);

    while (queue.length > 0) {
        const {x, y} = queue.shift();
        if (x === gx && y === gy) return true;  // 找到目标

        for (const [dx, dy] of [[0,1],[0,-1],[1,0],[-1,0]]) {
            // 广度优先扩展四个方向
        }
    }
    return false;  // 无法到达
}
```

地图规模随章节递增：`size = max(5, 6 + level + difficulty_bonus)`，障碍数量：`walls = max(0, 3 + level*2 + difficulty_bonus)`。

---

### 3. 状态机驱动的游戏循环

游戏使用明确的状态机管理生命周期，避免了复杂的条件分支——*每一个状态都是旅程中的一个瞬间：出发、行走、找到光、再次走入黑暗，循环往复，直到传说行者诞生。*

```
State: start ──────────────────────────────────────────▶ playing
   │                                                       │
   │  start()                                              │
   │  audio.init()                                         │
   │  scan()                                               │
   │                                                       ▼
   │                                                  ┌─────────┐
   │                                          handleMove()      │
   │                                                  │         │
   │                                          ┌───────┴───┐    │
   │                                          │ wall hit?  │    │
   │                                          └───────┬───┘    │
   │                                              no  │  yes   │
   │                                                  │   │    │
   │                                          scan()  │   │ bump()
   │                                                  │   │ echo()
   │                                          event?  │   │ vibrate()
   │                                          ┌───┴──┐    │
   │                                         yes    no    │
   │                                         │      │     │
   │                                    triggerRandomEvent│
   │                                         │      │     │
   │                                    waitingForContinue │
   │                                         │      │     │
   │                                    continueEvent()   │
   │                                         │      │     │
   │                                         └──┬───┘     │
   │                                            │         │
   │                                    goal reached?     │
   │                                      ┌───┴───┐      │
   │                                     yes     no ─────┘
   │                                      │
   │                               findLight()
   │                               playLightFound()
   │                               CSS transition (2.5s)
   │                                      │
   │                               nextChapter()
   │                               generateMap()
   │                                      │
   └──────────────────────────────────────┘
```

关键设计决策：
- **状态互斥**：`state !== 'playing'` 时忽略所有输入，防止状态混乱
- **异步冷却**：`eventCooldown` 递减机制控制事件触发频率
- **眩晕锁定**：`isStunned = true` 时冻结移动输入，用 `setTimeout` 定时解锁

---

### 4. 多模态反馈系统

游戏同时通过**四种感知通道**传递信息，让玩家在没有画面的情况下也能获得完整的空间感知——*当眼睛关闭，耳朵、皮肤和意识同时张开。*

| 通道 | 技术 | 用途 |
|:---:|:---:|:---|
| **听觉** | Web Audio API + SpeechSynthesis | 环境感知、方向判断、叙事朗读 |
| **触觉** | Vibration API | 碰撞反馈、事件提示、光明到达 |
| **视觉** | CSS 渐变动画 + 文字 | 状态指示、状态光晕（红/绿/金/蓝） |
| **认知** | 文字叙事系统 | 环境描述、距离提示、事件叙述 |

#### 震动模式设计

每种事件都有独特的震动模式，用触觉传递情绪——*皮肤成了阅读世界的语言：*

```javascript
// 撞墙：短-停-短（冲击感）
[150, 80]

// 摔倒：长-短-长（疼痛感）
[200, 100, 200]

// 找到光明：渐强节奏（希望感）
[50, 100, 50, 100, 50, 200, 100, 200]

// 正面事件：三个短脉冲（愉悦感）
[30, 30, 30]
```

#### 状态光晕动画

碰撞和事件触发时，屏幕中央出现径向渐变光晕：

```css
#status-overlay.red {
    background: radial-gradient(
        circle at center,
        rgba(255, 50, 50, 0.4),   /* 中心红色 */
        transparent 60%            /* 边缘透明 */
    );
    animation: pulse-red 0.8s ease-out;
}
```

四种状态颜色：🔴 红色（碰撞/负面）、🟢 绿色（找到光明）、🟡 金色（正面事件）、🔵 蓝色（普通提示）。

---

### 5. 权重随机事件系统

事件触发基于**冷却计数器 + 概率阈值**的双重控制——*命运的骰子，被难度参数悄然调校：*

```
每步移动 → eventCooldown--
         → eventCooldown ≤ 0 ?
              └─ yes → Math.random() < eventChance ?
                         └─ yes → 从加权池中随机选择事件
```

正面/负面事件的权重由难度决定：

| 难度 | `positiveEventWeight` | 每章最多事件数 | 冷却基数 |
|:---:|:---:|:---:|:---:|
| 简单 | 0.80 (80% 正面) | 1 | 5 |
| 普通 | 0.65 | 2 | 4 |
| 正常 | 0.60 | 2 | 4 |
| 困难 | 0.45 (55% 负面) | 3 | 3 |

每个事件携带 `vibration` 数组、`color` 状态色和 `effect` 类型标识，触发时三路并行输出：

```javascript
this.showStatus(event.color);    // 视觉光晕
this.vibrate(event.vibration);   // 触觉反馈
this.narrate(event.title, ...);  // 听觉叙事 + 语音朗读
```

10 种事件效果类型：`teleport`（传送）、`guide`（方向指引）、`vision`（视野恢复）、`shortcut`（移除障碍）、`skip`（跳跃前进）、`speed`（加速）、`heal`（恢复状态）、`stun`（眩晕）、`confuse`（迷失方向）、`slow`（减速）、`teleport_random`（随机传送）。

---

### 6. CSS 视觉系统

#### 2.5 秒全局过渡动画

找到光明时，整个页面从黑底白字**平滑过渡**到白底黑字——*2.5 秒的渐变，是黑暗缓缓退去的呼吸：*

```css
body {
    background: #000;
    color: #fff;
    transition: background-color 2.5s ease, color 2.5s ease;
}

body.light-found {
    background: #fff;
    color: #000;
}
```

#### 玻璃态按钮

方向按钮使用 `backdrop-filter: blur(10px)` 实现毛玻璃效果：

```css
.dir-btn {
    background: rgba(255, 255, 255, 0.03);
    border: 1px solid rgba(255, 255, 255, 0.15);
    backdrop-filter: blur(10px);
}
```

#### 呼吸动画

标题文字持续呼吸效果，暗示"光"的律动——*在黑暗中，光不是恒定的，它在呼吸，在等待被发现：*

```css
@keyframes breathe {
    0%, 100% { opacity: 0.7; }
    50% { opacity: 1; }
}
```

---

### 7. AI 停滞提示系统

当玩家在 20 秒内没有任何操作，系统自动计算目标方向并给出提示——*像是黑暗中一双看不见的手，在你迷路时轻轻拍了拍你的肩：*

```javascript
resetAIHintTimer() {
    this.aiHintTimer = setTimeout(() => this.showAIHint(), 20000);
}

showAIHint() {
    const dx = this.goal.x - this.player.x;
    const dy = this.goal.y - this.player.y;
    // 选择距离分量较大的轴作为提示方向
    if (Math.abs(dx) > Math.abs(dy))
        hint = dx > 0 ? '往右走' : '往左走';
    else
        hint = dy > 0 ? '往下走' : '往上走';

    this.ui.aiHint.textContent = `AI提示：${hint}`;
    this.audio.playAIHint();
    // 30秒冷却期，防止频繁提示
    setTimeout(() => this.aiHintCooldown = false, 30000);
}
```

---

### 8. 数据持久化架构

使用 LocalStorage 存储三类独立数据，实现存档与成就的分离管理——*每一段旅程都被记住，每一次突破都被铭记：*

```javascript
// 存档数据
localStorage.setItem('xunguang_saveSlots', JSON.stringify({
    saveSlots: this.saveSlots  // [{level, difficulty, timestamp}, ...]
}));

// 成就数据（章节成就单独存储，读取存档时不重置）
localStorage.setItem('xunguang_achievements', JSON.stringify({
    unlocked: [...this.unlockedAchievements],
    chapterAchievements: [...this.chapterAchievements],
    stats: this.stats  // 20+ 项统计指标
}));

// 实验室设置
localStorage.setItem('xunguang_extraEvents', this.extraEventsEnabled);
```

---

## 🥚 隐藏彩蛋 — 有些光，需要你亲手去点亮

这不是一个会让你绕路的功能，而是一个安静的秘密——它不写在教程里，不出现在提示中，只有足够好奇的人才能发现。

标题界面上，**「寻光」两个字中的「光」字**，是一个可交互的元素。你可以点击它。一次，两次，三次……

在第十次点击时：

> 噗。像有什么东西裂开了。

「光」字开始发出金色的光芒，随着微微的呼吸脉动。存档功能提前解锁，隐藏成就「早鸟」被点亮。

```javascript
onGuangClick() {
    this.guangClickCount++;
    if (this.guangClickCount >= 10 && !this.saveUnlockedEarly) {
        this.saveUnlockedEarly = true;
        this.stats.earlyUnlockSave = true;
        this.saveAchievementData();
        this.checkAchievements();
        this.updateSaveButtonState();
        this.showAlert('恭喜！你发现了隐藏彩蛋！\n存档功能已提前解锁！');
        document.querySelector('.guang-char').classList.add('unlocked');
        // CSS 将「光」字渲染成金色，并加上黄金阴影
    }
}
```

CSS 层面，这个状态由一行优雅的声明驱动：

```css
#game-title .guang-char.unlocked {
    color: #ffd700;
    text-shadow: 0 0 20px rgba(255, 215, 0, 0.5);
}
```

*原本需要通关第 10 章才能解锁存档——但你用 10 次点击，在旅程刚开始时就让黑暗裂开了一道缝。*

> 黑暗中总藏着秘密，而有些秘密，只对愿意反复触摸光的人开口。

---

## 游戏特性

> *在这段旅程中，你将学会用耳朵看世界，用皮肤读故事。*

### 四档难度

| 难度 | 地图规模公式 | 障碍数公式 | 事件概率 | 正面占比 | 最大事件数 |
|:---:|:---:|:---:|:---:|:---:|:---:|
| 简单 | `max(5, 4+L)` | `max(0, 1+2L)` | 15% | 80% | 1 |
| 普通 | `max(5, 5+L)` | `max(0, 2+2L)` | 20% | 65% | 2 |
| 正常 | `max(5, 6+L)` | `max(0, 3+2L)` | 25% | 60% | 2 |
| 困难 | `max(5, 8+L)` | `max(0, 6+2L)` | 35% | 45% | 3 |

> L = 当前章节号

### 30+ 随机事件

*黑暗中的相遇，有善意，也有险阻。每一步都可能改变命运的轨迹：*

**正面事件**：遇见巡警、导盲犬指引、老友重逢、热心店主、志愿者、出租车司机、晨跑老人、花店老板、教堂钟声...

**负面事件**：不慎摔倒、施工路段、突如其来的雨、上错车、拥挤人群、狂吠的犬、丢失钱包、积水坑、碎玻璃、醉汉...

### 18 项成就

*每一个成就，都是黑暗中的一座里程碑。你走过的每一步，都被光记住：*

| 图标 | 成就 | 解锁条件 |
|:---:|:---|:---|
| 👣 | 初出茅庐 | 完成第1章 |
| 🌟 | 生存者 | 完成第5章 |
| 👑 | 寻光大师 | 完成第10章 |
| ⭐ | 传说行者 | 完成第20章 |
| 🍀 | 幸运儿 | 连续遇到3次正面事件 |
| 🌧️ | 倒霉蛋 | 连续遇到3次负面事件 |
| ⚡ | 速通者 | 困难模式完成第3章 |
| 🕊️ | 和平主义者 | 零负面事件完成第5章 |
| 🗺️ | 探索者 | 单章移动超过50步 |
| 💪 | 东山再起 | 被传送后依然完成该章 |
| 🦉 | 夜行者 | 困难模式累计完成20章 |
| 🧱 | 撞墙专家 | 单章撞墙超过20次 |
| 📚 | 事件收集者 | 累计触发50次事件 |
| 🤫 | 静默行者 | 连续5章零事件 |
| 💾 | 存档大师 | 使用全部5个存档位 |
| 🔓 | 早鸟 | 提前解锁存档 |
| 🤖 | 寻求帮助 | 触发AI提示 |
| 🏃 | 马拉松 | 累计移动超过1000步 |

### 操作方式

| 平台 | 操作 |
|:---:|:---|
| 桌面 | `↑` `↓` `←` `→` 或 `W` `A` `S` `D` |
| 移动 | 屏幕四角方向按钮 |
| 继续 | 点击屏幕 / `Space` / `Enter` |

---

## 快速开始

*戴上耳机，打开浏览器，闭上眼睛——旅程从黑暗开始：*

```bash
# 方式一：直接打开
open index.html

# 方式二：本地服务器
npx serve .
```

浏览器打开即玩，无需安装任何依赖。

---

## 项目结构

```
寻光/
├── index.html          # 全部代码：HTML结构 + CSS样式 + JS逻辑（3419行）
└── README.md           # 本文件
```

---

## 技术栈

*3419 行代码，8 种技术，一个承诺：没有画面，也能看见世界。*

| 类别 | 技术 | 用途 |
|:---:|:---:|:---|
| 结构 | HTML5 | 语义化 DOM 结构 |
| 样式 | CSS3 | 动画、渐变、响应式布局 |
| 逻辑 | ES6 Class | AudioEngine / Game 类架构 |
| 音频 | Web Audio API | 程序化音频合成 |
| 语音 | SpeechSynthesis API | 文字转语音 |
| 触觉 | Vibration API | 移动端震动反馈 |
| 存储 | LocalStorage | 存档与成就持久化 |
| 算法 | BFS | 地图可达性验证 |

---

## 版本历史

*每一次迭代，都是光在黑暗中走得更远的足迹：*

| 版本 | 更新内容 |
|:---:|:---|
| V2.3 | 当前版本 |
| V2.2 | 弹窗触控滑动优化 |
| V2.1 | 丰富成就系统与统计追踪 |
| V2.0 | BFS算法、存档系统、难度选择、AI提示 |
| V1.0 | 初始版本（@雾画笙雨） |

---

## 作者

*这首诗由两个人写就——一个人点燃了第一盏灯，另一个人让它照亮了更远的路：*

- 初始版本：@雾画笙雨
- 后续开发：[@WTL2035](https://github.com/WTL2035)

---

<div align="center">

**3419 lines. Zero dependencies. All sound, no sight.**

*"在最深的黑暗里，声音成了眼睛，震动成了语言，而代码——是那束永不熄灭的光。"*

</div>
