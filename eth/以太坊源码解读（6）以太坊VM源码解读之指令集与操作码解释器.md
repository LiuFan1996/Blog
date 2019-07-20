# 以太坊指令集与操作码解释器

EVM事实是个堆栈机器。指令可能会使用栈上的数值作为参数，也会将值作为结果压入栈中，而指令的构成是由我们所编写的合约的ABI文件所生产，大致结构为

编写合约 > 生成ABI > 解析ABI得出指令集 > eth会将指令通过core/vm/opcodes.go文件中的操作码映射，映射成操作码集 > 生成一个operation[256] >

```go
type operation struct {
    //下列函数在core/vm/jump_table.go文件中定义了模板
   //操作函数
   execute executionFunc 
   // gas函数，返回执行所需的gas
   gasCost gasFunc
   // 验证操作的堆栈(大小)
   validateStack stackValidationFunc
   // 返回操作所需的内存大小
   memorySize memorySizeFunc

   halts   bool // 指示操作是否应停止进一步执行
   jumps   bool // 指示程序计数器是否不应递增
   writes  bool // 确定此操作是否为状态修改操作
   valid   bool // 指示检索的操作是否有效和已知
   reverts bool // 确定操作是否恢复状态(隐式停止)
   returns bool // 确定操作是否设置返回的数据内容
}
```

```go
type (
   executionFunc       func(pc *uint64, interpreter *EVMInterpreter, contract *Contract, 							memory *Memory, stack *Stack) ([]byte, error)
   gasFunc             func(params.GasTable, *EVM, *Contract, *Stack, *Memory, uint64) 							(uint64, error) 
   stackValidationFunc func(*Stack) error
   memorySizeFunc      func(*Stack) *big.Int
)
```

根据当前的版本选择对应的操作码生成函数

```go
var (
	frontierInstructionSet       = newFrontierInstructionSet()
	homesteadInstructionSet      = newHomesteadInstructionSet()
	byzantiumInstructionSet      = newByzantiumInstructionSet()
	constantinopleInstructionSet = newConstantinopleInstructionSet()
)
```

例如：家园版本

```go
//家园阶段可以执行的指令。
func newHomesteadInstructionSet() [256]operation {
   instructionSet := newFrontierInstructionSet()
   instructionSet[DELEGATECALL] = operation{
      execute:       opDelegateCall,
      gasCost:       gasDelegateCall,
      validateStack: makeStackFunc(6, 1),
      memorySize:    memoryDelegateCall,
      valid:         true,
      returns:       true,
   }
   return instructionSet
}
```

这样，对evm的操作实际上是操作码对程序的操作。

core/vm/opcodes.go：部分代码，实际上就是一堆的常量操作符，这些操作符会在core/vm/jump_table.go这个文件中映射成对应的操作函数，用于执行

```go
const (
   STOP OpCode = iota
   ADD
   MUL
   SUB
   DIV
   SDIV
   MOD
   SMOD
   ADDMOD
   MULMOD
   EXP
   SIGNEXTEND
)

// 0x10 range - comparison ops.
const (
   LT OpCode = iota + 0x10
   GT
   SLT
   SGT
   EQ
   ISZERO
   AND
   OR
   XOR
   NOT
   BYTE
   SHL
   SHR
   SAR

   SHA3 = 0x20
)

// 0x30 range - closure state.
const (
   ADDRESS OpCode = 0x30 + iota
   BALANCE
   ORIGIN
   CALLER
   CALLVALUE
   CALLDATALOAD
   CALLDATASIZE
   CALLDATACOPY
   CODESIZE
   CODECOPY
   GASPRICE
   EXTCODESIZE
   EXTCODECOPY
   RETURNDATASIZE
   RETURNDATACOPY
   EXTCODEHASH
)

// 0x40 range - block operations.
const (
   BLOCKHASH OpCode = 0x40 + iota
   COINBASE
   TIMESTAMP
   NUMBER
   DIFFICULTY
   GASLIMIT
)

// 0x50 range - 'storage' and execution.
const (
   POP OpCode = 0x50 + iota
   MLOAD
   MSTORE
   MSTORE8
   SLOAD
   SSTORE
   JUMP
   JUMPI
   PC
   MSIZE
   GAS
   JUMPDEST
)

// 0x60 range.
const (
   PUSH1 OpCode = 0x60 + iota
   PUSH2
   PUSH3
   PUSH4
   PUSH5
   PUSH6
   PUSH7
   PUSH8
   PUSH9
   PUSH10
   PUSH11
   PUSH12
   PUSH13
   PUSH14
   PUSH15
   PUSH16
   PUSH17
   PUSH18
   PUSH19
   PUSH20
   PUSH21
   PUSH22
   PUSH23
   PUSH24
   PUSH25
   PUSH26
   PUSH27
   PUSH28
   PUSH29
   PUSH30
   PUSH31
   PUSH32
   DUP1
   DUP2
   DUP3
   DUP4
   DUP5
   DUP6
   DUP7
   DUP8
   DUP9
   DUP10
   DUP11
   DUP12
   DUP13
   DUP14
   DUP15
   DUP16
   SWAP1
   SWAP2
   SWAP3
   SWAP4
   SWAP5
   SWAP6
   SWAP7
   SWAP8
   SWAP9
   SWAP10
   SWAP11
   SWAP12
   SWAP13
   SWAP14
   SWAP15
   SWAP16
)

// 0xa0 range - logging ops.
const (
   LOG0 OpCode = 0xa0 + iota
   LOG1
   LOG2
   LOG3
   LOG4
)

// unofficial opcodes used for parsing.
const (
   PUSH OpCode = 0xb0 + iota
   DUP
   SWAP
)
```

core/vm/jump_table.go：都是对操作码的映射解释，可以将jump理解为操作码解释器，将opcodes理解为指令解释器

```go
func newFrontierInstructionSet() [256]operation {
   return [256]operation{
      STOP: {
         execute:       opStop,
         gasCost:       constGasFunc(0),
         validateStack: makeStackFunc(0, 0),
         halts:         true,
         valid:         true,
      },
      ADD: {
         execute:       opAdd,
         gasCost:       constGasFunc(GasFastestStep),
         validateStack: makeStackFunc(2, 1),
         valid:         true,
      },
      MUL: {
         execute:       opMul,
         gasCost:       constGasFunc(GasFastStep),
         validateStack: makeStackFunc(2, 1),
         valid:         true,
      },
      SUB: {
         execute:       opSub,
         gasCost:       constGasFunc(GasFastestStep),
         validateStack: makeStackFunc(2, 1),
         valid:         true,
      },
      DIV: {
         execute:       opDiv,
         gasCost:       constGasFunc(GasFastStep),
         validateStack: makeStackFunc(2, 1),
         valid:         true,
      },
      SDIV: {
         execute:       opSdiv,
         gasCost:       constGasFunc(GasFastStep),
         validateStack: makeStackFunc(2, 1),
         valid:         true,
      },
      MOD: {
         execute:       opMod,
         gasCost:       constGasFunc(GasFastStep),
         validateStack: makeStackFunc(2, 1),
         valid:         true,
      },
      SMOD: {
         execute:       opSmod,
         gasCost:       constGasFunc(GasFastStep),
         validateStack: makeStackFunc(2, 1),
         valid:         true,
      },
      ADDMOD: {
         execute:       opAddmod,
         gasCost:       constGasFunc(GasMidStep),
         validateStack: makeStackFunc(3, 1),
         valid:         true,
      },
      MULMOD: {
         execute:       opMulmod,
         gasCost:       constGasFunc(GasMidStep),
         validateStack: makeStackFunc(3, 1),
         valid:         true,
      },
      EXP: {
         execute:       opExp,
         gasCost:       gasExp,
         validateStack: makeStackFunc(2, 1),
         valid:         true,
      },
      SIGNEXTEND: {
         execute:       opSignExtend,
         gasCost:       constGasFunc(GasFastStep),
         validateStack: makeStackFunc(2, 1),
         valid:         true,
      },
      LT: {
         execute:       opLt,
         gasCost:       constGasFunc(GasFastestStep),
         validateStack: makeStackFunc(2, 1),
         valid:         true,
      },
      GT: {
         execute:       opGt,
         gasCost:       constGasFunc(GasFastestStep),
         validateStack: makeStackFunc(2, 1),
         valid:         true,
      },
      SLT: {
         execute:       opSlt,
         gasCost:       constGasFunc(GasFastestStep),
         validateStack: makeStackFunc(2, 1),
         valid:         true,
      },
      SGT: {
         execute:       opSgt,
         gasCost:       constGasFunc(GasFastestStep),
         validateStack: makeStackFunc(2, 1),
         valid:         true,
      },
      EQ: {
         execute:       opEq,
         gasCost:       constGasFunc(GasFastestStep),
         validateStack: makeStackFunc(2, 1),
         valid:         true,
      },
      ISZERO: {
         execute:       opIszero,
         gasCost:       constGasFunc(GasFastestStep),
         validateStack: makeStackFunc(1, 1),
         valid:         true,
      },
      AND: {
         execute:       opAnd,
         gasCost:       constGasFunc(GasFastestStep),
         validateStack: makeStackFunc(2, 1),
         valid:         true,
      },
      XOR: {
         execute:       opXor,
         gasCost:       constGasFunc(GasFastestStep),
         validateStack: makeStackFunc(2, 1),
         valid:         true,
      },
      OR: {
         execute:       opOr,
         gasCost:       constGasFunc(GasFastestStep),
         validateStack: makeStackFunc(2, 1),
         valid:         true,
      },
      NOT: {
         execute:       opNot,
         gasCost:       constGasFunc(GasFastestStep),
         validateStack: makeStackFunc(1, 1),
         valid:         true,
      },
      BYTE: {
         execute:       opByte,
         gasCost:       constGasFunc(GasFastestStep),
         validateStack: makeStackFunc(2, 1),
         valid:         true,
      },
      SHA3: {
         execute:       opSha3,
         gasCost:       gasSha3,
         validateStack: makeStackFunc(2, 1),
         memorySize:    memorySha3,
         valid:         true,
      },
      ADDRESS: {
         execute:       opAddress,
         gasCost:       constGasFunc(GasQuickStep),
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      BALANCE: {
         execute:       opBalance,
         gasCost:       gasBalance,
         validateStack: makeStackFunc(1, 1),
         valid:         true,
      },
      ORIGIN: {
         execute:       opOrigin,
         gasCost:       constGasFunc(GasQuickStep),
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      CALLER: {
         execute:       opCaller,
         gasCost:       constGasFunc(GasQuickStep),
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      CALLVALUE: {
         execute:       opCallValue,
         gasCost:       constGasFunc(GasQuickStep),
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      CALLDATALOAD: {
         execute:       opCallDataLoad,
         gasCost:       constGasFunc(GasFastestStep),
         validateStack: makeStackFunc(1, 1),
         valid:         true,
      },
      CALLDATASIZE: {
         execute:       opCallDataSize,
         gasCost:       constGasFunc(GasQuickStep),
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      CALLDATACOPY: {
         execute:       opCallDataCopy,
         gasCost:       gasCallDataCopy,
         validateStack: makeStackFunc(3, 0),
         memorySize:    memoryCallDataCopy,
         valid:         true,
      },
      CODESIZE: {
         execute:       opCodeSize,
         gasCost:       constGasFunc(GasQuickStep),
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      CODECOPY: {
         execute:       opCodeCopy,
         gasCost:       gasCodeCopy,
         validateStack: makeStackFunc(3, 0),
         memorySize:    memoryCodeCopy,
         valid:         true,
      },
      GASPRICE: {
         execute:       opGasprice,
         gasCost:       constGasFunc(GasQuickStep),
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      EXTCODESIZE: {
         execute:       opExtCodeSize,
         gasCost:       gasExtCodeSize,
         validateStack: makeStackFunc(1, 1),
         valid:         true,
      },
      EXTCODECOPY: {
         execute:       opExtCodeCopy,
         gasCost:       gasExtCodeCopy,
         validateStack: makeStackFunc(4, 0),
         memorySize:    memoryExtCodeCopy,
         valid:         true,
      },
      BLOCKHASH: {
         execute:       opBlockhash,
         gasCost:       constGasFunc(GasExtStep),
         validateStack: makeStackFunc(1, 1),
         valid:         true,
      },
      COINBASE: {
         execute:       opCoinbase,
         gasCost:       constGasFunc(GasQuickStep),
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      TIMESTAMP: {
         execute:       opTimestamp,
         gasCost:       constGasFunc(GasQuickStep),
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      NUMBER: {
         execute:       opNumber,
         gasCost:       constGasFunc(GasQuickStep),
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      DIFFICULTY: {
         execute:       opDifficulty,
         gasCost:       constGasFunc(GasQuickStep),
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      GASLIMIT: {
         execute:       opGasLimit,
         gasCost:       constGasFunc(GasQuickStep),
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      POP: {
         execute:       opPop,
         gasCost:       constGasFunc(GasQuickStep),
         validateStack: makeStackFunc(1, 0),
         valid:         true,
      },
      MLOAD: {
         execute:       opMload,
         gasCost:       gasMLoad,
         validateStack: makeStackFunc(1, 1),
         memorySize:    memoryMLoad,
         valid:         true,
      },
      MSTORE: {
         execute:       opMstore,
         gasCost:       gasMStore,
         validateStack: makeStackFunc(2, 0),
         memorySize:    memoryMStore,
         valid:         true,
      },
      MSTORE8: {
         execute:       opMstore8,
         gasCost:       gasMStore8,
         memorySize:    memoryMStore8,
         validateStack: makeStackFunc(2, 0),

         valid: true,
      },
      SLOAD: {
         execute:       opSload,
         gasCost:       gasSLoad,
         validateStack: makeStackFunc(1, 1),
         valid:         true,
      },
      SSTORE: {
         execute:       opSstore,
         gasCost:       gasSStore,
         validateStack: makeStackFunc(2, 0),
         valid:         true,
         writes:        true,
      },
      JUMP: {
         execute:       opJump,
         gasCost:       constGasFunc(GasMidStep),
         validateStack: makeStackFunc(1, 0),
         jumps:         true,
         valid:         true,
      },
      JUMPI: {
         execute:       opJumpi,
         gasCost:       constGasFunc(GasSlowStep),
         validateStack: makeStackFunc(2, 0),
         jumps:         true,
         valid:         true,
      },
      PC: {
         execute:       opPc,
         gasCost:       constGasFunc(GasQuickStep),
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      MSIZE: {
         execute:       opMsize,
         gasCost:       constGasFunc(GasQuickStep),
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      GAS: {
         execute:       opGas,
         gasCost:       constGasFunc(GasQuickStep),
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      JUMPDEST: {
         execute:       opJumpdest,
         gasCost:       constGasFunc(params.JumpdestGas),
         validateStack: makeStackFunc(0, 0),
         valid:         true,
      },
      PUSH1: {
         execute:       makePush(1, 1),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH2: {
         execute:       makePush(2, 2),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH3: {
         execute:       makePush(3, 3),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH4: {
         execute:       makePush(4, 4),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH5: {
         execute:       makePush(5, 5),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH6: {
         execute:       makePush(6, 6),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH7: {
         execute:       makePush(7, 7),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH8: {
         execute:       makePush(8, 8),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH9: {
         execute:       makePush(9, 9),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH10: {
         execute:       makePush(10, 10),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH11: {
         execute:       makePush(11, 11),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH12: {
         execute:       makePush(12, 12),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH13: {
         execute:       makePush(13, 13),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH14: {
         execute:       makePush(14, 14),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH15: {
         execute:       makePush(15, 15),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH16: {
         execute:       makePush(16, 16),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH17: {
         execute:       makePush(17, 17),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH18: {
         execute:       makePush(18, 18),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH19: {
         execute:       makePush(19, 19),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH20: {
         execute:       makePush(20, 20),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH21: {
         execute:       makePush(21, 21),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH22: {
         execute:       makePush(22, 22),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH23: {
         execute:       makePush(23, 23),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH24: {
         execute:       makePush(24, 24),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH25: {
         execute:       makePush(25, 25),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH26: {
         execute:       makePush(26, 26),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH27: {
         execute:       makePush(27, 27),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH28: {
         execute:       makePush(28, 28),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH29: {
         execute:       makePush(29, 29),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH30: {
         execute:       makePush(30, 30),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH31: {
         execute:       makePush(31, 31),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      PUSH32: {
         execute:       makePush(32, 32),
         gasCost:       gasPush,
         validateStack: makeStackFunc(0, 1),
         valid:         true,
      },
      DUP1: {
         execute:       makeDup(1),
         gasCost:       gasDup,
         validateStack: makeDupStackFunc(1),
         valid:         true,
      },
      DUP2: {
         execute:       makeDup(2),
         gasCost:       gasDup,
         validateStack: makeDupStackFunc(2),
         valid:         true,
      },
      DUP3: {
         execute:       makeDup(3),
         gasCost:       gasDup,
         validateStack: makeDupStackFunc(3),
         valid:         true,
      },
      DUP4: {
         execute:       makeDup(4),
         gasCost:       gasDup,
         validateStack: makeDupStackFunc(4),
         valid:         true,
      },
      DUP5: {
         execute:       makeDup(5),
         gasCost:       gasDup,
         validateStack: makeDupStackFunc(5),
         valid:         true,
      },
      DUP6: {
         execute:       makeDup(6),
         gasCost:       gasDup,
         validateStack: makeDupStackFunc(6),
         valid:         true,
      },
      DUP7: {
         execute:       makeDup(7),
         gasCost:       gasDup,
         validateStack: makeDupStackFunc(7),
         valid:         true,
      },
      DUP8: {
         execute:       makeDup(8),
         gasCost:       gasDup,
         validateStack: makeDupStackFunc(8),
         valid:         true,
      },
      DUP9: {
         execute:       makeDup(9),
         gasCost:       gasDup,
         validateStack: makeDupStackFunc(9),
         valid:         true,
      },
      DUP10: {
         execute:       makeDup(10),
         gasCost:       gasDup,
         validateStack: makeDupStackFunc(10),
         valid:         true,
      },
      DUP11: {
         execute:       makeDup(11),
         gasCost:       gasDup,
         validateStack: makeDupStackFunc(11),
         valid:         true,
      },
      DUP12: {
         execute:       makeDup(12),
         gasCost:       gasDup,
         validateStack: makeDupStackFunc(12),
         valid:         true,
      },
      DUP13: {
         execute:       makeDup(13),
         gasCost:       gasDup,
         validateStack: makeDupStackFunc(13),
         valid:         true,
      },
      DUP14: {
         execute:       makeDup(14),
         gasCost:       gasDup,
         validateStack: makeDupStackFunc(14),
         valid:         true,
      },
      DUP15: {
         execute:       makeDup(15),
         gasCost:       gasDup,
         validateStack: makeDupStackFunc(15),
         valid:         true,
      },
      DUP16: {
         execute:       makeDup(16),
         gasCost:       gasDup,
         validateStack: makeDupStackFunc(16),
         valid:         true,
      },
      SWAP1: {
         execute:       makeSwap(1),
         gasCost:       gasSwap,
         validateStack: makeSwapStackFunc(2),
         valid:         true,
      },
      SWAP2: {
         execute:       makeSwap(2),
         gasCost:       gasSwap,
         validateStack: makeSwapStackFunc(3),
         valid:         true,
      },
      SWAP3: {
         execute:       makeSwap(3),
         gasCost:       gasSwap,
         validateStack: makeSwapStackFunc(4),
         valid:         true,
      },
      SWAP4: {
         execute:       makeSwap(4),
         gasCost:       gasSwap,
         validateStack: makeSwapStackFunc(5),
         valid:         true,
      },
      SWAP5: {
         execute:       makeSwap(5),
         gasCost:       gasSwap,
         validateStack: makeSwapStackFunc(6),
         valid:         true,
      },
      SWAP6: {
         execute:       makeSwap(6),
         gasCost:       gasSwap,
         validateStack: makeSwapStackFunc(7),
         valid:         true,
      },
      SWAP7: {
         execute:       makeSwap(7),
         gasCost:       gasSwap,
         validateStack: makeSwapStackFunc(8),
         valid:         true,
      },
      SWAP8: {
         execute:       makeSwap(8),
         gasCost:       gasSwap,
         validateStack: makeSwapStackFunc(9),
         valid:         true,
      },
      SWAP9: {
         execute:       makeSwap(9),
         gasCost:       gasSwap,
         validateStack: makeSwapStackFunc(10),
         valid:         true,
      },
      SWAP10: {
         execute:       makeSwap(10),
         gasCost:       gasSwap,
         validateStack: makeSwapStackFunc(11),
         valid:         true,
      },
      SWAP11: {
         execute:       makeSwap(11),
         gasCost:       gasSwap,
         validateStack: makeSwapStackFunc(12),
         valid:         true,
      },
      SWAP12: {
         execute:       makeSwap(12),
         gasCost:       gasSwap,
         validateStack: makeSwapStackFunc(13),
         valid:         true,
      },
      SWAP13: {
         execute:       makeSwap(13),
         gasCost:       gasSwap,
         validateStack: makeSwapStackFunc(14),
         valid:         true,
      },
      SWAP14: {
         execute:       makeSwap(14),
         gasCost:       gasSwap,
         validateStack: makeSwapStackFunc(15),
         valid:         true,
      },
      SWAP15: {
         execute:       makeSwap(15),
         gasCost:       gasSwap,
         validateStack: makeSwapStackFunc(16),
         valid:         true,
      },
      SWAP16: {
         execute:       makeSwap(16),
         gasCost:       gasSwap,
         validateStack: makeSwapStackFunc(17),
         valid:         true,
      },
      LOG0: {
         execute:       makeLog(0),
         gasCost:       makeGasLog(0),
         validateStack: makeStackFunc(2, 0),
         memorySize:    memoryLog,
         valid:         true,
         writes:        true,
      },
      LOG1: {
         execute:       makeLog(1),
         gasCost:       makeGasLog(1),
         validateStack: makeStackFunc(3, 0),
         memorySize:    memoryLog,
         valid:         true,
         writes:        true,
      },
      LOG2: {
         execute:       makeLog(2),
         gasCost:       makeGasLog(2),
         validateStack: makeStackFunc(4, 0),
         memorySize:    memoryLog,
         valid:         true,
         writes:        true,
      },
      LOG3: {
         execute:       makeLog(3),
         gasCost:       makeGasLog(3),
         validateStack: makeStackFunc(5, 0),
         memorySize:    memoryLog,
         valid:         true,
         writes:        true,
      },
      LOG4: {
         execute:       makeLog(4),
         gasCost:       makeGasLog(4),
         validateStack: makeStackFunc(6, 0),
         memorySize:    memoryLog,
         valid:         true,
         writes:        true,
      },
      CREATE: {
         execute:       opCreate,
         gasCost:       gasCreate,
         validateStack: makeStackFunc(3, 1),
         memorySize:    memoryCreate,
         valid:         true,
         writes:        true,
         returns:       true,
      },
      CALL: {
         execute:       opCall,
         gasCost:       gasCall,
         validateStack: makeStackFunc(7, 1),
         memorySize:    memoryCall,
         valid:         true,
         returns:       true,
      },
      CALLCODE: {
         execute:       opCallCode,
         gasCost:       gasCallCode,
         validateStack: makeStackFunc(7, 1),
         memorySize:    memoryCall,
         valid:         true,
         returns:       true,
      },
      RETURN: {
         execute:       opReturn,
         gasCost:       gasReturn,
         validateStack: makeStackFunc(2, 0),
         memorySize:    memoryReturn,
         halts:         true,
         valid:         true,
      },
      SELFDESTRUCT: {
         execute:       opSuicide,
         gasCost:       gasSuicide,
         validateStack: makeStackFunc(1, 0),
         halts:         true,
         valid:         true,
         writes:        true,
      },
   }
}
```

> 吐槽一句:知道负整数需要花费多少gasg码
>
> ```go
> ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
> ```
>
> 越大的负整数（`-1`大于`-2`）1越多，会花费比正整数多得多的gas。
>
> 这是部分gas规则：
>
> - 每笔交易需要支付 21000 gas
> - 每笔交易的0字节或代码需要支付 4 gas
> - 每笔交易的非0字节或代码需要支付 68 gas
>
> 所以当我们在abi中看到很多0的时候不要惊讶,因为0比其它的价格要低18倍
>
> 我们来来看看gas设置的源码：
>
> ```go
> func gasMStore8(gt params.GasTable, evm *EVM, contract *Contract, stack *Stack, mem *Memory, memorySize uint64) (uint64, error) {
>    var overflow bool
>    gas, err := memoryGasCost(mem, memorySize)
>    if err != nil {
>       return 0, errGasUintOverflow
>    }
>    if gas, overflow = math.SafeAdd(gas, GasFastestStep); overflow {
>       return 0, errGasUintOverflow
>    }
>    return gas, nil
> }
> 
> func gasMStore(gt params.GasTable, evm *EVM, contract *Contract, stack *Stack, mem *Memory, memorySize uint64) (uint64, error) {
>    var overflow bool
>    gas, err := memoryGasCost(mem, memorySize)
>    if err != nil {
>       return 0, errGasUintOverflow
>    }
>    if gas, overflow = math.SafeAdd(gas, GasFastestStep); overflow {
>       return 0, errGasUintOverflow
>    }
>    return gas, nil
> }
> 
> func gasCreate(gt params.GasTable, evm *EVM, contract *Contract, stack *Stack, mem *Memory, memorySize uint64) (uint64, error) {
>    var overflow bool
>    gas, err := memoryGasCost(mem, memorySize)
>    if err != nil {
>       return 0, err
>    }
>    if gas, overflow = math.SafeAdd(gas, params.CreateGas); overflow {
>       return 0, errGasUintOverflow
>    }
>    return gas, nil
> }
> ```
>
> 这是部分计算gas价格的源码：从上面可以看到价格的具体计算过程，错误好像大多数都是整数溢出的错误
>
> 我们最后在看一下解释器：解释器用于运行基于Ethereum的合同，并将使用通过环境查询外部源的状态信息。解释器将根据传递的字节码运行VM配置。
>
> ```go
> 据结构
>     
>     // Config are the configuration options for the Interpreter
>     type Config struct {
>         // Debug enabled debugging Interpreter options
>         Debug bool
>         // EnableJit enabled the JIT VM
>         EnableJit bool
>         // ForceJit forces the JIT VM
>         ForceJit bool
>         // Tracer is the op code logger
>         Tracer Tracer
>         // NoRecursion disabled Interpreter call, callcode,
>         // delegate call and create.
>         NoRecursion bool
>         // Disable gas metering
>         DisableGasMetering bool
>         // Enable recording of SHA3/keccak preimages
>         EnablePreimageRecording bool
>         // JumpTable contains the EVM instruction table. This
>         // may be left uninitialised and will be set to the default
>         // table.
>         JumpTable [256]operation
>     }
>     
>     // Interpreter is used to run Ethereum based contracts and will utilise the
>     // passed evmironment to query external sources for state information.
>     // The Interpreter will run the byte code VM or JIT VM based on the passed
>     // configuration.
>     type Interpreter struct {
>         evm      *EVM
>         cfg      Config
>         gasTable params.GasTable   // 标识了很多操作的Gas价格
>         intPool  *intPool
>     
>         readOnly   bool   // Whether to throw on stateful modifications
>         returnData []byte // Last CALL's return data for subsequent reuse 最后一个函数的返回值
>     }
> 
> 构造函数
>     
>     // NewInterpreter returns a new instance of the Interpreter.
>     func NewInterpreter(evm *EVM, cfg Config) *Interpreter {
>         // We use the STOP instruction whether to see
>         // the jump table was initialised. If it was not
>         // we'll set the default jump table.
>         // 用一个STOP指令测试JumpTable是否已经被初始化了, 如果没有被初始化,那么设置为默认值
>         if !cfg.JumpTable[STOP].valid { 
>             switch {
>             case evm.ChainConfig().IsByzantium(evm.BlockNumber):
>                 cfg.JumpTable = byzantiumInstructionSet
>             case evm.ChainConfig().IsHomestead(evm.BlockNumber):
>                 cfg.JumpTable = homesteadInstructionSet
>             default:
>                 cfg.JumpTable = frontierInstructionSet
>             }
>         }
>     
>         return &Interpreter{
>             evm:      evm,
>             cfg:      cfg,
>             gasTable: evm.ChainConfig().GasTable(evm.BlockNumber),
>             intPool:  newIntPool(),
>         }
>     }
> 
> 
> 解释器一共就两个方法enforceRestrictions方法和Run方法.
> 
> 
>     
>     func (in *Interpreter) enforceRestrictions(op OpCode, operation operation, stack *Stack) error {
>         if in.evm.chainRules.IsByzantium {
>             if in.readOnly {
>                 // If the interpreter is operating in readonly mode, make sure no
>                 // state-modifying operation is performed. The 3rd stack item
>                 // for a call operation is the value. Transferring value from one
>                 // account to the others means the state is modified and should also
>                 // return with an error.
>                 if operation.writes || (op == CALL && stack.Back(2).BitLen() > 0) {
>                     return errWriteProtection
>                 }
>             }
>         }
>         return nil
>     }
>     
>     // Run loops and evaluates the contract's code with the given input data and returns
>     // the return byte-slice and an error if one occurred.
>     // 用给定的入参循环执行合约的代码，并返回返回的字节片段，如果发生错误则返回错误。
>     // It's important to note that any errors returned by the interpreter should be
>     // considered a revert-and-consume-all-gas operation. No error specific checks
>     // should be handled to reduce complexity and errors further down the in.
>     // 重要的是要注意，解释器返回的任何错误都会消耗全部gas。 为了减少复杂性,没有特别的错误处理流程。
>     func (in *Interpreter) Run(snapshot int, contract *Contract, input []byte) (ret []byte, err error) {
>         // Increment the call depth which is restricted to 1024
>         in.evm.depth++
>         defer func() { in.evm.depth-- }()
>     
>         // Reset the previous call's return data. It's unimportant to preserve the old buffer
>         // as every returning call will return new data anyway.
>         in.returnData = nil
>     
>         // Don't bother with the execution if there's no code.
>         if len(contract.Code) == 0 {
>             return nil, nil
>         }
>     
>         codehash := contract.CodeHash // codehash is used when doing jump dest caching
>         if codehash == (common.Hash{}) {
>             codehash = crypto.Keccak256Hash(contract.Code)
>         }
>     
>         var (
>             op    OpCode        // current opcode
>             mem   = NewMemory() // bound memory
>             stack = newstack()  // local stack
>             // For optimisation reason we're using uint64 as the program counter.
>             // It's theoretically possible to go above 2^64. The YP defines the PC
>             // to be uint256. Practically much less so feasible.
>             pc   = uint64(0) // program counter
>             cost uint64
>             // copies used by tracer
>             stackCopy = newstack() // stackCopy needed for Tracer since stack is mutated by 63/64 gas rule 
>             pcCopy uint64 // needed for the deferred Tracer
>             gasCopy uint64 // for Tracer to log gas remaining before execution
>             logged bool // deferred Tracer should ignore already logged steps
>         )
>         contract.Input = input
>     
>         defer func() {
>             if err != nil && !logged && in.cfg.Debug {
>                 in.cfg.Tracer.CaptureState(in.evm, pcCopy, op, gasCopy, cost, mem, stackCopy, contract, in.evm.depth, err)
>             }
>         }()
>     
>         // The Interpreter main run loop (contextual). This loop runs until either an
>         // explicit STOP, RETURN or SELFDESTRUCT is executed, an error occurred during
>         // the execution of one of the operations or until the done flag is set by the
>         // parent context.
>         // 解释器的主要循环， 直到遇到STOP，RETURN，SELFDESTRUCT指令被执行，或者是遇到任意错误，或者说done 标志被父context设置。
>         for atomic.LoadInt32(&in.evm.abort) == 0 {
>             // Get the memory location of pc
>             // 难道下一个需要执行的指令
>             op = contract.GetOp(pc)
>     
>             if in.cfg.Debug {
>                 logged = false
>                 pcCopy = uint64(pc)
>                 gasCopy = uint64(contract.Gas)
>                 stackCopy = newstack()
>                 for _, val := range stack.data {
>                     stackCopy.push(val)
>                 }
>             }
>     
>             // get the operation from the jump table matching the opcode
>             // 通过JumpTable拿到对应的operation
>             operation := in.cfg.JumpTable[op]
>             // 这里检查了只读模式下面不能执行writes指令
>             // staticCall的情况下会设置为readonly模式
>             if err := in.enforceRestrictions(op, operation, stack); err != nil {
>                 return nil, err
>             }
>     
>             // if the op is invalid abort the process and return an error
>             if !operation.valid { //检查指令是否非法
>                 return nil, fmt.Errorf("invalid opcode 0x%x", int(op))
>             }
>     
>             // validate the stack and make sure there enough stack items available
>             // to perform the operation
>             // 检查是否有足够的堆栈空间。 包括入栈和出栈
>             if err := operation.validateStack(stack); err != nil {
>                 return nil, err
>             }
>     
>             var memorySize uint64
>             // calculate the new memory size and expand the memory to fit
>             // the operation
>             if operation.memorySize != nil { // 计算内存使用量，需要收费
>                 memSize, overflow := bigUint64(operation.memorySize(stack))
>                 if overflow {
>                     return nil, errGasUintOverflow
>                 }
>                 // memory is expanded in words of 32 bytes. Gas
>                 // is also calculated in words.
>                 if memorySize, overflow = math.SafeMul(toWordSize(memSize), 32); overflow {
>                     return nil, errGasUintOverflow
>                 }
>             }
>     
>             if !in.cfg.DisableGasMetering { //这个参数在本地模拟执行的时候比较有用，可以不消耗或者检查GAS执行交易并得到返回结果
>                 // consume the gas and return an error if not enough gas is available.
>                 // cost is explicitly set so that the capture state defer method cas get the proper cost
>                 // 计算gas的Cost 并使用，如果不够，就返回OutOfGas错误。
>                 cost, err = operation.gasCost(in.gasTable, in.evm, contract, stack, mem, memorySize)
>                 if err != nil || !contract.UseGas(cost) {
>                     return nil, ErrOutOfGas
>                 }
>             }
>             if memorySize > 0 { //扩大内存范围
>                 mem.Resize(memorySize)
>             }
>     
>             if in.cfg.Debug {
>                 in.cfg.Tracer.CaptureState(in.evm, pc, op, gasCopy, cost, mem, stackCopy, contract, in.evm.depth, err)
>                 logged = true
>             }
>     
>             // execute the operation
>             // 执行命令
>             res, err := operation.execute(&pc, in.evm, contract, mem, stack)
>             // verifyPool is a build flag. Pool verification makes sure the integrity
>             // of the integer pool by comparing values to a default value.
>             if verifyPool {
>                 verifyIntegerPool(in.intPool)
>             }
>             // if the operation clears the return data (e.g. it has returning data)
>             // set the last return to the result of the operation.
>             if operation.returns { //如果有返回值，那么就设置返回值。 注意只有最后一个返回有效果。
>                 in.returnData = res
>             }
>     
>             switch {
>             case err != nil:
>                 return nil, err
>             case operation.reverts:
>                 return res, errExecutionReverted
>             case operation.halts:
>                 return res, nil
>             case !operation.jumps:
>                 pc++
>             }
>         }
>         return nil, nil
>     }
> 
> ```

总结：与智能合约交互，你需要发送原始字节。他会进行一些映射，然后进行运算。方法调用实际上不存在，这是ABI创造的一个很有意思假象，然我们以为自己在调用方法，实际上只是执行了一段指令。

