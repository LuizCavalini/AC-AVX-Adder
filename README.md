# Atividade Prática AVX — Arquitetura de Computadores
**UFRJ — Escola Politécnica — DEL**

## Descrição
Somador/subtrator vetorial de 32 bits em VHDL com Carry-Lookahead (CLA).
Opera sobre vetores de 4, 8, 16 ou 32 bits selecionados via vecSize_i.

## Estrutura
src/cla4.vhd — Bloco CLA de 4 bits
src/cla4_tb.vhd — Testbench do bloco CLA4
src/vec_adder.vhd — Somador vetorial completo (top-level)
src/vec_adder_tb.vhd — Testbench com test vectors
digital/vec_adder.dig — Circuito no simulador Digital

## Como simular
ghdl -a src/cla4.vhd
ghdl -a src/cla4_tb.vhd
ghdl -e cla4_tb
ghdl -r cla4_tb

## Interface
A_i 32 bits — Operando A
B_i 32 bits — Operando B
mode_i 1 bit — 0=soma 1=subtração
vecSize_i 2 bits — 00=4b 01=8b 10=16b 11=32b
S_o 32 bits — Resultado
