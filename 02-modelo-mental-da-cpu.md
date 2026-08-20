# Módulo 2 — Modelo Mental da CPU

> **Arquitetura:** x86-64
> **Sistema operacional de referência:** Linux
> **ABI:** System V AMD64
> **Sintaxe:** Intel

## 1. Introdução

No Módulo 1, vimos que a CPU executa instruções através de um ciclo: **fetch → decode → execute**. Este módulo tem um único objetivo: construir um modelo mental sólido de **o que realmente acontece** quando uma instrução é executada.

Esse modelo mental é a base para todo o resto do curso. Sem ele, ler Assembly vira decoreba. Com ele, ler Assembly vira raciocínio.

## 2. As quatro peças fundamentais

Toda execução de um programa em Assembly pode ser entendida através de quatro componentes que interagem constantemente:

```
CPU
├── Registradores
├── Unidade de execução
├── Memória
└── Instruções
```

### 2.1 Registradores

Pequenos espaços de armazenamento **dentro da CPU**. São extremamente rápidos, mas em número limitado (algumas dezenas). Servem como "mesa de trabalho" da CPU: é onde os valores ficam enquanto estão sendo manipulados.

Estudaremos os registradores em detalhe no Módulo 3. Por enquanto, pense neles como **variáveis muito rápidas e muito próximas do processador**.

### 2.2 Unidade de execução

É a parte da CPU responsável por realmente **fazer** a operação que uma instrução pede: somar dois números, comparar valores, mover dados. Ela lê os operandos (que podem estar em registradores ou memória), executa a operação definida pela instrução, e escreve o resultado em algum lugar (também registrador ou memória).

Você não precisa entender a unidade de execução em nível de hardware. O importante é saber que **é ali que o "trabalho" acontece** — o resto (registradores, memória) é sobre **onde os dados estão guardados antes e depois**.

### 2.3 Memória

Um espaço de armazenamento **externo à CPU** (a RAM). É muito maior que os registradores (gigabytes contra poucos bytes), mas também mais lenta de acessar. Todo programa, seu código e seus dados residem, em algum momento, na memória.

Estudaremos isso em profundidade no Módulo 5.

### 2.4 Instruções

São os "comandos" que dizem à CPU o que fazer: mover um valor, somar, comparar, desviar o fluxo. Cada instrução Assembly é traduzida (pelo assembler) em uma sequência de bytes que a CPU consegue decodificar.

## 3. O ciclo fetch–decode–execute, em detalhe

Vamos detalhar o ciclo mencionado no Módulo 1, agora com os componentes que acabamos de descrever.

```
┌─────────────────────────────────────────────┐
│                                               │
│   1. FETCH                                   │
│      A CPU busca a próxima instrução na      │
│      memória, no endereço apontado pelo      │
│      registrador RIP (Instruction Pointer)   │
│                                               │
│   2. DECODE                                  │
│      A CPU interpreta os bytes lidos e       │
│      identifica: qual operação é, quais são  │
│      os operandos (registradores? memória?   │
│      valores constantes?)                    │
│                                               │
│   3. EXECUTE                                 │
│      A unidade de execução realiza a         │
│      operação: lê os operandos, calcula ou   │
│      move o que for necessário, escreve o    │
│      resultado                               │
│                                               │
│   4. RIP avança                              │
│      RIP passa a apontar para a próxima      │
│      instrução (ou para outro endereço, se   │
│      a instrução foi um desvio/jump)         │
│                                               │
└─────────────────────────────────────────────┘
         │
         └──────────► volta ao passo 1
```

Esse ciclo se repete, instrução após instrução, do início ao fim da execução do programa (ou até ele ser interrompido).

### 3.1 O papel do RIP

`RIP` (*Instruction Pointer*, em x86-64) é um registrador especial que **sempre aponta para o endereço de memória da próxima instrução a ser executada**. Ele não é usado diretamente para cálculos como outros registradores — sua função é guiar o ciclo fetch.

Isso é essencial para entender **controle de fluxo** mais adiante: instruções como `jmp`, `je`, `call` funcionam **modificando o valor de RIP**, fazendo a CPU "pular" para outro ponto do código em vez de simplesmente seguir para a instrução seguinte.

> Fixe esta ideia: **"pular" em Assembly significa "mudar o valor de RIP"**. Vamos voltar a isso repetidamente nos módulos de flags e controle de fluxo.

## 4. Um exemplo passo a passo

Considere a instrução:

```asm
add eax, ebx
```

Vamos aplicar o ciclo:

1. **Fetch:** a CPU lê, no endereço apontado por RIP, os bytes que representam essa instrução.
2. **Decode:** a CPU identifica que a operação é uma **soma**, que o destino é o registrador `EAX` e que a origem é o registrador `EBX`.
3. **Execute:** a unidade de execução lê o valor atual de `EAX`, lê o valor atual de `EBX`, soma os dois, e escreve o resultado de volta em `EAX` (sobrescrevendo o valor anterior).
4. **RIP avança** para apontar para a instrução seguinte na memória.

Note que **nada aconteceu com a memória RAM** neste exemplo — tudo se passou dentro da CPU, usando apenas registradores. Isso é comum e é um dos motivos pelos quais operações com registradores são muito mais rápidas do que operações que envolvem memória.

## 5. Registradores vs. Memória: uma analogia

Uma forma útil (mas não perfeita) de pensar nisso:

- **Registradores** são como os itens que você está segurando nas mãos enquanto cozinha: acesso instantâneo, mas você só tem duas mãos (poucos registradores).
- **Memória** é como a despensa: você guarda muito mais coisas lá, mas precisa "ir buscar" cada item antes de usá-lo (acesso mais lento, requer um endereço).

Uma instrução como `mov eax, [rbx]` significa, informalmente: "vá até a despensa, no endereço indicado por RBX, pegue o valor que está lá, e traga para minha mão (EAX)". Esse tipo de instrução será aprofundado no Módulo 5.

## 6. Por que esse modelo importa para leitura de Assembly

Quando você olhar para qualquer trecho de Assembly a partir de agora, você deve conseguir se perguntar, para cada instrução:

- Essa instrução opera sobre **registradores**, sobre **memória**, ou sobre ambos?
- O que a **unidade de execução** está realmente calculando aqui?
- Essa instrução é uma instrução "normal" (RIP simplesmente avança) ou é um **desvio** (RIP muda de forma não sequencial)?

Essas três perguntas, repetidas instrução por instrução, são o núcleo do processo de leitura que formalizaremos mais adiante (Módulo 13 do plano geral: "Como ler Assembly").

## 7. Exercícios

### Nível 1 — Conceitual

1. Descreva, com suas palavras, o que acontece em cada uma das três etapas do ciclo fetch-decode-execute.
2. Qual registrador guia o processo de fetch? O que ele indica exatamente?
3. Por que operações usando apenas registradores tendem a ser mais rápidas que operações envolvendo memória?
4. O que significa, em termos do modelo mental deste módulo, dizer que uma instrução "desviou o fluxo de execução"?

### Nível 2 — Aplicação

5. Para a instrução `sub ecx, edx`, descreva o que acontece em cada etapa do ciclo (fetch, decode, execute, avanço do RIP) — de forma análoga ao exemplo da Seção 4.

---

## 8. Respostas

1. **Fetch:** a CPU busca, na memória, os bytes da próxima instrução, usando o endereço indicado por RIP. **Decode:** a CPU interpreta esses bytes para determinar qual operação é e quais são seus operandos. **Execute:** a unidade de execução realiza de fato a operação (soma, comparação, movimentação de dado etc.) e grava o resultado.
2. O registrador `RIP` (*Instruction Pointer*). Ele indica o endereço de memória da próxima instrução a ser buscada.
3. Porque registradores estão fisicamente dentro da CPU e têm acesso quase instantâneo, enquanto a memória (RAM) é externa à CPU e mais lenta de acessar — é necessário calcular/enviar um endereço e esperar a resposta.
4. Significa que, em vez de o RIP simplesmente avançar para a instrução seguinte na sequência, ele foi alterado para apontar para outro endereço — ou seja, a execução "pulou" para outro ponto do código, em vez de seguir linearmente.
5. **Fetch:** a CPU busca, no endereço apontado por RIP, os bytes correspondentes a `sub ecx, edx`. **Decode:** a CPU identifica que a operação é uma **subtração**, com destino `ECX` e origem `EDX`. **Execute:** a unidade de execução lê o valor atual de `ECX`, lê o valor atual de `EDX`, subtrai (`ECX - EDX`), e grava o resultado de volta em `ECX`. **Avanço do RIP:** RIP passa a apontar para a instrução seguinte na sequência do programa.

---

*Módulo anterior: [Módulo 1 — O que é Assembly](./01-o-que-e-assembly.md)*
*Próximo módulo: [Módulo 3 — Registradores](./03-registradores.md)*
