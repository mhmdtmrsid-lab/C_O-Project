# 🖥️ Basic Computer — Hardwired CPU (Logisim Evolution)

> Implementation of a 16-bit Basic Computer with a Hardwired Control Unit, Data Unit, and I/O Interface using **Logisim Evolution**.
>
> **Course:** Computer Architecture and Organization 
<img src="Screenshot%202026-04-26%20032536.png" width="1000"/>

---

## 📋 Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Data Unit](#data-unit)
- [Control Unit](#control-unit)
- [I/O Interface](#io-interface)
- [Instruction Set](#instruction-set)
- [How to Run](#how-to-run)
- [Sample Programs](#sample-programs)
- [Project Structure](#project-structure)
- [Troubleshooting](#troubleshooting)

---

## 🎯 Project Overview

This project implements a **Basic Computer** architecture with:

- ✅ **Data Unit** — 7 registers, 16-bit Data Bus, ALU, and 1024×16-bit RAM
- ✅ **Control Unit** — Hardwired logic with Sequence Counter, Decoders, and Control Logic Gates
- ✅ **I/O Interface** — 8 input switches (INPR) and 8 output LEDs (OUTR) with flag-based handshaking
- ✅ **15 Instructions** — Full instruction set implementation

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────┐
│                    MAIN CIRCUIT                      │
│                                                      │
│   ┌──────────────┐         ┌──────────────────┐     │
│   │ CONTROL UNIT │◄───────►│    DATA UNIT     │     │
│   │              │         │                  │     │
│   │  SC Counter  │         │  AR  PC  IR  DR  │     │
│   │  Timing Dec. │         │  AC  INPR  OUTR  │     │
│   │  Instr. Dec. │         │  ALU + RAM       │     │
│   │  Logic Gates │         │  16-bit Data Bus │     │
│   └──────────────┘         └──────────────────┘     │
│                                                      │
│   [START] [RESET] [FGI]    [8 Switches] [8 LEDs]   │
└─────────────────────────────────────────────────────┘
```

---

## 📦 Data Unit

<img src="Screenshot%202026-04-26%20033715.png" width="1000"/>

### Registers

| Register | Width | Operations | Function |
|---|---|---|---|
| AR | 11-bit | LD, INC, CLR, RESET | Memory address register |
| PC | 11-bit | LD, INC, CLR, RESET | Program counter |
| IR | 16-bit | LD, RESET | Instruction register |
| DR | 16-bit | LD, INC, CLR, RESET | Data register |
| AC | 16-bit | LD, CLR, RESET | Accumulator |
| INPR | 8-bit | LD, RESET | Input register (from switches) |
| OUTR | 8-bit | LD, RESET | Output register (to LEDs) |

### Register Internal Structure

Each register sub-circuit contains:

```
D_in (16-bit) ──► Splitter ──► MUX ──► Register ──► Extender ──► Buffer ──► Q_out (16-bit)
                               ▲                                      ▲
                         Priority Logic                           _out Enable
                         (LD/INC/CLR)
```

**Priority Logic:**
```
S1 = CLR
S0 = INC AND (NOT CLR)
WE = LD OR INC OR CLR
```
<img src="Screenshot 2026-04-26 033744.png" width="1000"/>


### IR Register Special Output

```
IR[15:12] ──► IR_OPCODE (4-bit) ──► Instruction Decoder
IR[11]    ──► IR_I (1-bit)      ──► Control Logic
IR[10:0]  ──► IR_ADDR (11-bit)  ──► Data Bus (via Buffer, enabled at T2)
```

### ALU Operations

| Signal | Operation | Result |
|---|---|---|
| ADD | Addition | AC + DR |
| AND | Bitwise AND | AC AND DR |
| CMA | Complement | NOT AC |
| CLA | Clear | 0x0000 |

**Flags:**
- **E Flag** — Carry out from Adder (D Flip-Flop), cleared by CLE instruction
- **Z Flag** — Zero result detector (16-input OR + NOT), cleared on SC_CLR
<img src="Screenshot 2026-04-26 033824.png" width="1000"/>

### Memory

```
Size    : 1024 × 16-bit RAM
Address : AR Q_out → Splitter(16→10) → RAM Address (A pin)
Data In : Data Bus → RAM D pin
Data Out: RAM Q pin → Data Bus (controlled by OE = MEM_READ)
Write   : MEM_WRITE → RAM WE pin
```
<img src="Screenshot 2026-04-26 033847.png" width="1000"/>

---

## ⚙️ Control Unit

<img src="Screenshot 2026-04-26 033727.png" width="1000"/>

### Components

```
┌─────────────────────────────────────────┐
│             CONTROL UNIT                │
│                                         │
│  CLK ──► T_FlipFlop ──► AND ──► SC_EN  │
│  S_FLAG Q ────────────────►             │
│                                         │
│  SC (4-bit) ──► Timing Decoder          │
│                 T0 ... T15              │
│                                         │
│  IR_OPCODE ──► Instr. Decoder           │
│                D0 ... D15              │
│                                         │
│  T × D ──► Control Logic Gates          │
│            ──► All Control Signals      │
└─────────────────────────────────────────┘
```

### Timing

> ⚠️ The SC counts every **2 CLK pulses** (via T Flip-Flop) to ensure stable Bus data before registers sample.

| T State | Operation | Active Signals |
|---|---|---|
| T0 | Fetch — AR ← PC | PC_out, AR_LD |
| T1 | Fetch — IR ← M[AR], PC++ | MEM_READ, IR_LD, PC_INC |
| T2 | Decode — AR ← IR_ADDR | IR_ADDR_out, AR_LD |
| T3 | Execute Step 1 | Instruction-dependent |
| T4 | Execute Step 2 | Instruction-dependent |
| T5 | Execute Step 3 | ISZ / BSA only |

### Control Logic Equations

```
AR_LD        = T0 + T2
PC_out       = T0 + (D9·T3)
PC_INC       = T1 + (D7·T5·Z) + (D13·T3·FGI) + (D14·T3·FGO)
PC_LD        = (D8·T3) + (D9·T5)
IR_LD        = T1
IR_ADDR_out  = T2
DR_LD        = (D0+D3+D4+D7)·T3
DR_INC       = D7·T4
DR_out       = (D0·T4) + (D7·T5)
AC_LD        = (D0·T4)+(D3·T4)+(D4·T4)+(D5·T3)+(D6·T3)+(D11·T3)
AC_out       = (D1·T3) + (D9·T3) + (D12·T3)
ADD          = D3·T4
AND          = D4·T4
CMA          = D5·T3
CLA          = D6·T3
ALU_out      = ADD + AND + CMA + CLA
MEM_READ     = T1 + (D0+D3+D4+D7)·T3
MEM_WRITE    = (D1·T3) + (D7·T5) + (D9·T3)
AR_out       = (D8·T3) + (D9·T5)
AR_INC       = D9·T4
INPR_out     = D11·T3
INPR_LD      = FGI
OUTR_LD      = D12·T3
FGI_CLR      = D11·T3
FGO_CLR      = D12·T3
CLE          = D10·T3
S_CLR        = D15·T3
SC_CLR       = (D0·T4)+(D1·T3)+(D3·T4)+(D4·T4)+(D5·T3)+(D6·T3)
               +(D7·T5)+(D8·T3)+(D9·T5)+(D10·T3)+(D11·T3)+(D12·T3)
               +(D13·T3)+(D14·T3)+(D15·T3)
```
<img src="Screenshot 2026-04-26 033950.png" width="1000"/>

---

## 🔌 I/O Interface

```
8 Switches ──► INPR D_in
               INPR_LD = FGI (auto-load when input ready)
               FGI Button ──► FGI Flag SET
               FGI_CLR (D11·T3) ──► FGI Flag RESET

OUTR Q_out ──► 8 LEDs
               OUTR_LD = D12·T3 (load on OUT instruction)
               FGO_CLR (D12·T3) ──► FGO Flag RESET
```

---

## 📝 Instruction Set

| Instruction | Opcode | HEX | Operation | Description |
|---|---|---|---|---|
| LDA | 0000 | 0 | AC ← M[EA] | Load AC from memory |
| STA | 0001 | 1 | M[EA] ← AC | Store AC to memory |
| ADD | 0011 | 3 | AC ← AC + M[EA] | Add memory to AC |
| AND | 0100 | 4 | AC ← AC ∧ M[EA] | AND memory with AC |
| CMA | 0101 | 5 | AC ← AC' | Complement AC |
| CLA | 0110 | 6 | AC ← 0 | Clear AC |
| ISZ | 0111 | 7 | M[EA]++; if 0: PC++ | Increment and skip if zero |
| BUN | 1000 | 8 | PC ← EA | Unconditional branch |
| BSA | 1001 | 9 | M[EA]←PC; PC←EA+1 | Branch and save return |
| CLE | 1010 | A | E ← 0 | Clear E flag |
| INP | 1011 | B | AC(0-7) ← INPR | Input from switches |
| OUT | 1100 | C | OUTR ← AC(0-7) | Output to LEDs |
| SKI | 1101 | D | if FGI=1: PC++ | Skip on input flag |
| SKO | 1110 | E | if FGO=1: PC++ | Skip on output flag |
| HLT | 1111 | F | S ← 0 | Halt computer |

### Instruction Format

```
 15  14  13  12 | 11 | 10  9  8  7  6  5  4  3  2  1  0
 ───────────────|────|───────────────────────────────────
    OPCODE      │ I  │           ADDRESS (11-bit)
    (4-bit)     │    │
```

---

## 🚀 How to Run

### Prerequisites

- [Logisim Evolution](https://github.com/logisim-evolution/logisim-evolution) installed

### Steps

**1. Open the project:**
```
File → Open → CO_Project.circ
```

**2. Load a program into RAM:**
```
Right-click RAM → Edit Contents → Enter hex values → Close Window
```

**3. Run the simulation:**
```
Press RESET  →  All registers cleared to 0
Press START  →  S_FLAG = 1, computer starts
Simulate → Enable Clock  →  Auto execution
```

**4. For I/O programs:**
```
Set DIP switches to desired value
Press FGI button → FGI = 1 (input ready)
Observe LEDs for output
```

---

## 💾 Sample Programs

### Program 1 — Addition (5 + 3 = 8)

```
Address | Hex  | Assembly | Description
--------|------|----------|---------------------------
000     | 0008 | LDA 8    | Load M[008] = 5 into AC
001     | 3009 | ADD 9    | AC = AC + M[009] = 5+3=8
002     | 100A | STA A    | Store result at M[00A]
003     | F000 | HLT      | Stop
008     | 0005 | DATA     | First operand = 5
009     | 0003 | DATA     | Second operand = 3
00A     | 0000 | RESULT   | Result stored here
```

**Expected:** `AC = 0008`, `M[00A] = 0008`

---

### Program 2 — Input/Output

```
Address | Hex  | Assembly | Description
--------|------|----------|---------------------------
000     | D000 | SKI      | Skip if FGI=1
001     | 8000 | BUN 000  | Loop back if not ready
002     | B000 | INP      | AC ← INPR (switches)
003     | C000 | OUT      | OUTR ← AC (LEDs)
004     | F000 | HLT      | Stop
```

**How to run:** Set switches → Press FGI → Press START

---

## 📁 Project Structure

```
CO_Project/
│
├── CO_Project.circ          # Main Logisim Evolution file
│
├── Sub-circuits/
│   ├── DataUnit             # Top-level data unit
│   ├── AR_Register          # 11-bit address register
│   ├── PC_Register          # 11-bit program counter
│   ├── IR_Register          # 16-bit instruction register
│   ├── DR_Register          # 16-bit data register
│   ├── AC_Register          # 16-bit accumulator
│   ├── INPR                 # 8-bit input register
│   ├── OUTR                 # 8-bit output register
│   ├── ALU                  # Arithmetic & Logic Unit
│   ├── ControlUnit          # Top-level control unit
│   ├── Sequence_Counter     # 4-bit SC with T Flip-Flop
│   ├── Timing_Decoder       # 4×16 decoder (T0-T15)
│   ├── Instruction_Decoder  # 4×16 decoder (D0-D15)
│   ├── Control_Logic        # All control signal gates
│   ├── S_FLAG               # Start/Stop flip-flop
│   └── FLAGS_I_O            # FGI and FGO flip-flops
│
├── README.md                # This file
└── BasicComputer_Project_Report.docx  # Full project report
```

---

## 🔧 Troubleshooting

| Wire Color | Meaning | Fix |
|---|---|---|
| 🟠 Orange | Floating bus — no Buffer enabled | Ensure exactly one Buffer Enable = 1 |
| 🟡 Yellow | High impedance — unconnected pin | Connect all Enable pins |
| 🔴 Red | Conflict — two sources on same bus | Check only one output signal active per T |
| 🔵 Blue | Unknown value — unconnected input | Check all input pins are connected |

### Common Issues

**SC not counting:**
- Check S_FLAG = 1 (press START)
- Verify T Flip-Flop output connected to SC Enable via AND gate

**IR loading unknown values:**
- Check MEM_READ = 1 in T1
- Verify AR has correct PC value after T0
- Check RAM has program loaded at address 000

**Wrong instruction executing:**
- Check IR Register Splitter: bits 15:12 → IR_OPCODE
- Verify IR_OPCODE connected to Instruction Decoder input

---

## 📚 References

- Morris Mano, *Computer System Architecture*, 3rd Edition
- Logisim Evolution Documentation: [github.com/logisim-evolution](https://github.com/logisim-evolution/logisim-evolution)

---

*Course: Computer Architecture and Organization*
