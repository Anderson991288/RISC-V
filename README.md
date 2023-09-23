# RISC-V_CPU

A simple CPU for **RISC-V** written by Verilog.

用Verilog編寫的簡單的RISC-V處理器。

## Introduction

- Design a processor based on the RISC-V instruction set by Verilog;
- 使用Verilog設計實現基於RISC-V指令集的處理器；
- Implements some instructions in the **RV32I** instruction set;
- 實現RV32I指令集中的部分指令；
- Run the test codes and get correct results.
- 運行測試指令，得到正確的運行結果

## Content

- Design a **single-cycle** processor;
- 設計實現單週期的32位元RISC-V處理器；
- Improve to a **multi-cycle** processor;
- 在單週期RISC-V處理器的基礎上進行改進，實現多週期的RISC-V處理器；
  - **Single issue**, **5 stages**;
  - **Data forwarding**, **pipeline blocking**;
  - Branch Prediction (not necessary).
- Use **Modelsim** to view waveform;
- 使用Modelsim進行模擬驗證，查看對應訊號波形；
- Use Vivado to optimize timing (not necessary);
- 使用Vivado對程式碼進行綜合，盡可能的最佳化時序(選做)；
- Run the test codes.
- 運行測試指令，得到正確的輸出結果。

<img src="https://github.com/VenciFreeman/RISC-V/blob/master/img/structure.jpg" style="zoom:50%;" />

## Set

| Instruction | Opcode  | Funct3 | Funct6/7 |
| :---------: | :-----: | :----: | :------: |
|    `add`    | 0110011 |  000   | 0000000  |
|   `addi`    | 0010011 |  000   |   N/A    |
|    `sub`    | 0110011 |  000   | 0100000  |
|    `and`    | 0110011 |  111   | 0000000  |
|    `or`     | 0110011 |  110   | 0000000  |
|    `xor`    | 0110011 |  100   | 0000000  |
|    `blt`    | 1100111 |  100   |   N/A    |
|    `beq`    | 1100111 |  000   |   N/A    |
|    `jal`    | 1101111 |  N/A   |   N/A    |
|    `sll`    | 0110011 |  001   | 0000000  |
|    `srl`    | 0110011 |  101   | 0000000  |
|    `lw`     | 0000011 |  010   |   N/A    |
|    `sw`     | 0100011 |  010   |   N/A    |

## Explain

> Only if.v and register.v is sequential circuit.

![](https://github.com/VenciFreeman/RISC-V/blob/master/img/SingleCycle.jpg)

### if.v (single cycle)

- PC+4 per cycle;
- 程序計數器PC 在正常情況下，每個週期加4；
- PC value send to inst_mem as the address of instruction, then send the instruction to decode module;
- PC值送入inst_mem作為指令的位址，將所獲得的指令直接送入譯碼模組；
- If current instruction is beq, blt or jal, update PC immediately.
- 如果目前的指令是條件分支指令(beq,blt)或直接跳轉指令(jal)，當分支成立時，PC需要更新為跳躍指令的目標位址。
  
<img src="https://github.com/VenciFreeman/RISC-V/blob/master/img/inst_mem.jpg" alt=" " style="zoom:50%;" />

| Signal | Bit width |                    Function                    |
| :----: | :-------: | :--------------------------------------------: |
|   ce   |     1     | Chip select signal, high level enable inst_mem |
|  addr  |    32     |              instruction address               |
|  inst  |    32     |                  instruction                   |

### id.v (single cycle)

- Decode the instruction, get ALUop by opcode, funct3 and funct7;
- 對取到的指令進行譯碼，根據opcode, funct3, funct7確定具體的ALUop；
- Determine if the register needs to be read or not and send the register numbers rs1, rs2 to the register file, and read the corresponding data as the source operand;
- 判斷是否需要讀取暫存器，把要讀取的暫存器號碼rs1,rs2 送入register file，讀取對應的資料當作來源運算元；
- Determine if the register needs to be written or not then output target register number rd and write register flag for register write back;
- 判斷是否需要寫入暫存器，輸出目標暫存器號rd、寫入暫存器標誌，用於暫存器回寫；
- Determine if the branch instruction is true or not and calculate the jump address (or in ex.v);
- 判斷分支指令是否成立，併計算跳轉位址(也可以在ex.v中實現)；
- Sign extension (or unsigned extension) for instructions containing immediate values as one of the source operands.
- 將含有立即數的指令進行符號擴展(或無符號擴展，具體參考spec檔)，作為其中的一個來源操作數。

| Instruction | opcode  | funct3 | funct7  | ALUop  |
| :---------: | :-----: | :----: | :-----: | :----: |
|     add     | 0110011 |  000   | 0000000 | 000001 |
|     sub     | 0110011 |  000   | 0100000 | 000010 |
|     sll     | 0110011 |  001   | 0000000 | 000011 |
|     jal     | 1101111 |  N/A   |   N/A   | 000100 |

### ex.v (single cycle)

- Use the ALUop and the two source operands decoded in id.v to perform the corresponding operation;
- 根據id.v中譯碼所得的ALUop和兩個源操作數，進行相應的操作；
- If ALUop indicates that it's an addition operation, the two operands will be added; the sub operation can be implemented by complement.
- 如果ALUop顯示是加法操作，則將兩個操作數相加；減法操作可以透過加補碼來實現。

<img src="https://github.com/VenciFreeman/RISC-V/blob/master/img/ALUop.jpg" style="zoom:50%;" />

| ALUop  | Operation |
| :----: | :-------: |
| 000001 |     +     |
| 000011 |    <<     |
| 000010 |     -     |

### mem.v (single cycle)

- Read 32bit data from data_mem if instruction is lw;
- 指令是lw指令，則從data_mem讀取32bit的資料；
- Write 32bit data into data_mem if instruction is sw;
- 指令是sw指令，則向data_ mem中寫入32bit的資料；
- Do no operation if there is other instruction.
- 如果是其他指令，則不做操作。

| Signal | Bit width |                           Function                           |
| :----: | :-------: | :----------------------------------------------------------: |
|   ce   |     1     |        Chip select signal, high level enable inst_mem        |
|   we   |     1     | Write into data_mem at high level and read from data_mem at low level |
|  addr  |    32     |                        Address signal                        |
| data_i |    32     |          the data which need to write into data_mem          |
| data_o |    32     |               the data which is from data_mem                |

### register.v

- When we need to read rs1, rs2, read data from register file according to the address in rs1, rs2;
- 當需要讀取rs1,rs2時，根據rs1,rs2的地址從register file中讀取資料；
- When we need to write rd, write data into register file according to the address in rd;
- 當需要寫入rd時，依照rd的位址往register file中寫入數據
- The operation of writing data to the register file uses sequential logic, and the operation of reading data from the register file using combinational logic;
- 往register file中寫入資料的操作採用時序邏輯，即在下一個時脈上升沿寫入數據，從register file中讀取資料採用組合邏輯；
- X0 is a constant zero register in RISC-V. When the target register rd is X0, the data won't actually be written to X0;
- RISC-V中X0為恆零暫存器，當目標暫存器rd為X0時，資料實際上不會被寫入X0；
- If the read and write register signals are valid at the same time, and if the read address is the same as the write address, then the data which need to write can be directly output as read data to achieve data forwarding.
- 如果讀暫存器訊號與寫入寄存器訊號同時有效，且讀取位址與寫入位址相同，此時則可以將要寫入的資料直接輸出為讀取資料,實現資料轉送。
  
![](https://github.com/VenciFreeman/RISC-V/blob/master/img/FiveStages.jpg)

### stall.v (5 stages)

- Pause the pipeline When data adventures cannot be resolved through data forwarding;
- 當資料冒險無法透過資料轉發解決，使流水暫停；
- Collect stall request signals from levels. If some level sends a stall request, all sequential circuits in front of the level should be paused.
- 從各級之間收集stall請求訊號，如果某級發出stall請求，則將該級前面所有的時序電路暫停。
  
### if_id.v (5 stages)

- Sequential logic: pass PC value and inst. When the pipeline is blocked, keep pc and inst; when the branch is established, clear pc and inst.
- 時序邏輯: 傳遞PC值和inst。流水線阻塞時，pc和inst不變；分公司成立時，pc和inst清零。

### id_ex.v (5 stages)

- Sequential logic: pass the decoded ALUop, source operand, destination register address, write register flag and other signals in id.v.
- 時序邏輯: 傳遞id.v中譯碼得到的ALUop，來源操作數，目標暫存器位址，寫入暫存器標誌等訊號。
- When the pipeline is blocked, the above signals remain unchanged or Bubble.
- 流水線阻塞時，以上訊號保持不變或清零(Bubble)

### ex_mem.v (5 stages)

- Sequential logic: Pass the calculated result data in ex.v, target register address, write register flag and other signals.
- 時序邏輯: 傳遞ex.v中計算得到的結果數據，目標暫存器位址，寫入暫存器標誌等訊號。
- When the pipeline is blocked, the above signals remain unchanged or Bubble.
- 流水線阻塞時，以上訊號保持不變或清零(Bubble)

### mem_wb.v (5 stages)

- Sequential logic: Pass the result data to be written to the register, the destination register address, the write register flag and other signals.
- 時序邏輯: 傳遞要寫入暫存器的結果數據，目標暫存器位址，寫入暫存器標誌等訊號。


![](https://github.com/VenciFreeman/RISC-V/blob/master/img/AddModule.jpg)

