## Intro


A collection of EVM puzzles. Each puzzle consists on sending a successful transaction to a contract. The bytecode of the contract is provided, and you must fill the transaction data that won't revert the execution.

Credit for the challenges goes to [@fvictorio_nan](https://twitter.com/fvictorio_nan).

Source code with installation instructions and how to play can be found [here](https://github.com/fvictorio/evm-puzzles).

I'll use the [evm.code](https://www.evm.codes/) website to debug the bytecode and for documentation references.


## Puzzle 1

evm.code bytecode challenge: [link](https://www.evm.codes/playground?callValue=&unit=Wei&callData=&codeType=Bytecode&code=%273456FDFDFDFDFDFD5B00%27_).

```
PC    OPCODE    NAME
-----------------------
00      34      CALLVALUE
01      56      JUMP
02      FD      REVERT
03      FD      REVERT
04      FD      REVERT
05      FD      REVERT
06      FD      REVERT
07      FD      REVERT
08      5B      JUMPDEST
09      00      STOP     
```

Bytecode analysis:
- `CALLVALUE` gets the transaction value (in wei) sent to this contract and pushes it on top of the stack
- `JUMP` jumps to the instruction at the offset value stored on top of the stack

Our goal is to send a transaction value (in wei) such that we reach the `JUMPDEST` opcode and not revert the execution.

The `JUMP` instruction will jump at a location equal to the transaction value sent (`CALLVALUE`).
Since the instruction `JUMPDEST` is at PC `08`, the value that must be stored on top of the stack when executing the `JUMP` instruction is `8`.

In other words, we need to send `8` wei to solve this challenge.

### Solution

```json
{"value":8,"data":"0x"}
```

evm.code bytecode solution challenge: [link](https://www.evm.codes/playground?callValue=8&unit=Wei&callData=&codeType=Bytecode&code='3456FDFDFDFDFDFD5B00'_).


## Puzzle 2

evm.code bytecode challenge: [link](https://www.evm.codes/playground?callValue=&unit=Wei&callData=&codeType=Bytecode&code='34380356FDFD5B00FDFD'_).

```
PC    OPCODE    NAME
-----------------------
00      34      CALLVALUE
01      38      CODESIZE
02      03      SUB
03      56      JUMP
04      FD      REVERT
05      FD      REVERT
06      5B      JUMPDEST
07      00      STOP
08      FD      REVERT
09      FD      REVERT
```

Bytecode analysis (opcodes unseen from previous challenges):
- `CODESIZE` pushes the size of the contract code in bytes on top of the stack
- `SUB` pops the first two values from the stack, subtracts the second from the first one, and pushes the result

This puzzle has `10` instructions (each instruction is `1` byte), so the `CODESIZE` will push `0a` on top of the stack when executed. `SUB` will compute the difference between `0a` and the `msg.value` (output of `CALLVALUE`).

The `JUMP` instruction will jump at the location returned by `SUB`. Since the `JUMPDEST` instruction is at location `6`, we need to solve the following:
```math
0a - x = 6
```
```math
x = 0a - 6
```
where $x$ is the amount (in wei) sent to this contract (i.e. `msg.value`).

If we convert the hex values to decimal, this translates to:
```math
x = 10 - 6 = 4
```

To reach the `JUMPDEST` instruction, we need to send `4` wei.

### Solution

```json
{"value":4,"data":"0x"}
```
evm.code bytecode solution challenge: [link](https://www.evm.codes/playground?callValue=4&unit=Wei&callData=&codeType=Bytecode&code='34380356FDFD5B00FDFD'_).


## Puzzle 3

evm.code bytecode challenge: [link](https://www.evm.codes/playground?callValue=&unit=Wei&callData=&codeType=Bytecode&code='3656FDFD5B00'_).

```
PC    OPCODE    NAME
-----------------------
00      36      CALLDATASIZE
01      56      JUMP
02      FD      REVERT
03      FD      REVERT
04      5B      JUMPDEST
05      00      STOP
```

Bytecode analysis (opcodes unseen from previous challenges):
- `CALLDATASIZE` pushes the size of the `calldata` on top of the stack


What is the `calladata`?


The evm.codes website provide a simple and clear explanation ([link](https://www.evm.codes/about#calldata)):
> The call data region is the data that is sent with a transaction. In the case of contract creation, it would be the constructor code. This region is immutable and can be read with the instructions CALLDATALOAD, CALLDATASIZE, and CALLDATACOPY.

This puzzle is similar to [Puzzle 1](#puzzle-1). The only difference is that the value pushed on top of the stack is not the `msg.value` but the size of the `calldata`. The `JUMPDEST` instruction is at PC `04`, so we need to send a `calldata` of size `4` bytes, for example, `0x01010101`.

### Solution

```json
{"data":"0x01010101","value":0}
```

evm.code bytecode solution challenge: [link](https://www.evm.codes/playground?callValue=0&unit=Wei&callData=0x01010101&codeType=Bytecode&code='3656FDFD5B00'_)


## Puzzle 4

evm.code bytecode challenge: [link](https://www.evm.codes/playground?callValue=&unit=Wei&callData=&codeType=Bytecode&code='34381856FDFDFDFDFDFD5B00'_).

```
PC    OPCODE    NAME
-----------------------
00      34      CALLVALUE
01      38      CODESIZE
02      18      XOR
03      56      JUMP
04      FD      REVERT
05      FD      REVERT
06      FD      REVERT
07      FD      REVERT
08      FD      REVERT
09      FD      REVERT
0A      5B      JUMPDEST
0B      00      STOP
```


Bytecode analysis (opcodes unseen from previous challenges):
- `XOR` pops the first two values from the stack, computes their bitwise XOR, and pushes the result

**XOR Table**

| x	| y	| x^y |
|---|---|-----|
|0	| 0	| 0 | 
|0	| 1	| 1 |
|1	| 0	| 1 |
|1	| 1	| 0 |


`CALLVALUE` pushes the `msg.value` on top of the stack. The next instruction, `CODESIZE` pushes the code size on top of the stack. We know this value will be `0c` (`12` in decimal) since our puzzle code has `12` instructions. `JUMP` will use the result from the `XOR` instruction to jump to the next instruction. We want to reach the location of `JUMPDEST` at PC `0a` (`10` in decimal). It means the result of the `XOR` instruction must be `0a`.

The only unknown is the `CALLVALUE` (i.e. the `msg.value`). The formula we want to compute, 
where $x$ is the value returned by `CALLVALUE`, is the following:
```math
0c \oplus x = 0a
```
```math
x = 0a \oplus 0c
```

```math
    \text{Hex} \quad \text{Decimal} \quad \text{Binary}
```
```math
    \quad\quad\quad 0a \quad\quad 10 \quad\quad\quad 1010 \quad\quad \oplus
```
```math
    \quad\quad\quad 0c \quad\quad 12 \quad\quad\quad 1100 \quad\quad =
```
```math
    \quad\quad--------------
```
```math
    \quad\quad\quad 06 \quad\quad 06 \quad\quad\quad 0110 \quad\quad \,\,\,\,\,\,
```

The amount of wei we need to send is `6`.


### Solution

```json
{"value":6,"data":"0x"}
```

evm.code bytecode solution challenge: [link](https://www.evm.codes/playground?callValue=6&unit=Wei&callData=&codeType=Bytecode&code='34381856FDFDFDFDFDFD5B00'_)


## Puzzle 5

evm.code bytecode challenge: [link](https://www.evm.codes/playground?callValue=&unit=Wei&callData=&codeType=Bytecode&code='34800261010014600C57FDFD5B00FDFD'_).

```
PC    OPCODE    NAME
-----------------------
00      34      CALLVALUE
01      80      DUP1
02      02      MUL
03      610100  PUSH2 0100
06      14      EQ
07      600C    PUSH1 0C
09      57      JUMPI
0A      FD      REVERT
0B      FD      REVERT
0C      5B      JUMPDEST
0D      00      STOP
0E      FD      REVERT
0F      FD      REVERT
```

Bytecode analysis (opcodes unseen from previous challenges):
- `DUP1` duplicates the value on top of the stack and pushes it on top of the stack
- `MUL` pops the first two values from the stack and pushes the result of the multiplication of these two numbers
- `PUSH2` pushes two bytes on top of the stack
- `EQ` pops the first two values from the stack and pushes `1` if the values are equal, `0` otherwise
- `PUSH1` pushes one byte on top of the stack
- `JUMPI` jumps to the instruction on top of the stack if the second value is different from `0`

The location of `JUMPDEST` at PC `0c` (`12` in decimal). This value is pushed on top of the stack by `PUSH1 0C`. `JUMPI` will jump at that location if the second value on top of the stack differs from `0`. It means the `EQ` instruction must return `1`, which is possible only if the output of the `MUL` instruction and `PUSH2 0100` are equal. The output of `MUL` is, in turn, computed by multiplying the value of `msg.value` by himself (because of the `DUP` instruction).

The formula we want to compute to obtain $1$ from `EQ` instruction, 
where $x$ is the value returned by `CALLVALUE`, is the following:

```math
x * x = 0100
```
```math
x^2 = 256
```
```math
x  = \sqrt256
```
```math
x  = 16
```

The amount of wei we need to send is `16` (`10` in hexadecimal).

### Solution

```json
{"value":16,"data":"0x"}
```

evm.code bytecode solution challenge: [link](https://www.evm.codes/playground?callValue=16&unit=Wei&callData=&codeType=Bytecode&code='34800261010014600C57FDFD5B00FDFD'_)


## Puzzle 6

evm.code bytecode challenge: [link](https://www.evm.codes/playground?callValue=&unit=Wei&callData=&codeType=Bytecode&code='60003556FDFDFDFDFDFD5B00'_).

```
PC    OPCODE    NAME
-----------------------
00      6000    PUSH1 00
02      35      CALLDATALOAD
03      56      JUMP
04      FD      REVERT
05      FD      REVERT
06      FD      REVERT
07      FD      REVERT
08      FD      REVERT
09      FD      REVERT
0A      5B      JUMPDEST
0B      00      STOP
```

Bytecode analysis (opcodes unseen from previous challenges):
- `PUSH1 00` pushes `00` on top of the stack
- `CALLDATALOAD` pops the offset of the `calldata` to read from the stack and pushes this data on top of the stack

The location of `JUMPDEST` at PC `0a`. `PUSH1 00` sets the offset that the `CALLDATALOAD` instruction will use. Since we need to jump at location `0a` and the `calldata` is read at offset `0`, we need to send a `calldata` of `32` bytes with all `0` except for the right-most byte that will be the exact value of `JUMPDEST` location, that is `0a`.

The `calldata` we need to send is: `0x000000000000000000000000000000000000000000000000000000000000000A`.

### Solution

```json
{"data":"0x000000000000000000000000000000000000000000000000000000000000000A","value":0}
```

evm.code bytecode solution challenge: [link](https://www.evm.codes/playground?callValue=0&unit=Wei&callData=0x000000000000000000000000000000000000000000000000000000000000000A&codeType=Bytecode&code='60003556FDFDFDFDFDFD5B00'_)


## Puzzle 7

evm.code bytecode challenge: [link](https://www.evm.codes/playground?callValue=&unit=Wei&callData=&codeType=Bytecode&code='36600080373660006000F03B600114601357FD5B00'_).

```
PC    OPCODE    NAME
-----------------------
00      36      CALLDATASIZE
01      6000    PUSH1 00
03      80      DUP1
04      37      CALLDATACOPY
05      36      CALLDATASIZE
06      6000    PUSH1 00
08      6000    PUSH1 00
0A      F0      CREATE
0B      3B      EXTCODESIZE
0C      6001    PUSH1 01
0E      14      EQ
0F      6013    PUSH1 13
11      57      JUMPI
12      FD      REVERT
13      5B      JUMPDEST
14      00      STOP
```

Bytecode analysis (opcodes unseen from previous challenges):
- `CALLDATACOPY` copies transaction data (`calldata`) into `memory`. It uses `3` values from the stack:         
    - `destOffset` is the bytes offset in `memory` where the `calldata` data will be copied
    - `offset` is the bytes offset of the `calldata` from where we want to start copying the data
    - `size` is the byte size of the `calldata` to be copied in `memory`
- `CREATE` is used to create a new contract. Like `CALLDATACOPY`, it uses `3` values from the stack: 
    - `value` is the amount of wei we want to send to the contact
    - `offset` is the bytes offset in the `memory` where to start coping contract code
    - `size` is the size of the initialization code (i.e. the bytes we want to copy from `memory` starting from the `offset`). 
    This instruction pushes on top of the stack the address of the contract deployed or `0` if it fails
- `EXTCODESIZE` pops a 20-bytes address from the stack; as a result, it pushes the byte size of the contract code

`JUMPDEST` is at PC `13` (value pushed on the stack by `PUSH1 13`). `JUMPI` is used to jump to the `JUMPDEST` location if the value of the second element on the stack is different from `0`. This value is the result of the following OP codes:
```
PC    OPCODE    NAME
-----------------------
0B      3B      EXTCODESIZE
0C      6001    PUSH1 01
0E      14      EQ
0F      6013    PUSH1 13
```
`EQ` compares `01` and the output of `EXTCODESIZE` and returns `1` if they are equal, `0` otherwise. We want this value to be different from `0`, so the output of `EXTCODESIZE` must be `1`.

`EXTCODESIZE` reads the 20 bytes address returned by `CREATE` and returns the byte size of that contract code. So we need to create a contract that, when deployed, returns only `1` instruction (`1` bytecode size).

We can use the following code that uses the `RETURN` instruction to return only `1` instruction: `0x60016000F3`:
```
OPCODE   NAME
-----------------------
6001     PUSH 01 (size of the data we want to copy)
6000     PUSH 00 (where we want to start copying the data from memory)
F3       RETURN
```

`RETURN` pops two values from the stack: 
- bytes `offset` where we want to start copying the data from memory
- bytes `size` of the data we want to copy

Since the `memory` is initialized to `0` by default ([docs](https://www.evm.codes/about#memory)), in the above code, the `RETURN` instruction will return `1` byte from memory (starting at offset `0`), that is `00` that corresponds to the `STOP` instruction.

### Solution

```json
{"data":"0x60016000F3","value":0}
```

evm.code bytecode solution challenge: [link](https://www.evm.codes/playground?callValue=0&unit=Wei&callData=0x60016000F3&codeType=Bytecode&code='36600080373660006000F03B600114601357FD5B00'_)


## Puzzle 8

evm.code bytecode challenge: [link](https://www.evm.codes/playground?callValue=&unit=Wei&callData=&codeType=Bytecode&code='36600080373660006000F0600080808080945AF1600014601B57FD5B00'_).

```
PC    OPCODE    NAME
-----------------------
00      36      CALLDATASIZE
01      6000    PUSH1 00
03      80      DUP1
04      37      CALLDATACOPY
05      36      CALLDATASIZE
06      6000    PUSH1 00
08      6000    PUSH1 00
0A      F0      CREATE
0B      6000    PUSH1 00
0D      80      DUP1
0E      80      DUP1
0F      80      DUP1
10      80      DUP1
11      94      SWAP5
12      5A      GAS
13      F1      CALL
14      6000    PUSH1 00
16      14      EQ
17      601B    PUSH1 1B
19      57      JUMPI
1A      FD      REVERT
1B      5B      JUMPDEST
1C      00      STOP
```

Bytecode analysis (opcodes unseen from previous challenges):
- `SWAP5` swaps the value at the top of the stack with the value at position `5` (i.e. `1 0 0 0 0 0 2` -> `2 0 0 0 0 0 1`)
- `GAS` pushes the gas remaining after this instruction
- `CALL`  calls a contract at a given address. It returns `0` if the call succeeds, `0` otherwise. It accepts different values:
    - `gas` is the gas to send during the call
    - `address` is the address where the call will execute the call.
    - `value` is the amount of wei sent
    - `argsOffset` is the byte offset in the memory in bytes
    - `argsSize` is the byte size to copy
    - `retOffset` is the byte offset in the memory in bytes, where to store the return data 
    - `retSize` is the byte size to copy

Let's do the same reasoning as the previous challenge. `JUMPDEST` is at PC `1B` (value pushed on the stack by `PUSH1 1B`). `JUMPI` is used to jump to the `JUMPDEST` location if the value of the second element on the stack is different from `0`. This value is the result of the following OP codes:
```
PC    OPCODE    NAME
-----------------------
13      F1      CALL
14      6000    PUSH1 00
16      14      EQ
17      601B    PUSH1 1B
```

`EQ` compares `00` and the output of `CALL` and returns `1` if they are equal, `0` otherwise. We want this value to be different from `0`, so the output of `CALL` must be `00`. It means we want the call to our contract to revert (`CALL` must return `0`).

To create a contract that returns the `REVERT` instruction, we can use the following bytecode (similar to the previous challenge):
```
OPCODE   NAME
-----------------------
60FD     PUSH FD (REVERT instruction)
6000     PUSH 00 (offset in memory)
53       MSTORE8 (stores REVERT instruction starting at offset `00`)
6001     PUSH 01 (size of the data we want to copy)
6000     PUSH 00 (where we want to start copying the data from memory)
F3       RETURN
```

The first three instructions above push the `FD` instruction and load it into memory. The `REVERT` instruction will be the code of our contract. When the contract is called, it will revert; thus, the return value from the `CALL` instruction will be `0`.

### Solution

```json
{"data":"0x60FD60005360016000F3","value":0}
```

evm.code bytecode solution challenge: [link](https://www.evm.codes/playground?callValue=0&unit=Wei&callData=0x60FD60005360016000F3&codeType=Bytecode&code='36600080373660006000F0600080808080945AF1600014601B57FD5B00'_)


## Puzzle 9

evm.code bytecode challenge: [link](https://www.evm.codes/playground?callValue=&unit=Wei&callData=&codeType=Bytecode&code='36600310600957FDFD5B343602600814601457FD5B00'_).

```
PC    OPCODE    NAME
-----------------------
00      36      CALLDATASIZE
01      6003    PUSH1 03
03      10      LT
04      6009    PUSH1 09
06      57      JUMPI
07      FD      REVERT
08      FD      REVERT
09      5B      JUMPDEST
0A      34      CALLVALUE
0B      36      CALLDATASIZE
0C      02      MUL
0D      6008    PUSH1 08
0F      14      EQ
10      6014    PUSH1 14
12      57      JUMPI
13      FD      REVERT
14      5B      JUMPDEST
15      00      STOP

```

Bytecode analysis (opcodes unseen from previous challenges):
- `LT` pops two values from the stack and returns `1` if `a < b` , `0` otherwise, where `a` is the element at the top of the stack

First, we need to reach the `JUMPDEST` at PC `09`. `PUSH1 09` pushes this location on top of the stack, and `JUMPI` is used to jump to that instruction. To succeed, `CALLDATASIZE` must be greater than `03`, so `LT` will return `1`.

The first condition we need to satisfy is:
```math
x > 3
```

where $x$ is the `CALLDATASIZE`.

Then, we need to reach the`JUMPDEST` at PC `14`. This value is pushed by `PUSH1 14`. For `JUMPI` instruction to jump to that instruction, `EQ` must return `1`. `EQ` compares `08` with the product between `CALLVALUE` and `CALLDATASIZE`.

The second condition we need to satisfy is:
```math
x * y = 8
```

where $x$ is the `CALLDATASISE`, and $y$ is the `CALLVALUE`.

To summarize, we need to solve the following:
```math
x * y = 8
```
```math
x > 3
```

The values that satisfy the above are:
```math
x = 4
```
```math
y = 2
```

We need to send `2` wei and a `calldata` of `4` bytes (for example, `01010101`).

### Solution

```json
{"value":2,"data":"0x01010101"}
```

evm.code bytecode solution challenge: [link](https://www.evm.codes/playground?callValue=2&unit=Wei&callData=0x01010101&codeType=Bytecode&code='36600310600957FDFD5B343602600814601457FD5B00'_)


## Puzzle 10

evm.code bytecode challenge: [link](https://www.evm.codes/playground?callValue=&unit=Wei&callData=&codeType=Bytecode&code='38349011600857FD5B3661000390061534600A0157FDFDFDFD5B00'_).

```
PC    OPCODE    NAME
-----------------------
00      38      CODESIZE
01      34      CALLVALUE
02      90      SWAP1
03      11      GT
04      6008    PUSH1 08
06      57      JUMPI
07      FD      REVERT
08      5B      JUMPDEST
09      36      CALLDATASIZE
0A      610003  PUSH2 0003
0D      90      SWAP1
0E      06      MOD
0F      15      ISZERO
10      34      CALLVALUE
11      600A    PUSH1 0A
13      01      ADD
14      57      JUMPI
15      FD      REVERT
16      FD      REVERT
17      FD      REVERT
18      FD      REVERT
19      5B      JUMPDEST
1A      00      STOP
```

Bytecode analysis (opcodes unseen from previous challenges):
- `GT` pops two values from the stack and returns `1` if `a > b`, `0` otherwise, where `a` is the element at the top of the stack
- `SWAP1` swaps the first two values from the stack
- `MOD` computes the integer modulo `a % b`. If the denominator is `0`, the result will be `0`.
- `ISZERO` pops the value from the stack and pushes `1` if the value is `0`, `0` otherwise

First, we need to reach the `JUMPDEST` at PC `08`. `PUSH1 08` pushes this location on top of the stack, and `JUMPI` is used to jump to that instruction. To succeed, `GT` must return` 1`, which is possible if `CODESIZE > CALLVALUE`. We know that the `CODESIZE` is `1b` (there are `27` instructions).

The first condition we need to satisfy is:  
```math
27 > x
```
where $x$ is the `CALLVALUE`.

Then, we need to reach the `JUMPDEST` at PC `19`.   The result of `ISZERO` must be different from `0` (this value is used by `JUMPI`), and this is possible if `CALLDATASIZE % 3 = 0`. Then, `0A` is pushed on top of the stack ad added to the `CALLVALUE`. We want this sum to be `19`, so `CALLVALUE + A0 = 19`.

Here are the conditions we need to satisfy:
```math
y \bmod 3 = 0
```
```math
x + 10 = 25
```

where $y$ is the `CALLDATASIZE`.

To summarize, we need to solve the following:  
```math
y \bmod 3 = 0
```
```math
x + 10 = 25
```
```math
27 > x
```


The values that satisfy the above are: 
```math
x = 15
```
```math
y = {0, 3, 6, 9, ...}
```

The `CALLDATASIZE` can have multiple values as long the result from `CALLDATASIZE mod 3` is zero (`0 mod 3 = 0`, `3 mod 3 = 0`, `6 mod 3 = 0`, etcc).

To solve this challenge, we can send `15` wei and a `calldata` of `3` bytes (for example, `010101`), or we can omit the `calldata` (this way, `0 mod 3` will still return `0` ).

### Solution

- `15` wei and `calldata` of `3` bytes
```json
{"value":15,"data":"0x010101"}
```
evm.code bytecode solution challenge: [link](https://www.evm.codes/playground?callValue=15&unit=Wei&callData=0x010101&codeType=Bytecode&code='38349011600857FD5B3661000390061534600A0157FDFDFDFD5B00'_)


- `15` wei and no `calldata`
```json
{"value":15,"data":"0x"}
```

evm.code bytecode solution challenge: [link](https://www.evm.codes/playground?callValue=15&unit=Wei&callData=&codeType=Bytecode&code='38349011600857FD5B3661000390061534600A0157FDFDFDFD5B00'_)

## References

- [MUL](https://www.evm.codes/#02)
- [SUB](https://www.evm.codes/#03)
- [MOD](https://www.evm.codes/#06)
- [LT](https://www.evm.codes/#10)
- [GT](https://www.evm.codes/#11)
- [EQ](https://www.evm.codes/#14)
- [ISZERO](https://www.evm.codes/#15)
- [XOR](https://www.evm.codes/#18)
- [CALLVALUE](https://www.evm.codes/#34)
- [CALLDATALOAD](https://www.evm.codes/#35)
- [CALLDATASIZE](https://www.evm.codes/#36)
- [CALLDATACOPY](https://www.evm.codes/#37)
- [CODESIZE](https://www.evm.codes/#38)
- [EXTCODESIZE](https://www.evm.codes/#3b)
- [JUMP](https://www.evm.codes/#56)
- [JUMPI](https://www.evm.codes/#57)
- [GAS](https://www.evm.codes/#5a)
- [PUSH1](https://www.evm.codes/#60)
- [PUSH2](https://www.evm.codes/#61)
- [DUP1](https://www.evm.codes/#80)
- [SWAP1](https://www.evm.codes/#90)
- [SWAP5](https://www.evm.codes/#94)
- [CREATE](https://www.evm.codes/#f0)
- [CALL](https://www.evm.codes/#f1)
- [RETURN](https://www.evm.codes/#f3)
