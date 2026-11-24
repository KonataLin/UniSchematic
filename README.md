<h1 align="center">UniSchematic</h1>

<p align="center">模拟电路原理图自动布局与布线 · USC 设计手册（兼 Skill）</p>

<p align="center">
  <a href="https://github.com/KonataLin/UniSchematic/releases/latest"><strong>下载</strong></a>
  · <a href="./examples/README.md">示例</a>
  · <a href="./Reference/README.md">构图参考</a>
</p>

<p align="center">
  <a href="./Reference/README.md">
    <img src="./Reference/figures/sky130-preview.png" width="840" alt="SKY130 三模块原理图预览">
  </a>
</p>

**前言：本 README 不含任何 AI 成分**，可以放心食用，请**您严肃遵循 UniSchematic 的 License 规范**

> **专有非商用授权 / Proprietary non-commercial license**
>
> The UniSchematic executable and USC documentation are governed by the
> [proprietary license](./LICENSE). You and AI agents acting on your behalf
> may use the specification to author USC files and render them with the
> official binary for personal or academic non-commercial work.
>
> The agreement prohibits unauthorized commercial use, reverse engineering,
> use to build third-party parsers or competing engines, and automated
> harvesting of input/output pairs to train or benchmark competing systems.
>
> UniSchematic 可供个人学习、教育与非商用学术研究使用。协议禁止未经许可的
> 商用、逆向、利用 USC 规范开发第三方解析器或竞品，以及为竞品训练用途批量
> 采集输入与输出。符号数据的独立许可和上游来源见 [symbols](./symbols/ORIGIN.md)。

---

## ★USC 设计理念与使用规范

UniSchematic 旨在给画图模型薄弱的模型、需要快速出图的模型使用，快速综合一个人类可读的电路。**示例参考 [examples 文件夹](./examples/README.md)（一些 <u>好的</u> 设计范例供你参考！）**。

### 1 USC 工作原理：

· USC 将整个 schematic 分为无限个竖直的 track，每个 track 上面按照顺序像葡萄一样挂入一些元件（FIFO 顺序挂入一些元件），你通过分配元件的旋转方式和从属的 track 来构思出好看的 schematic；

​	至于这些 track 的间距、元件的排列方式则由 USC 布局布线器给出，

​	你只要思考怎么设计 track 让整个 schematic 更加规整（要预先考虑到布局布线）在哪些 track 如何排列元件 如何旋转元件……

· USC **支持 module 模块化和模块复用，你在每个 module 里头设计完 track（元件排列方式） 之后**，重复的模块可以在 layout 中指出合适声明！因此，如何把一个大块电路（如果足够小就不必了）按照功能合适区分出模块，这也是之后在阅读本文需要考虑的。综合器逐块校验内部 LVS，顶层连接和电路功能需要你自行核对。

### 2 USC 电路图绘制

· 这一小节包含了 USC 语法的主要理念：轨道挂元件理念；这方便你更好地去思考元件怎么摆放，电压怎么在 track 内部从上往下流动 信号怎么在 track 之间传播，需要你利用我们的理念想清楚，这会比直接使用坐标绘图方便许多；

#### 2.1 track 关键字 ——创建轨道

· track 用来指定竖直轨道，创建一个竖直轨道；需要注意的是，**轨道不代表间距，相邻轨道之间的距离是由综合器给出的**；使用方法为 track(0) 或者 track(3) 同时也支持小数，例如 track(2.5) 表示在 track(2) 和 track(3) 中间那条轨道；**防止写一半发现：哦这个元件应该放在这两个元件中间** 小数轨道是这个作用；**但是需要注意 track(2.01) track(2.99) track(2.5) 同时代表同一条轨道**；

**如何在 track 上挂元件**：每个元件只能被挂一次。**你可以指定某个元件的某个端子作为锚点（端子名通过 `-instance` 查找）**，固定在某个 track 上（具体在哪不用你操心 会通过综合器来算出 你只需要在内心里估计怎么放置是美观符合 IC 拉扎维规范的）。普通器件只写器件名时，锚点取旋转或镜像后的图形包络中心。要让主电流支路严格共轴，建议写到具体引脚（如 `MP.s`、`MN.d`、`RZ.1`、`VDD.p`）。**`port` 直接写端口名即可（如 `VBN`、`{out}`）**。

**★额外注意**：每个器件只能指定它的某**一个端子**在某个 track(n) 上 然后通过旋转（face）来确定其朝向

**含义**：声明编号为 `k` 的竖直列，编号从小到大对应**从左到右**的 X 坐标顺序；冒号后用逗号分隔的元素严格按从上到下（FIFO）排布。

**举例**：

``` code
	track(0): {x}
	track(1): RZ_{phase}.1
	track(2): CZ_{phase}.1
	track(2.5): {cm}                 # 插在列 2 与列 3 之间
	track(3): MP_{phase}.s, MN_{phase}.d
	track(3.5): VDD.p, VSS.p         # 插在两列推挽管之间，精确居中用 center_x
	track(4): AP_{phase}.s, AN_{phase}.d
```

#### 2.2 face关键字——调节元件旋转方向

· 依旧，需要你通过 -instance 指令来查找端子，这里需要支持两种语法形式：

1. **单引脚简写**（常用于电源/地）：

   ```
   face VDD.p down
   face VSS.p up
   ```

2. **批量键值写法**（多个同名朝向的器件/端口用空格隔开，冒号后写 `引脚:方向`）：

   ```
   face MBP MDP MXP: g:left s:up
   face MBN MDN MXN: g:right s:up
   face CP_{phase} CN_{phase} RP_{phase} RN_{phase}: 1:left 2:right
   face C_D: 1:up 2:down
   face VBP VBN XP VINP: p:right
   face XN VINN: p:left
   ```

你需要考虑一个问题：约束需要把旋转或镜像方向定下来。**例如 MOS 只指定栅极方向，仍可能有两种镜像。** 推荐用 `face` 同时指定栅极和源极（或漏极）的方向。也可以把 `face M1: g:left` 和同一器件的 `above M1.d M1.s` 配合使用。旋转方向会极大地影响电路的美观性，需要你思考怎么旋转综合出来的线不会绕来绕去！

**端口(port)朝向是比较特殊的（最容易写反！）**：`p:<方向>` 表示**引脚朝哪边连向电路内部**：

- 放在模块**左边界**的端口 →\to→ 导线向右接入电路 →\to→ 必须写 **`p:right`**；
- 放在模块**右边界**的端口 →\to→ 导线向左接入电路 →\to→ 必须写 **`p:left`**。

#### 2.3`align_y` —— 水平等高对齐与中点对齐

· **多目标等高对齐**（支持 2 个及以上目标，一次写齐）：

```
align_y MBP.s MBN.s MDP.s MDN.s VDD.p
align_y RZ_{phase}.1 CZ_{phase}.1 {out} MP_{phase}.d
```

· **中点对齐 `align_y A (B C)`**：将目标 `A` 的 Y 坐标精准对齐到 `B` 与 `C` 的**垂直中点**。

```
align_y MSA.g (RS_P.2 RS_N.2)           # 栅极对准两颗采样电阻右端正中
align_y VCM_CTRL (MLB.d MSB.d)          # 输出口对准上下管漏极正中
align_y RP_{phase}.1 (AP_{phase}.s AP_{phase}.g)
```

#### 2.4 `above A B` —— 严格上下层级约束

- 强制 `A` 位于 `B` 上方（`A`、`B` 可为器件名或具体引脚名）。

- **常规分层**：拉开上下级电源与管子之间的垂直通道：

  Plaintext

  ```
  above IREF MNREF
  above MDP.d MXP.s
  above MXP.d MIP.d
  above MP_{phase}.d MN_{phase}.d
  ```

​	· **Special Notice!!**: 你需要避免这样的环：A 和 B 是严格错位的 但是又通过别的约束让他们对齐 比如 C 上面的元素和它下面的元素对齐 align_y 这样在数学上是 not accept 的！你用这个的时候需要注意这一点 避免不可能的布局结果！

#### 2.5 `center_x A (B C)` —— 水平居中对称约束

将 `A` 的 X 坐标锁定在 `B` 和 `C` 的水平正中，常用于尾电流管对准差分对、公共电源/地对准左右对称支路：

```
center_x MTAIL.d (MIP.s MIN.s)
center_x MST.d (MSA.s MSB.s)
```

​	· **Special Notice!!**: 2.3 - 2.5 相互配合 是有可能产生数学上无解的情况的 请在三检查避免不可能的布局（impossible layout）

### 3 模块化设计与文件声明

#### 3.1 circuit 命名空间

**`circuit` 是命名空间，`module` 是 Circuit 内部的绘图单元**：

​	一个 `.usc` 文件由一个或多个 `circuit Name { ... }` 构成。**它声明了一块画布**。 

#### 3.2 module 子模块

​	你可以通过**子模块来让电路更加层次化且更清晰**，因此，你有**必要搞清楚你的电路是什么功能并作合适的拆分**！如果模块间是相接的，你可以声明一个 port （见 example）出去，在另外一个 module 也声明一个 port 进来；值得注意的是，**一个布局良好的 schematic 应该是适当拆分的，粒度不会过细也不会过粗，有一定层次化（比如 Bias 单独拿出来），但是又不全放在一起**；(Example 中两个例子 一个是全部挤在一起 但是因为良好的布局 仍然有一定的美观性，另一个则是分了恰当的模块化 + 模块复用 因此看起来也比较美观)

​	子模块可以被复用，用于描述电路中的重复性结构单元；

​	参数化 `module` 一例：

```
circuit OTACells {
  module OutputStage(phase, x, out, cm, color) {
    title "Output {phase} with edge assist"
    ...
  }
}
```

**纯字符串模板替换**：参数全部视为纯字符串，使用 `{param}` 占位，可出现在 `title`、器件名、内联颜色、网名、`port` 名和任意布局约束中；**不支持表达式求值**（如 `{i+1}` 非法）。

**`title "..."`**：可选语句，指定渲染时虚线框正下方居中的标题文本，必须独占一行。

#### 3.3 layout —— 最终把 module 放在一起！实现 module 复用！	

​	在设计完各个子 module 后，入口文件在一个 `circuit` 内写恰好一个非空 `layout` 来调用 module。通过 `include` 引入的库文件可以只写模块定义：

​	下面是我们把 module 都放在 stages.usc 设计完成了 你当然可以单独做成一个文件 这随你的文件管理模式来操作！

```
# 顶层入口文件（如 top.usc）
circuit DifferentialEdgeOTA {
  include "stages.usc"        # 引入同一工程/目录下的模块库文件

  layout {                    # 按书写顺序从左到右横向排布各模块
    OTACells.Bias()
    OTACells.InputStage()
    OTACells.OutputStage(
      phase = P;N,
      x     = XP;XN,
      out   = VOP;VON,
      cm    = VCM_CTRL,       # 长度为 1 的参数自动广播
      color = #B35B25;#805AA7
    )
    OTACells.CommonMode()
  }
}
```

**注释**：以 `#` 开头至行尾。

**模块引用**：同 `circuit` 内可直接写 `ModuleName(...)`，跨 `circuit` 写 `CircuitName.ModuleName(...)`。

**单文件写法**：不需要拆文件时，可以直接把 `module ... { ... }` 和 `layout { ... }` 写在同一个 `circuit` 块中。

**`;` 批量展开规则（Zip + 广播）**：

​	调用参数值中用 `;` 分隔表示列表（如 `phase=P;N` 长度为 2）。

​	同一次调用中的多个列表按同下标一一配对（Zip）展开成多个横向连续模块，不做笛卡尔积。

​	长度为 1 的标量参数（如 `cm=VCM_CTRL`）自动广播给本次展开的所有实例。

​	若同一调用中存在两个长度均大于 1 且不相等的列表，编译器报错。

### 4 常用的元件一列

每个语句独占一行，声明格式为：

```
<器件名>[(<#RRGGBB>)] <网络1> [<网络2> ...] <器件类型>
```

- **内联颜色**：可选，写在器件声明名后的括号内，如 `IREF(#25805C)` 或 `MP_{phase}({color})`。**仅在声明处写一次**，后续约束中仍使用原器件名（如 `IREF`、`MP_{phase}`）。

| **器件类型**         | **声明格式（网络顺序严格固定）**  | **引脚名（用于约束）**   | **face 合法写法示例**     |
| -------------------- | --------------------------------- | ------------------------ | ------------------------- |
| **`NMOS`**           | `M1(#hex) <D> <G> <S> NMOS`       | `.d`, `.g`, `.s`         | `face M1: g:left s:down`  |
| **`PMOS`**           | `M2(#hex) <D> <G> <S> PMOS`       | `.d`, `.g`, `.s`         | `face M2: g:right s:up`   |
| **`NPN`**            | `Q1(#hex) <C> <B> <E> NPN`        | `.c`, `.b`, `.e`         | `face Q1: b:left e:down`  |
| **`PNP`**            | `Q2(#hex) <C> <B> <E> PNP`        | `.c`, `.b`, `.e`         | `face Q2: b:left e:up`    |
| **`CURRENT_SOURCE`** | `I1(#hex) <P> <N> CURRENT_SOURCE` | `.p`, `.n`               | `face I1: p:up n:down`    |
| **`RESISTOR`**       | `R1(#hex) <1> <2> RESISTOR`       | `.1`, `.2`               | `face R1: 1:left 2:right` |
| **`CAPACITOR`**      | `C1(#hex) <1> <2> CAPACITOR`      | `.1`, `.2`               | `face C1: 1:up 2:down`    |
| **`SUPPLY_SYMBOL`**  | `VDD <net> SUPPLY_SYMBOL`         | `.p`                     | `face VDD.p down`         |
| **`GROUND_SYMBOL`**  | `VSS <net> GROUND_SYMBOL`         | `.p`                     | `face VSS.p up`           |
| **`port`**           | `port <net>`                      | `.p`（或直接写 `<net>`） | `face VBN: p:right`       |

> **关键易错点——MOS 网序**：与 SPICE 一致，严格为 **`漏极(D) 栅极(G) 源极(S)`**，且不包含体端（Bulk）。不要把源极和漏极写反。

### 5 可能遇到的问题

**禁止残留未定义语句或换行黏连**：`title "..."` 之后必须换行再写第一条器件声明。器件和约束语句写在 `module` 内，`layout` 只写模块调用。

**等价的约束留一条即可**：同一条 `track` 里选中的锚点已有相同的 X 坐标（例如写了 `track(4): VDD.p, MTAIL.d, VSS.p` 后，`center_x MTAIL.d (MIP.s MIN.s)` 就能把这条轨道放在两点正中）。

**禁止 `align_y` 与 `above` 冲突**：若写了 `align_y A B`，绝对不能再对 `A` 和 `B`（或其同高传递节点）写 `above A B`。

**同列 FIFO 顺序必须与 `above` 一致**：如果 `track(1): A, B`（`A` 在 `B` 上），就不能在别处写 `above B A`，也不能把 `B` 的源极和 `VDD.p` 对齐而把 `A` 和 `VSS.p` 对齐。

**检查每条跨块网是否漏写 `port`**：总图用同名端口标识跨块接口。普通网连到两个及以上模块时，漏写同名 `port` 会给出警告，各块仍分别校验内部 LVS。核对跨模块信号在发送块和接收块都有同名 `port`。

**检查端口方向有没有反**：再次确认最左列端口全是 `p:right`，最右列端口全是 `p:left`。

**检查 `;` 列表长度对齐**：`layout {}` 调用中，所有含 `;` 的参数长度必须完全一致（标量除外），且 `{param}` 拼写与 `module(...)` 签名一字不差。

**检查 `NMOS`/`PMOS` 引脚顺序**：逐行确认第 1 个网是 Drain、第 2 个是 Gate、第 3 个是 Source。

---

## Commercial Licensing / 商业授权

For commercial licensing, contact [wenkang_dou@163.com](mailto:wenkang_dou@163.com).

## Quick Start / 快速调用

Download the archive for your system from GitHub Releases, then extract it.
Run these commands from the extracted directory in PowerShell.

```powershell
.\unischematic.exe --license
.\unischematic.exe -run examples/ua741-q18-right/main.usc --accept-license
.\unischematic.exe -instance inductor
```

On Linux and macOS use `./unischematic`. First use requires explicit
agreement. An interactive terminal may enter `yes` after reviewing the terms.
The binary records acceptance locally and asks again when the agreement changes.

### 出图失败怎么查

先看报错里的模块、文件和行列，再按 `How to fix` 检查。加 `-json` 可读取同样的 `code`、`span`、`related` 和 `hint`。

- 提示 `track(k) requires 'A' above 'B'` 时，同轨列表要求 A 在 B 上方。如果本来想让两者等高，把其中一个放到不同轨道并保留 `align_y`。如果本来想上下叠放，检查列表顺序、`face` 方向和参与等高或上下约束的具体引脚。
- 端子数、端子名或朝向报错时，用 `unischematic -instance MODEL` 核对端子顺序与名称，再调整器件声明或 `face`。

完整的网表转图案例、摆位标注和两版 USC 见 [构图参考](./Reference/REFERENCE.md)。

## uA741 示例图

![uA741 示例图](./examples/ua741-q18-right/final.png)

