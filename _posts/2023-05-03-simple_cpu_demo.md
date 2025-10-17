---
layout: post
title: "对CPU运算过程的简单模拟（使用C描述）"
date:   2023-05-03
tags: [Logic Circuit,Hardware,C]
comments: true
author: nullptr-0
---

一个体现CPU基本运算流程的demo。

```C
#include <stdio.h>

// CSA
int fullAdder(int a, int b, int carryIn, int* carryOut) {
    int sum = a ^ b ^ carryIn;  // 异或操作，计算和
    *carryOut = (a & b) | (carryIn & (a ^ b)); // 或和与操作，计算进位
    return sum;
}

int halfAdder(int a, int b, int* carry) {
    return fullAdder(a, b, 0, carry);
}

// 6位RCA: 适配指令集的6位操作数，抛弃最高位进位
// halfAdder和fullAdder的组合: 假设最低位的全加器carry=0，实际就退化为半加器
int sixBitRCA(int a, int b, int carry) {
    int sum = 0;
    for (int i = 0; i < 6; i++) {
        int aBit = (a >> i) & 1;  // 获取a的第i位
        int bBit = (b >> i) & 1;  // 获取b的第i位
        int sumBit = fullAdder(aBit, bBit, carry, &carry);  // 计算和的第i位
        sum |= sumBit << i;
    }
    return sum;
}

// 6位乘法器: 适配指令集的6位操作数，12位分两个寄存器存储
// 逐位相乘，再使用比RCA更高效的CSA计算前5位的部分和和进位，最后使用RCA将剩余的部分和与各个位的进位加起来
int sixBitMultiplier(int a, int b) {
    int ret = 0;
    int carry = 0;
    int ps; // partial sum
    ret |= a & (b & 1) & 1;
    ps = fullAdder(a * (b & 1) >> 1, a * ((b >> 1) & 1), (a * ((b >> 2) & 1) << 1) & 0x3F, &carry);
    ret |= (ps & 1) << 1;
    ps = fullAdder((a * ((b >> 3) & 1) << 1) & 0x3F, (ps >> 1) | (a * ((b >> 2) & 1) & 0x1F), carry, &carry);
    ret |= (ps & 1) << 2;
    ps = fullAdder((a * ((b >> 4) & 1) << 1) & 0x3F, (ps >> 1) | (a * ((b >> 3) & 1) & 0x1F), carry, &carry);
    ret |= (ps & 1) << 3;
    ps = fullAdder((a * ((b >> 5) & 1) << 1) & 0x3F, (ps >> 1) | (a * ((b >> 4) & 1) & 0x1F), carry, &carry);
    ret |= (ps & 1) << 4;
    ps = sixBitRCA(0, (ps >> 1) | (a * ((b >> 5) & 1) & 0x1F), carry, &carry);
    ret |= ps << 5;
    return ret;
}

int negative(int a) {
    return (a ^ 0x003F) + 1;
}

#define MEMORY_SIZE 32

typedef struct {
    int data;
} Register;

typedef struct {
    Register registers[16];
} IndexRegisters;

typedef struct {
    int address;
    int data;
} Memory;

typedef struct {
    int instruction; // 4bit-op & 6bit-op1 & 6bit-op2
} InstructionRegister;
// why this?
// this is a prefix-expression like. and it is easier to process:
// the operator comes first, so we know what to do and what kind of operands we need.
// then we go and get our operands and perform an instruction.
// this is a simple demo so we'll always have op2.
// however in real-world implementations, we might have a flexible-sized structure to save memory.
// the actual size then will be operator dependent.

// 从pc内存处读一条指令，并移至下一指令处
void fetchInstruction(InstructionRegister* ir, Memory* memory, Register* pc) {
    ir->instruction = memory[pc->data].data;
    pc->data++;
}

void decodeInstruction(InstructionRegister* ir, Register* op, Register* op1, Register* op2) {
    op->data = (ir->instruction & 0xF000) >> 12;
    op1->data = (ir->instruction & 0x0FC0) >> 6;
    op2->data = ir->instruction & 0x003F;
}

void executeInstruction(Register* op, Register* op1, Register* op2, IndexRegisters* irs, Register* acc, Memory* memory) {
    switch (op->data) {
    case 0:  // NOP
        break;
    case 1:  // NEG irs[op1] 补码表示取负
        irs->registers[op1->data].data = negative(irs->registers[op1->data].data);
        break;
    case 2:  // ADD irs[op1], op2
        irs->registers[op1->data].data = sixBitRCA(irs->registers[op1->data].data, op2->data, 0);
        break;
    case 3:  // ADD irs[op1], irs[op2]
        irs->registers[op1->data].data = sixBitRCA(irs->registers[op1->data].data, irs->registers[op2->data].data, 0);
        break;
    case 4:  // SUB irs[op1], op2
        irs->registers[op1->data].data = sixBitRCA(irs->registers[op1->data].data, negative(op2->data), 0); // ADD op1, (NEG op2)
        break;
    case 5:  // SUB irs[op1], irs[op2]
        irs->registers[op1->data].data = sixBitRCA(irs->registers[op1->data].data, irs->registers[negative(op2->data)].data, 0); // ADD op1, (NEG op2)
        break;
    case 6:  // MUL irs[op1], op2
        irs->registers[op1->data].data = sixBitMultiplier(irs->registers[op1->data].data, op2->data);
        acc->data = irs->registers[op1->data].data & 0x0FC0;
        irs->registers[op1->data].data = irs->registers[op1->data].data & 0x003F;
        break;
    case 7:  // MUL irs[op1], irs[op2]
        irs->registers[op1->data].data = sixBitMultiplier(irs->registers[op1->data].data, irs->registers[op2->data].data);
        irs->registers[op1->data].data = irs->registers[op1->data].data & 0x0FC0;
        irs->registers[op1->data].data = irs->registers[op1->data].data & 0x003F;
        break;
    case 8:  // MOV irs[op1], op2
        if (op1->data == 0x10)
        {
            acc->data = op2->data;
        }
        else
        {
            irs->registers[op1->data].data = op2->data;
        }
        break;
    case 9:  // MOV irs[op1], irs[op2]
        if (op1->data == 0x10)
        {
            acc->data = irs->registers[op2->data].data;
        }
        else
        {
            if (op2->data == 0x10)
            {
                irs->registers[op1->data].data = acc->data;
            }
            else
            {
                irs->registers[op1->data].data = irs->registers[op2->data].data;
            }
        }
        break;
    case 10:  // STR irs[op1], irs[op2]
        memory[irs->registers[op1->data].data].data = irs->registers[op2->data].data;
        break;
    case 11:  // LDM irs[op1], irs[op2]
        irs->registers[op1->data].data = memory[irs->registers[op2->data].data].data;
        break;
    }
}

int main() {
    Register pc = { 0 };  // 程序计数器
    Register acc = { 0 };  // 累加器 reflect to r16
    Register op = { 0 };  // 操作码
    Register op1 = { 0 };  // 操作数1
    Register op2 = { 0 };  // 操作数2
    InstructionRegister ir = { 0 };  // 指令寄存器
    IndexRegisters irs; // 索引寄存器 0-15
    Memory memory[MEMORY_SIZE];  // 存储器

    // 初始化存储器
    
    // let's do 1 + 1:
    memory[0].address = 0;
    memory[0].data = 0x8001; // MOV r0, 1
    memory[1].address = 1;
    memory[1].data = 0x9040; // MOV r1, r0
    memory[2].address = 2;
    memory[2].data = 0x3001; // ADD r0, r1
    memory[3].address = 3;
    memory[3].data = 0x8090; // MOV r2, 0x10
    memory[4].address = 4;
    memory[4].data = 0xA080; // STR r2, r0
    memory[5].address = 5;
    memory[5].data = 0x0000;

    // 模拟CPU的执行过程
    while (1)
    {
        fetchInstruction(&ir, memory, &pc);
        if (!ir.instruction)
        {
            break;
        }
        decodeInstruction(&ir, &op, &op1, &op2);
        executeInstruction(&op, &op1, &op2, &irs, &acc, memory);
    }

    // 打印结果
    int result = memory[0x10].data;
    printf("结果： %d\n", result);

    return 0;
}
```

**需要特别注意的是，上述实现只是说明原理性质的简化版本，实际的CPU会更为复杂。另外，代码中用到的指令集只是用于配合实现demo的功能，其本身并不完善。**