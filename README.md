# RISCV_core-
This repository contains everything about my self made risc v core
# important components
                 ┌──────────────┐
                 │      PC      │
                 └──────┬───────┘
                        ↓
               ┌─────────────────┐
               │ Instruction Mem │
               └────────┬────────┘
                        ↓
                  Instruction
                        ↓
              ┌──────────────────┐
              │  Control Unit    │
              └──────────────────┘
                        │
                        ↓
              ┌──────────────────┐
              │  Register File   │
              └────────┬─────────┘
                       ↓
                  ┌─────────┐
                  │   ALU   │
                  └────┬────┘
                       ↓
               ┌──────────────┐
               │ Data Memory  │
               └──────┬───────┘
                      ↓
                  Write Back
