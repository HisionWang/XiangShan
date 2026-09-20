# 在 XiangShan 中添加 `vdot.vv` 指令

本文记录在 XiangShan 中添加自定义向量指令 `vdot.vv` 的完整方法。内容覆盖指令编码、前端识别、后端译码、内部操作类型、执行单元、点积数据通路、结果合并、流水线对齐、uop 切分和验证方法。

本文采用的最终指令语义是：

```python
def vdot_vv(vs1, vs2, sew=8):
    result = 0
    for i in range(8):
        result += sign_extend(vs1[i]) * sign_extend(vs2[i])
    vd[0] = result
    vd[1..VLEN-1] = 0
```

也就是说：

```text
输入：8 个 signed int8 元素
输出：1 个 signed int32 结果
写回：只写 vd[0]
其他目标元素：全部写 0
vm：固定为 1
vsew：只支持 e8
```

## 1. 总体执行路径

一条 `vdot.vv` 指令的硬件路径如下：

```text
32-bit instruction
    |
    v
Instructions.scala
    |
    v
RiscvInst.scala 前端向量识别
    |
    v
VecDecoder.scala
    |
    +--> fuType   = FuType.vialuF
    +--> fuOpType = VialuFixType.vdot_vv
    |
    v
已有 Dispatch/Issue/FU 选择路径
    |
    v
VIAluFix
    |
    v
VDot64b：8 个 signed int8 乘积求和
    |
    v
128-bit 结果：低 32 bit 为结果，其余 bit 为 0
    |
    v
Mgu：完成结果写回和清零策略
    |
    v
向量寄存器 vd
```

`vdot` 复用了 `FuType.vialuF`，因此不需要新建 issue queue、dispatch 端口或新的 `FuType`。

## 2. 指令编码

使用的字段是：

```text
funct6 = 110000
vm     = 1
funct3 = 000
opcode = 0001011
```

字段布局如下：

```text
31       26 25 24 20 19 15 14 12 11 7 6 0
+----------+--+-----+-----+-----+----+------+
|  funct6  |vm| vs2 | vs1 |funct3| vd |opcode|
+----------+--+-----+-----+-----+----+------+
|  110000  | 1|  ...|  ...| 000 | ...|0001011|
+----------+--+-----+-----+-----+----+------+
```

## 3. 添加 BitPat

文件：

```text
src/main/scala/xiangshan/backend/decode/Instructions.scala
```

增加：

```scala
object Vdot {
  // funct6=110000, vm=1, funct3=000, opcode=custom-0
  def VDOT_VV = BitPat("b1100001??????????000?????0001011")
}
```

其中 `vd`、`vs1`、`vs2` 使用通配符，`vm` 固定为 `1`。当前实现因此只支持无 mask 形式。

## 4. 前端识别自定义向量指令

文件：

```text
src/main/scala/xiangshan/backend/decode/isa/bitfield/RiscvInst.scala
```

### 4.1 添加 `CUSTOM_0`

在 `OPCODE7Bit` 中加入：

```scala
object OPCODE7Bit {
  val VECTOR_ARITH = "b1010111".U
  val CUSTOM_0     = "b0001011".U
}
```

### 4.2 添加 `isVdot`

在向量指令相关 trait 中加入：

```scala
def isVdot = {
  this.OPCODE === OPCODE7Bit.CUSTOM_0 &&
  this.FUNCT6 === "b110000".U &&
  this.VM === 1.U &&
  this.FUNCT3 === "b000".U
}
```

### 4.3 扩展 `isVecArith`

原本只识别标准向量算术 opcode：

```scala
def isVecArith = {
  this.OPCODE5Bit === OPCODE5Bit.OP_V
}
```

改为：

```scala
def isVecArith = {
  this.OPCODE5Bit === OPCODE5Bit.OP_V || this.isVdot
}
```

这样 `custom-0` 编码的 `vdot` 才会进入向量算术相关的前端处理路径。

## 5. 添加内部操作类型

### 5.1 添加向量 ALU opcode

文件：

```text
yunsuan/src/main/scala/yunsuan/encoding/Opcode/VialuOpcode.scala
```

在 `VialuOpcode` 中增加：

```scala
def vdot = 56.U(width.W)
```

`56` 是 XiangShan 内部使用的 6-bit 运算编号，必须确认它不与已有 opcode 冲突。

### 5.2 添加 `vdot_vv`

文件：

```text
yunsuan/src/main/scala/yunsuan/package.scala
```

在 `VialuFixType` 中增加：

```scala
def vdot_vv = LiteralCat(FMT.VVV, SINT, VialuOpcode.vdot)
```

`VialuFixType` 的字段布局为：

```text
bits[8:7] = format
bit [6]   = signed/unsigned
bits[5:0] = opcode
```

因此该类型表示：两个向量源、signed 整数运算、`vdot` opcode。

## 6. 加入 `VecDecoder` 表项

文件：

```text
src/main/scala/xiangshan/backend/decode/VecDecoder.scala
```

在 `opivv` 表中加入：

```scala
VDOT_VV -> OPIVV(
  FuType.vialuF,
  VialuFixType.vdot_vv,
  T,
  F,
  F,
  UopSplitType.VEC_VVV
),
```

字段含义：

```text
FuType.vialuF        进入现有向量整数 ALU
VialuFixType.vdot_vv 设置内部操作类型
T                    写向量寄存器 vd
F                    不写 mask 寄存器
F                    不更新 vxsat
VEC_VVV              两个源操作数都是向量
```

## 7. 为什么 Dispatch 和 Issue 不需要单独修改

现有 `VialuCfg` 已经配置了：

```scala
val VialuCfg = FuConfig(
  name = "vialuFix",
  fuType = FuType.vialuF,
  fuGen = (p: Parameters, cfg: FuConfig) =>
    Module(new VIAluFix(cfg)(p).suggestName("VialuFix")),
  // ...
)
```

`vdot` 使用的也是：

```scala
FuType.vialuF
```

IssueQueue 和执行单元选择按 `fuType` 匹配，不需要为每一个具体 `fuOpType` 增加新的分发端口。因此默认不需要修改：

```text
src/main/scala/xiangshan/backend/issue/
src/main/scala/xiangshan/backend/dispatch/
src/main/scala/xiangshan/backend/fu/FuType.scala
src/main/scala/xiangshan/Parameters.scala
```

只有在 `vdot` 需要独立资源、不同延迟、独立写回端口或并发限制时，才需要修改执行单元配置。

## 8. 设置源和目的元素宽度

文件：

```text
src/main/scala/xiangshan/backend/fu/wrapper/VIAluFix.scala
```

类：

```scala
class VIAluSrcTypeModule extends Module
```

`vdot` 的类型不是普通 `VVV` 的：

```text
普通 VVV：vs2=eX, vs1=eX, vd=eX
vdot：    vs2=e8, vs1=e8, vd=e32
```

增加：

```scala
private val isVdot = opcode === VialuOpcode.vdot

private val vdotTypes =
  Cat(
    0.U(1.W), isSign, VSew.e8,
    0.U(1.W), isSign, VSew.e8,
    0.U(1.W), isSign, VSew.e32
  ).asTypeOf(new Vs2Vs1VdType)
```

然后在三个类型选择器中优先使用 `vdotTypes`：

```scala
private val vs2Type = Mux1H(Seq(
  isVdot -> vdotTypes.vs2,
  isDstMask -> maskTypes.vs2,
  isExt -> Cat(intType, vextSews.vs2),
  (!isExt && !isDstMask) -> Cat(intType, addSubSews.vs2),
))

private val vs1Type = Mux1H(Seq(
  isVdot -> vdotTypes.vs1,
  isDstMask -> maskTypes.vs1,
  isExt -> Cat(intType, vextSews.vs1),
  (!isExt && !isDstMask) -> Cat(intType, addSubSews.vs1),
))

private val vdType = Mux1H(Seq(
  isVdot -> vdotTypes.vd,
  isDstMask -> maskTypes.vd,
  isExt -> Cat(intType, vextSews.vd),
  (!isExt && !isDstMask) -> Cat(intType, addSubSews.vd),
))
```

`vdotTypes` 应在上述三个 `Mux1H` 之前定义。

### 8.1 限制 `vsew=e8`

增加：

```scala
private val vdotIllegal = isVdot && vsew =/= VSew.e8

private val illegal =
  widenIllegal ||
  narrowIllegal ||
  vextIllegal ||
  vdotIllegal
```

如果未来支持 int16 或 int32 点积，需要同时重新定义输入数量、乘积宽度、累加宽度、输出 EEW 和数据布局，不能只删除这个非法判断。

## 9. 让 ALU 控制表识别 `vdot`

文件：

```text
yunsuan/src/main/scala/yunsuan/vector/VectorALU/VAluDecode.scala
```

当前未知 opcode 会使用：

```scala
val default = BitPat("b?  1  0")
```

因此必须增加：

```scala
val table = Seq(
  BitPat(vdot) -> BitPat("b0  0  0"),
  BitPat(vadd) -> BitPat("b0  0  0"),
  BitPat(vsub) -> BitPat("b1  0  0"),
  // 其他已有条目
)
```

该表项表示：

```text
sub  = 0
misc = 0
cmp  = 0
```

它只负责避免 `vdot` 被误分类为 misc 或 compare。真正的乘法和累加由独立点积模块完成。

## 10. 实现 `VDot64b`

建议文件：

```text
yunsuan/src/main/scala/yunsuan/vector/VectorALU/VDot64b.scala
```

### 10.1 模块接口

因为严格语义只读取 8 个 int8，所以点积模块接收 64 bit、输出 32 bit：

```scala
class VDot64b extends Module {
  val io = IO(new Bundle {
    val fire = Input(Bool())
    val vs1 = Input(UInt(64.W))
    val vs2 = Input(UInt(64.W))
    val vd  = Output(UInt(32.W))
  })
  // 运算逻辑
}
```

### 10.2 八项 signed 点积

删除“两个四元素分组”的实现：

```scala
val result = Wire(Vec(2, SInt(32.W)))
for (g <- 0 until 2) {
  // 每四个元素产生一个结果
}
```

改成一个八项累加：

```scala
val products = Seq.tabulate(8) { i =>
  val a = io.vs1(8 * i + 7, 8 * i).asSInt
  val b = io.vs2(8 * i + 7, 8 * i).asSInt

  (a * b).pad(32)
}

val sum = products.reduce(_ +& _)

io.vd := RegEnable(sum.asUInt, io.fire)
```

这里的要求不能改变：

```text
a、b 都按 signed int8 解释
一共处理 8 个元素
每个乘积按有符号值累加
累加宽度至少为 32 bit
不读取 old_vd
输出寄存一次
```

如果当前 Chisel 版本对 `SInt` 隐式扩展不稳定，应显式将 16-bit 乘积符号扩展到 32 bit，再做 `+&` 累加。

## 11. 在 `VIAluFix` 中连接点积单元

文件：

```text
src/main/scala/xiangshan/backend/fu/wrapper/VIAluFix.scala
```

### 11.1 只实例化一个点积单元

普通 ALU 仍然使用两个 64 bit 子模块：

```scala
private val vIntFixpAlus =
  Seq.fill(numVecModule)(Module(new VIntFixpAlu64b))
```

点积不能写成：

```scala
private val vdotUnits =
  Seq.fill(numVecModule)(Module(new VDot64b))
```

因为当前 `numVecModule=2`，这样会对 16 个 byte 生成多个点积结果。应改为：

```scala
private val vdotUnit = Module(new VDot64b)
```

### 11.2 只读取第一个 64-bit 块

已有输入拆分模块：

```scala
private val vs2Split = Module(new VecDataSplitModule(dataWidth, dataWidthOfDataModule))
private val vs1Split = Module(new VecDataSplitModule(dataWidth, dataWidthOfDataModule))
```

连接：

```scala
vdotUnit.io.fire := io.in.valid
vdotUnit.io.vs1 := vs1Split.io.outVec64b(0)
vdotUnit.io.vs2 := vs2Split.io.outVec64b(0)
```

`outVec64b(0)` 是低 64 bit，对应输入元素 0 到 7。不能把两个 64-bit 块都拿去做点积，否则会违反语义中的 `range(8)`。

### 11.3 生成 128-bit 结果

普通 ALU 结果：

```scala
private val normalVd =
  Cat(vIntFixpAlus.reverse.map(_.io.vd))
```

点积结果：

```scala
private val dotResult32 = vdotUnit.io.vd

private val dotVd =
  Cat(
    0.U((dataWidth - 32).W),
    dotResult32
  )
```

在当前 `dataWidth=128` 时：

```text
dotVd[127:32] = 0
dotVd[31:0]   = dotResult32
```

输出阶段根据已经流水对齐的控制选择结果：

```scala
private val outIsVdot =
  VialuFixType.getOpcode(outCtrl.fuOpType) === VialuOpcode.vdot

private val outVdTmp =
  Mux(outIsVdot, dotVd, normalVd)
```

这里必须使用 `outCtrl`，不能直接使用输入阶段的 `inCtrl`。

## 12. 设置输出 EEW

`vdot` 的输出元素是 e32，而输入 `vsew` 是 e8。因此在 `VIAluFix.scala` 中使用：

```scala
private val outEew = MuxCase(
  outVecCtrl.vsew,
  Seq(
    outIsVdot -> (outVecCtrl.vsew + 2.U),
    outWiden  -> (outVecCtrl.vsew + 1.U)
  )
)
```

并连接：

```scala
mgu.io.in.info.eew := outEew
```

如果 `vsew=e8`，则：

```text
outEew=e32
```

不能让 MGU 继续以 e8 处理点积输出，否则它会以错误的元素粒度进行 tail、mask 和写回计算。

## 13. 确保 `vd[1..]` 清零

### 13.1 普通 MGU 的问题

文件：

```text
src/main/scala/xiangshan/backend/fu/vector/Mgu.scala
```

普通逻辑类似：

```scala
resVecByte(i) := MuxCase(oldVdVecByte(i), Seq(
  activeEn(i) -> vdVecByte(i),
  agnosticEn(i) -> byte1s,
))
```

非 active byte 的默认值是 `oldVd`。因此即使 `dotVd` 的高位已经是零，MGU 仍可能将高位恢复成旧值。

### 13.2 增加 `forceZeroInactive`

在 `MguIO` 的输入 bundle 中增加：

```scala
val forceZeroInactive = Input(Bool())
```

将结果选择改成：

```scala
resVecByte(i) := MuxCase(
  Mux(in.forceZeroInactive, 0.U(8.W), oldVdVecByte(i)),
  Seq(
    activeEn(i) -> vdVecByte(i),
    agnosticEn(i) -> byte1s,
  )
)
```

在 `VIAluFix.scala` 中连接：

```scala
mgu.io.in.forceZeroInactive := outIsVdot
```

于是：

```text
普通指令：非 active 元素默认保留 oldVd
vdot：非 active 元素默认写 0
```

### 13.3 为什么不建议直接绕过 MGU

可以直接在最终输出处对 `vdot` 使用 `dotVd`，但这样会绕过 MGU 的统一处理，影响 `vstart`、active 状态、异常和写回时序。更稳妥的方式是保留 MGU，仅增加 `forceZeroInactive` 这种明确的自定义写回模式。

## 14. uop 切分和 LMUL

当前 `VEC_VVV` 的 uop 数量规则是：

```scala
UopSplitType.VEC_VVV -> lmul
```

普通向量算术可以这样处理，但本文的 `vdot` 是固定归约：

```text
只读取 8 个输入元素
只生成一个结果
只写 vd[0]
```

如果只验证 `LMUL=1`，可以暂时继续使用 `VEC_VVV`。如果要严格保证所有 LMUL 下都只产生一个 uop，应增加专用类型：

文件：

```text
src/main/scala/xiangshan/package.scala
```

增加一个未使用编码：

```scala
def VEC_VDOT = "bxxxxxx".U
```

文件：

```text
src/main/scala/xiangshan/backend/decode/UopInfoGen.scala
```

加入：

```scala
UopSplitType.VEC_VDOT -> 1.U
```

文件：

```text
src/main/scala/xiangshan/backend/decode/VecDecoder.scala
```

将 `VDOT_VV` 的 `UopSplitType.VEC_VVV` 改为：

```scala
UopSplitType.VEC_VDOT
```

如果不准备扩展 LMUL，必须在实现文档和测试中明确限制为 `LMUL=1`，否则多个 uop 可能重复执行点积并产生多个写回结果。

## 15. 覆盖写语义

“覆盖写”包含两个要求：

### 15.1 运算不使用旧 `vd`

点积只能是：

```text
result = sum(vs1[i] * vs2[i])
```

不能变成：

```text
result = old_vd + sum(vs1[i] * vs2[i])
```

因此 `VDot64b` 不需要接收 `oldVd`。

### 15.2 未写元素不能恢复旧 `vd`

即使点积计算不读旧值，如果 MGU 对未 active 元素默认保留旧值，最终仍不符合覆盖写语义。因此必须同时做到：

```text
点积运算不读取 oldVd
vdot 的非 active 输出元素强制写 0
```

## 16. 延迟和流水线

当前 `VialuCfg` 的延迟是：

```scala
latency = CertainLatency(1)
```

因此 `VDot64b` 应使用一级寄存器：

```scala
io.vd := RegEnable(sum.asUInt, io.fire)
```

如果点积数据通路以后改成两级或更多级流水，必须同步修改：

```text
FuConfig.scala 的 latency
VIAluFix 的 valid 对齐
outCtrl
outVecCtrl
outOldVd
outEew
outSrcMask
dotVd
```

否则会出现结果与控制错位。

## 17. 文件修改清单

### 17.1 必须检查或修改

```text
src/main/scala/xiangshan/backend/decode/Instructions.scala
src/main/scala/xiangshan/backend/decode/VecDecoder.scala
src/main/scala/xiangshan/backend/decode/isa/bitfield/RiscvInst.scala

yunsuan/src/main/scala/yunsuan/encoding/Opcode/VialuOpcode.scala
yunsuan/src/main/scala/yunsuan/package.scala
yunsuan/src/main/scala/yunsuan/vector/VectorALU/VAluDecode.scala
yunsuan/src/main/scala/yunsuan/vector/VectorALU/VDot64b.scala

src/main/scala/xiangshan/backend/fu/wrapper/VIAluFix.scala
src/main/scala/xiangshan/backend/fu/vector/Mgu.scala
```

### 17.2 只有扩展 LMUL/uop 语义时需要修改

```text
src/main/scala/xiangshan/package.scala
src/main/scala/xiangshan/backend/decode/UopInfoGen.scala
```

### 17.3 默认不需要修改

```text
src/main/scala/xiangshan/backend/issue/
src/main/scala/xiangshan/backend/dispatch/
src/main/scala/xiangshan/backend/fu/FuType.scala
src/main/scala/xiangshan/Parameters.scala
```

## 18. 验证方法

### 18.1 点积数学结果

输入：

```text
vs1 = [1, 2, 3, 4, 5, 6, 7, 8]
vs2 = [1, 1, 1, 1, 1, 1, 1, 1]
```

期望：

```text
1+2+3+4+5+6+7+8 = 36
```

### 18.2 有符号测试

输入：

```text
vs1 = [-1, -2, 3, -4, 5, -6, 7, -8]
vs2 = [ 1,  2, 3,  4, 5,  6, 7,  8]
```

期望：

```text
-1 - 4 + 9 - 16 + 25 - 36 + 49 - 64
```

### 18.3 极值测试

```text
vs1 的 8 个元素全部为 -128
vs2 的 8 个元素全部为 -128
```

期望：

```text
8 * 16384 = 131072
```

### 18.4 结果布局测试

如果数学结果是 `36`，128-bit 结果必须是：

```text
0x00000000_00000000_00000000_00000024
```

以下结果都不正确：

```text
0x00000024_00000024_00000024_00000024
0x11223344_55667788_99aabbcc_00000024
```

前者表示重复生成多个点积结果，后者表示高位仍然保留了旧 `vd`。

### 18.5 流水线测试

连续执行两条源数据不同的 `vdot.vv`，检查：

```text
第一条指令的结果不会被第二条覆盖
outCtrl.fuOpType 与 dotResult32 属于同一条指令
outIsVdot、outEew、outOldVd 与结果正确对齐
```

重点观察信号：

```text
outCtrl.fuOpType
outIsVdot
dotResult32
outVdTmp
mgu.io.out.vd
```

## 19. 常见错误

### 错误一：每四个元素输出一个结果

```text
错误：8 个输入元素 -> 2 个 int32 结果
正确：8 个输入元素 -> 1 个 int32 结果
```

### 错误二：对两个 64-bit 块分别点积

当前物理数据宽度为 128 bit，不代表该指令应该处理 16 个 byte。严格语义是 `range(8)`，只读取前 8 个 int8。

### 错误三：只把点积模块输出的高位清零

普通 MGU 默认会把非 active 位置恢复成旧 `vd`。因此必须修改 MGU 的默认值，或者提供等价的专用写回路径。

### 错误四：把指令改到 `vimac`

现有 VIMAC 通常包含旧 `vd` 累加语义，容易实现成：

```text
old vd + dot(vs1, vs2)
```

这与当前指令的覆盖写语义不同。除非完整改造 VIMAC，否则保持：

```text
vdot -> FuType.vialuF -> VIAluFix
```

### 错误五：只增加 `VialuOpcode.vdot`

opcode 只是控制编号，不能自动生成乘法、累加、结果布局和清零逻辑。

### 错误六：改变点积流水级但不改变 latency

这会导致数据与 `fuOpType`、mask、vstart、old `vd` 错位。

## 20. 最终检查表

```text
[ ] funct6=110000
[ ] vm=1
[ ] funct3=000
[ ] opcode=0001011
[ ] 前端通过 isVdot 识别
[ ] VecDecoder 使用 FuType.vialuF
[ ] fuOpType 使用 VialuFixType.vdot_vv
[ ] VialuOpcode.vdot 不冲突
[ ] vdot 不进入 misc/compare 路径
[ ] vs1、vs2 按 signed e8 解释
[ ] 只读取 8 个输入 byte
[ ] 8 个乘积累加为一个 signed int32
[ ] old vd 不参与运算
[ ] vd[0] 写入结果
[ ] vd[1..] 全部清零
[ ] 输出 EEW 为 e32
[ ] MGU 不恢复高位 old vd
[ ] 点积数据和控制延迟对齐
[ ] 普通向量 ALU 指令行为不受影响
```

## 21. 结论

添加 `vdot.vv` 不是只增加一条译码表项，也不是只在 ALU 中增加一个乘法表达式。完整实现需要同时建立：

```text
指令编码识别
前端向量预译码
内部 fuOpType
源/目的元素宽度
独立点积数据通路
单结果布局
强制清零写回
流水线控制对齐
边界测试
```

本文语义对应的最终硬件关系是：

```text
8 个 signed int8 输入
    -> 8 个 signed 乘法
    -> 一个 signed int32 累加结果
    -> vd[0] = result
    -> vd[1..] = 0
```

