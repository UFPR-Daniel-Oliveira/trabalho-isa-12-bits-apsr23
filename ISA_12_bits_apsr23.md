# ISA 12 bits – Definição Completa

---

## 1 · Introdução

A **ISA 12 bits** foi inspirada nos princípios das ISAs RISC‑V (32 bits) e MICO XII. As instruções, registradores e imediatos foram restritos para caber no formato de 12 bits, mas mantêm convenções familiares (nomes de registradores, semântica) para facilitar a transposição de exemplos.
---

## 2 · Limitações Conhecidas

| Categoria               | ISA 12 bits | RISC‑V RV32I | Impacto / Mitigação                                                        |
| ----------------------- | ----------- | ------------ | -------------------------------------------------------------------------- |
| **Bits / instr.**       | 12 bits     | 32 bits      | Menos campos & menor imediato                                              |
| **Registradores**       | 8 × 12 bits | 32 × 32 bits | Maior uso de pilha para salvar estado.                                     |
| **Imediato BEQ**        | ±4 (3 bits) | ±2048 (12 b) | Desvios longos usam encadeamento de `BEQ` + `J`.                           |
| **Alcance JAL**         | 9 bits      | 20 bits      | Rotinas maiores exigem `JR` indireto.                                      |
| **Sem U‑type**          | —           | LUI/AUIPC    | Literais de 12 bits requerem duas `ADDI`.                                  |

---

## 3 · Visão Geral da Arquitetura

- **Tamanho da palavra / registrador:** 12 bits, complemento‑de‑dois.  
- **Endereçamento:** memória de instruções e dados indexadas em palavras (12 bits cada).  
- **Paradigma:** *Load/Store*; apenas registradores participam da ALU.  

---

## 4 · Formatos de Instrução

### 4.1 Diagrama Geral

```
  11        9 8        6 5        3 2        0  bit
 +-----------+-----------+-----------+-----------+
 |  Opcode   |    Rd     |    Rs1    |  Rs2/Imm |
 +-----------+-----------+-----------+-----------+
```

### 4.2 Tabela de Formatos

| Formato | Campos                                | Exemplos de uso |
| ------- | ------------------------------------- | --------------- |
| **R**   | opcode(3) · rd(3) · rs1(3) · rs2(3)   | `ADD`, `SUB`, `AND`, … |
| **I**   | opcode(3) · rd/rs2(3) · rs1(3) · imm3 | `ADDI`, `LW`, `SW`, `BEQ` |
| **J**   | opcode(3) · target[8:0]               | `JAL` (PC ← target) |

---

## 5 · Banco de Registradores

| Nº | Nome | Papel (ABI)           | Salvo por |
| -- | ---- | --------------------- | ----------|
| r0 | `zero` | Constante 0 (somente leitura) | — |
| r1 | `v0`   | Valor de retorno            | Chamador |
| r2 | `a0`   | 1.º argumento               | Chamador |
| r3 | `a1`   | 2.º argumento               | Chamador |
| r4 | `sp`   | Ponteiro de pilha           | Callee   |
| r5 | `ra`   | Endereço de retorno (`JAL`) | Callee   |
| r6 | `t0`   | Temporário                  | Chamador |
| r7 | `s0`   | Saved / variável local      | Callee   |

### 5.1 Comparativo com RV32I

| Característica       | ISA 12 bits | RV32I |
| -------------------- | ----------- | ----- |
| Instrução            | 12 bits     | 32 bits |
| Registradores        | 8           | 32 |
| `zero`               | r0          | x0 |
| `sp`                 | r4          | x2 |
| `ra`                 | r5          | x1 |
| Argumentos (`a*`)    | r2–r3       | x10–x17 |
| Valor de retorno     | r1          | x10/x11 |

---

## 6 · Green Card – Conjunto de Instruções

### 6.1 R‑Type (opcode 000–110)

| Opcode | Mnemônico | Semântica                       | Obs.      |
| ------ | --------- | --------------------------------| --------- |
| 000    | `ADD`     | `rd ← rs1 + rs2`                | — |
| 001    | `SUB`     | `rd ← rs1 − rs2`                | — |
| 010    | `AND`     | `rd ← rs1 ∧ rs2`                | — |
| 011    | `OR`      | `rd ← rs1 ∨ rs2`                | — |
| 100    | `SLT`     | `rd ← (rs1 < rs2) ? 1 : 0` (signed) | — |
| 101    | `XOR`     | `rd ← rs1 ⊕ rs2`                | — |
| 110    | `NOT`     | `rd ← ¬rs1` (rs2 ignorado)      | unário |

### 6.2 I‑Type

| Opcode | Mnemônico | Semântica                                |
| ------ | --------- | ---------------------------------------- |
| 111    | `ADDI`    | `rd ← rs1 + sext(imm3)`                  |
| 010    | `LW`      | `rd ← M[rs1 + sext(imm3)]`               |
| 011    | `SW`      | `M[rs1 + sext(imm3)] ← rs2`              |
| 100    | `BEQ`     | `if (rs1 == rs2) PC ← PC + sext(imm3)`   |

### 6.3 J‑Type

| Opcode | Mnemônico | Semântica                 |
| ------ | --------- | ------------------------- |
| 110    | `JAL`     | `ra ← PC+1 ; PC ← target` |
| 101    | `JR`      | `PC ← rs1`                |

### 6.4 Pseudoinstruções

| Mnemônico | Expansão | Função |
| --------- | -------- | ------ |
| `NOP` | `ADDI r0, r0, 0` | Não‑operação |
| `HALT` | (opcode reservado `111_000_000_000`) | Parar simulador |
| `SHOW rA` | (instr. de I/O para banco de testes) | Depuração |
| `BNE` rs1,rs2,off | `BEQ rs1,rs2,+1` ; `J off` | Branch‑Not‑Equal |
| `BLT` rs1,rs2,off | `SLT t0,rs1,rs2` ; `BEQ t0,r0,+1` ; `J off` | signed `<` |
| `BGE` rs1,rs2,off | `SLT t0,rs1,rs2` ; `BEQ t0,r0,off` | signed `≥` |
| `SLTU` rd,rs1,rs2 | (opcode extra) | unsigned `<` |
| `BLTU` / `BGEU`   | usam `SLTU` como `SLT` | unsigned branches |

---

## 7 · Imediatos: Sign‑Extend (3 bits)

```
Imm[2:0]  sext(Imm)
 000        0
 001        +1
 010        +2
 011        +3
 100        −4
 101        −3
 110        −2
 111        −1
```

---

## 8 · Branches Ajustes

### 8.1 Resumo do RV32I B‑Type

| RV32I | Condição |
| ----- | -------- |
| `BEQ` | rs1 == rs2 |
| `BNE` | rs1 ≠ rs2 |
| `BLT` | rs1 < rs2 (signed) |
| `BGE` | rs1 ≥ rs2 (signed) |
| `BLTU`| rs1 < rs2 (unsigned) |
| `BGEU`| rs1 ≥ rs2 (unsigned) |

### 8.2 Implementação na ISA 12 bits

Imediatos de ±4 significam que cada *branch* cobre até sete instruções adiante. Alvos distantes requerem um `J` adicional.

#### Exemplo – `BNE r2,r3,target`

```asm
    BEQ r2, r3, +1   ; se igual, pula o J
    J   target       ; se diferente, salta
```

#### Exemplo – `BLT r1,r5,target`

```asm
    SLT r6, r1, r5   ; r6=1 se r1<r5
    BEQ r6, r0, +1   ; se r1>=r5, pula
    J   target
```

---

## 9 · Estruturas de Programação

- **Seleção (`if/else`)**: comparação + `BEQ` / pseudobranches.  
- **Laços (`while/for`)**: teste invertido + salto de volta.  
- **Funções**: `JAL` chama, `JR ra` retorna; argumentos em `a0–a1`; convenção *caller/callee save* conforme Seção 5.

---

## 10 · Impacto de Design (HW × SW)

- **ALU simples**: apenas operações básicas; `SLTU` acrescenta comparador unsigned.  
- **Controle**: 3 bits de opcode → decodificação direta.  
- **Software**: ~1,4 × mais instruções que RV32I, porém binário 3 × menor.

---

## 11 · Exemplo de `BNE` longo (> ±4)

```asm
        BEQ r2, r3, +1
        J   +2
        ADDI r6, r0, hi(target)
        ADDI r6, r6, lo(target)
        JR   r6
```

