# Módulo 1 — O que é Assembly

> **Arquitetura:** x86-64
> **Sistema operacional de referência:** Linux
> **ABI:** System V AMD64
> **Sintaxe:** Intel

## 1. Introdução

Este é o primeiro módulo de um curso focado em **ler e interpretar código Assembly x86-64**. O objetivo não é decorar instruções, mas construir um modelo mental de como um processador executa um programa.

Antes de falar em `mov`, `push` ou `jmp`, é preciso entender **o que é** Assembly e **onde** ele se encaixa entre o código-fonte que um programador escreve e o que a CPU realmente executa.

## 2. Do código-fonte à CPU

Todo programa que roda em um computador passa, em algum momento, pelo seguinte caminho:

```
Código C
   ↓
Compilador
   ↓
Assembly
   ↓
Assembler
   ↓
Código de máquina
   ↓
CPU
```

Cada seta representa uma transformação. Vamos detalhar cada etapa.

### 2.1 Código C

É o texto que um programador escreve, em um nível de abstração alto. Exemplo:

```c
int soma(int a, int b) {
    return a + b;
}
```

O programador não precisa saber em qual registrador `a` ou `b` vão parar. O compilador decide isso.

### 2.2 Compilador

O compilador (por exemplo, `gcc` ou `clang`) lê o código C e o traduz para Assembly. Nesse processo, ele:

- decide como os dados serão organizados (em registradores ou na memória);
- decide quais instruções de máquina representam cada operação do código-fonte;
- pode **otimizar** o código, reordenando, eliminando ou combinando operações.

Isso é importante: **o Assembly gerado não é uma tradução mecânica, linha por linha, do C.** Um mesmo trecho de C pode gerar Assembly bem diferente dependendo do compilador, da versão dele e do nível de otimização usado.

### 2.3 Assembly

Assembly é uma representação **textual e legível por humanos** das instruções que a CPU executa. Cada linha de Assembly corresponde (quase sempre) a uma única instrução de máquina.

Exemplo de Assembly (sintaxe Intel) para a função `soma` acima:

```asm
soma:
    mov     eax, edi
    add     eax, esi
    ret
```

Isso ainda não é o que a CPU executa diretamente — é uma representação textual que precisa ser convertida em números.

### 2.4 Assembler

O **assembler** (por exemplo, `as`, ou o assembler embutido no `gcc`) pega o código Assembly e o converte em **código de máquina**: sequências de bytes que a CPU consegue decodificar e executar diretamente.

Cada instrução Assembly vira um ou mais bytes chamados **opcode** (mais detalhes abaixo).

### 2.5 Código de máquina

É a forma final: uma sequência de bytes em binário/hexadecimal. Por exemplo, a instrução:

```asm
add eax, esi
```

pode ser representada, em código de máquina, por bytes como:

```
01 F0
```

Você **não precisa decorar** essas correspondências. O importante, neste momento, é entender que Assembly é uma camada de tradução entre texto legível e bytes que a CPU executa.

### 2.6 CPU

A CPU lê o código de máquina, byte a byte, e o interpreta como instruções. Para cada instrução, ela:

1. busca a instrução na memória (**fetch**);
2. decodifica o que ela significa (**decode**);
3. executa a operação (**execute**).

Esse ciclo (fetch → decode → execute) se repete continuamente enquanto o programa roda. Vamos aprofundar esse modelo no Módulo 2.

## 3. Definições fundamentais

| Termo | Definição |
|---|---|
| **Linguagem de máquina** | Conjunto de instruções em formato binário que a CPU executa diretamente. |
| **Assembly** | Representação textual e legível de cada instrução de máquina, específica para uma arquitetura de CPU. |
| **Assembler** | Programa que converte código Assembly em código de máquina. |
| **Opcode** (*operation code*) | A parte da instrução de máquina que identifica **qual operação** deve ser executada (ex.: somar, mover, comparar). |
| **CPU** | O hardware que busca, decodifica e executa instruções. |
| **Registrador** | Um pequeno espaço de armazenamento dentro da própria CPU, extremamente rápido, usado para guardar valores durante a execução. |
| **Memória** | Espaço de armazenamento externo à CPU (RAM), muito maior, porém mais lento que os registradores. |

## 4. Por que Assembly é específico da arquitetura?

Cada família de CPU (x86-64, ARM, MIPS, RISC-V...) tem seu próprio conjunto de instruções, chamado **ISA** (*Instruction Set Architecture*). Isso significa que o Assembly de x86-64 é diferente do Assembly de ARM, mesmo que ambos possam ter vindo do mesmo código C.

Neste curso, vamos trabalhar **exclusivamente com x86-64**, rodando em **Linux**, seguindo a **System V AMD64 ABI**, com instruções escritas em **sintaxe Intel**. Sempre que uma explicação depender especificamente de algum desses fatores, isso será destacado.

> **Nota sobre sintaxe:** existem duas notações comuns para x86: **Intel** (`mov eax, ebx`) e **AT&T** (`mov %ebx, %eax`). Este curso usa Intel. Se você encontrar código em outro lugar usando `%` e `$`, é AT&T — a lógica é a mesma, a notação é diferente.

## 5. Visão geral do que vem a seguir

Este módulo respondeu "o que é" Assembly. Os próximos módulos vão construir, peça por peça, a capacidade de ler código Assembly real:

- **Módulo 2:** modelo mental da CPU (o que acontece ao executar uma instrução).
- **Módulo 3:** registradores (RAX, RBX, RSP, RIP, e suas subdivisões).
- **Módulo 4:** como dados são representados (bits, bytes, words, hexadecimal).
- **Módulo 5:** memória, endereços e ponteiros.
- **Módulo 6:** instruções fundamentais.
- **Módulo 7:** flags.
- **Módulo 8:** controle de fluxo (if, while, for em Assembly).
- **Módulo 9:** a stack.
- **Módulo 10:** funções e calling convention.
- **Módulo 11:** exemplos integrados C → Assembly.

## 6. Exercícios

### Nível 1 — Conceitual

1. Coloque em ordem, do mais abstrato para o mais concreto: `Código de máquina`, `Código C`, `Assembly`.
2. Qual é a diferença entre **assembler** e **Assembly**? (Cuidado: são coisas diferentes, apesar do nome parecido.)
3. Por que o mesmo código C pode gerar Assembly diferente em compiladores diferentes?
4. Verdadeiro ou falso: "Assembly é a mesma coisa em qualquer CPU, só muda a sintaxe." Justifique.
5. O que é um **opcode**?

---

## 7. Respostas

1. `Código C` → `Assembly` → `Código de máquina` (do mais abstrato/legível ao mais concreto/binário).
2. **Assembly** é a linguagem textual (as instruções escritas, como `mov eax, ebx`). O **assembler** é o programa que converte esse texto em código de máquina (bytes). Um é a linguagem, o outro é a ferramenta que a processa.
3. Porque o compilador toma decisões: como alocar registradores, quais otimizações aplicar, como organizar o fluxo do programa. Não existe uma tradução única e obrigatória de C para Assembly — existem várias formas válidas de gerar código equivalente.
4. **Falso.** Cada arquitetura (x86-64, ARM, MIPS...) tem seu próprio conjunto de instruções (ISA). Assembly de x86-64 e Assembly de ARM não são a mesma linguagem com sintaxes diferentes — são conjuntos de instruções distintos, com registradores e capacidades diferentes.
5. É a parte da instrução de máquina (em binário) que diz à CPU **qual operação** executar (por exemplo, "somar", "mover dado", "comparar").

---

*Próximo módulo: [Módulo 2 — Modelo mental da CPU](./02-modelo-mental-da-cpu.md)*
