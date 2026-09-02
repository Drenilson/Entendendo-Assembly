# Módulo 8 — Controle de Fluxo

> **Arquitetura:** x86-64
> **Sistema operacional de referência:** Linux
> **ABI:** System V AMD64
> **Sintaxe:** Intel

## 1. Introdução

Este é o módulo em que tudo que vimos até aqui se junta. Já sabemos:

- que `RIP` guia a execução, e que "desviar" significa alterar `RIP` (Módulo 2);
- que `cmp`/`test` calculam algo e descartam o resultado, deixando pistas em RFLAGS (Módulo 6);
- exatamente o que cada flag significa, e como combiná-las para `==`, `!=`, `>`, `<`, `>=`, `<=` (Módulo 7).

Falta uma peça: as instruções que **leem essas flags e decidem para onde `RIP` vai**. Essas são os **desvios condicionais** (`je`, `jne`, `jg`...) e o desvio incondicional (`jmp`).

O objetivo deste módulo não é aprender a escrever `if`/`while`/`for` em Assembly, e sim reconhecer os padrões que esses construtos geram, para poder ler código desconhecido e entender sua lógica.

## 1.1 Antes de começar: o que significa `mov dword [i], 10`?

Os exemplos deste módulo usam variáveis com nome dentro de colchetes, como `[i]`, `[soma]`, `[x]`. Vale parar um momento antes de prosseguir, porque isso combina duas coisas que já vimos separadamente, mas nunca exatamente juntas dessa forma.

**A parte que já é familiar:** colchetes significam dereferência: "o valor armazenado no endereço de memória entre colchetes" (Módulo 5). E um sufixo como `dword` antes dos colchetes indica o tamanho do dado sendo lido ou escrito (Módulo 4). Então:

```asm
mov dword [i], 10
```

lê-se: "escreva o valor 10 (ocupando 4 bytes, uma dword) no endereço de memória associado a `i`".

**A parte nova:** até aqui, os endereços que vimos eram sempre expressões com registradores, como `[rbx]` ou `[rbp-4]`. Aqui, em vez disso, aparece um nome, como `i`, `soma` ou `x`. Isso é uma **simplificação didática deste módulo**: em vez de escrever o endereço real (que, para uma variável local, normalmente seria algo como `[rbp-4]`, um deslocamento a partir de um registrador-base, assunto do Módulo 9), estamos usando o nome da variável como um "apelido" legível para esse mesmo endereço.

Pense nisso da mesma forma que já pensamos nos rótulos de desvio, como `inicio_loop:` ou `fim:` (Seção 2, a seguir): um rótulo é um nome legível que o assembler traduz para um endereço concreto. `[i]` funciona de forma parecida, mas para uma posição de **dado**, não de código: é um "endereço com nome", em vez do endereço numérico ou da expressão com registrador que ele realmente representa.

> **Por que não usar `[rbp-4]` desde já, então?** Porque a mecânica de **por que** variáveis locais ficam em deslocamentos de `RBP` (e como esses deslocamentos são organizados) é justamente o assunto do próximo módulo, o Módulo 9 (Stack). Introduzir isso agora, no meio de aprender controle de fluxo, misturaria dois assuntos. Por ora, basta saber: `[i]` representa "onde quer que a variável `i` esteja guardada na memória"; o **como** ela chega lá será detalhado em breve.

Um exemplo rápido comparando as duas formas, só para deixar a equivalência clara:

```asm
mov dword [i], 10          ; forma simbólica (usada neste módulo)
mov dword [rbp-4], 10      ; forma real equivalente, assumindo que 'i' está no deslocamento -4 de RBP
```

As duas linhas fazem exatamente a mesma coisa; a primeira só é mais fácil de ler enquanto o foco está em entender **fluxo de execução**, não em calcular deslocamentos de stack.

## 2. `jmp` — desvio incondicional

### O que faz

Altera `RIP` incondicionalmente: sempre desvia, sem depender de nenhuma flag.

### Sintaxe

```asm
jmp rótulo
```

Onde `rótulo` é um marcador textual em algum ponto do código (uma "etiqueta" que o assembler traduz para um endereço).

### Exemplo

```asm
jmp fim
mov eax, 1     ; esta linha é PULADA, nunca executada
fim:
mov eax, 2     ; execução continua daqui
```

Depois de `jmp fim`, `RIP` passa a apontar diretamente para a instrução logo após o rótulo `fim:`; a linha `mov eax, 1` nunca é alcançada.

## 3. Desvios condicionais: a família `j__`

### 3.1 A ideia geral

Um desvio condicional só altera `RIP` **se** uma combinação específica de flags for verdadeira. Caso contrário, a execução simplesmente segue para a instrução seguinte, como se o `j__` não existisse.

```asm
cmp eax, ebx
je  rotulo       ; desvia para 'rotulo' SE ZF = 1 (ou seja, se EAX == EBX)
mov ecx, 0        ; executado apenas se NÃO desviou
```

Existem dez instruções nesta família, mas todas seguem o mesmo mecanismo: **testar uma combinação específica de flags** (que já vimos no Módulo 7) **e desviar apenas se ela for verdadeira**. Nas próximas subseções, vamos ver cada grupo com calma: em vez de apresentar só uma tabela, cada instrução vem com seu próprio exemplo mínimo, mostrando os dois caminhos possíveis (quando desvia, e quando não desvia).

### 3.2 Igualdade: `je` e `jne`

Este é o par mais simples, porque não depende de interpretar os valores como signed ou unsigned: igualdade é igualdade, não importa como os bits são lidos (isso já foi visto no Módulo 7, Seção 7.1).

**`je`** (*jump if equal*, "salte se igual") desvia quando `ZF = 1`, ou seja, quando a subtração feita pelo `cmp` anterior deu exatamente zero.

```asm
mov eax, 5
mov ebx, 5
cmp eax, ebx
je  sao_iguais     ; ZF = 1 (5-5=0) → DESVIA
mov ecx, 0          ; esta linha é pulada
sao_iguais:
mov ecx, 1
```

**`jne`** (*jump if not equal*, "salte se diferente") faz exatamente o oposto: desvia quando `ZF = 0`.

```asm
mov eax, 5
mov ebx, 8
cmp eax, ebx
jne sao_diferentes  ; ZF = 0 (5-8=-3, não é zero) → DESVIA
mov ecx, 0            ; esta linha é pulada
sao_diferentes:
mov ecx, 1
```

Repare que `je` e `jne` são exatamente opostos: em qualquer par `cmp` + um dos dois, exatamente um deles desvia, e o outro não; nunca os dois, nunca nenhum.

### 3.3 Comparações signed: `jg`, `jge`, `jl`, `jle`

Este grupo usa as letras **g** (*greater*, "maior") e **l** (*less*, "menor"), e são usadas quando os valores comparados devem ser interpretados **com sinal** (Módulo 4, Seção 5), ou seja, quando negativos são possíveis e relevantes. Elas leem a combinação de SF e OF que vimos no Módulo 7 (Seção 7.3), não apenas uma flag isolada.

**`jg`** (*jump if greater*, "salte se maior") desvia quando o primeiro operando do `cmp` era, de fato, maior que o segundo (`ZF = 0` e `SF = OF`).

```asm
mov eax, 10
mov ebx, 3
cmp eax, ebx
jg  eax_maior      ; 10 > 3 (signed) → DESVIA
mov ecx, 0
eax_maior:
mov ecx, 1
```

**`jl`** (*jump if less*, "salte se menor") desvia quando o primeiro era menor (`SF != OF`).

```asm
mov eax, 3
mov ebx, 10
cmp eax, ebx
jl  eax_menor      ; 3 < 10 (signed) → DESVIA
mov ecx, 0
eax_menor:
mov ecx, 1
```

**`jge`** (*jump if greater or equal*, "salte se maior ou igual") e **`jle`** (*jump if less or equal*, "salte se menor ou igual") são variações que também desviam no caso de igualdade, como os próprios nomes sugerem. `jge` desvia quando `SF = OF` (o que cobre tanto "maior" quanto "igual"); `jle` desvia quando `ZF = 1` **ou** `SF != OF` (cobrindo "igual" ou "menor").

```asm
mov eax, 7
mov ebx, 7
cmp eax, ebx
jge iguais_ou_maior   ; 7 >= 7 → DESVIA (aqui, por causa da igualdade)
mov ecx, 0
iguais_ou_maior:
mov ecx, 1
```

> **Truque para não confundir a letra com o sentido:** pense em **g**/**l** como as iniciais de *greater*/*less* mesmo em português: "**g**rande" para `g` e "**l**imitado/pequeno" para `l` ajuda a fixar a associação sem depender de lembrar o inglês exato.

### 3.4 Comparações unsigned: `ja`, `jae`, `jb`, `jbe`

Este grupo usa as letras **a** (*above*, "acima") e **b** (*below*, "abaixo"), e são usadas quando os valores devem ser interpretados **sem sinal**: todos os valores são tratados como não-negativos, mesmo que o padrão de bits, lido como signed, pareceria negativo. Elas leem a flag CF, vista no Módulo 7 (Seção 7.2).

**`ja`** (*jump if above*, "salte se acima") desvia quando o primeiro operando era maior, tratando ambos como unsigned (`CF = 0` e `ZF = 0`).

```asm
mov al, 200      ; tratado como unsigned: 200
mov bl, 50
cmp al, bl
ja  eax_maior      ; 200 > 50 (unsigned) → DESVIA
mov ecx, 0
eax_maior:
mov ecx, 1
```

**`jb`** (*jump if below*, "salte se abaixo") desvia quando o primeiro era menor, unsigned (`CF = 1`).

```asm
mov al, 50
mov bl, 200
cmp al, bl
jb  eax_menor      ; 50 < 200 (unsigned) → DESVIA
mov ecx, 0
eax_menor:
mov ecx, 1
```

Assim como no grupo anterior, **`jae`** (*jump if above or equal*) e **`jbe`** (*jump if below or equal*) são as variantes que também desviam em caso de igualdade.

> **Por que existem dois grupos inteiros (g/l e a/b) para basicamente a mesma ideia de "maior"/"menor"?** Porque, como vimos em detalhe no Módulo 7, o mesmo padrão de bits pode representar números diferentes dependendo da interpretação (Módulo 4, Seção 5). O grupo g/l confia em SF combinada com OF (a forma correta de comparar COM sinal); o grupo a/b confia em CF (a forma correta de comparar SEM sinal). Usar o grupo errado para o tipo de dado gera comparações silenciosamente incorretas; por isso os compiladores escolhem cuidadosamente entre um grupo e outro, dependendo de a variável original em C ser `int` (signed) ou `unsigned int`.

### 3.5 Tabela-resumo

Agora que cada instrução foi vista individualmente, esta tabela funciona como referência rápida: a mesma informação das subseções acima, condensada.

| Instrução | Significa | Condição de flags | Contexto |
|---|---|---|---|
| `je` | *jump if equal* | ZF = 1 | Independe de signed/unsigned |
| `jne` | *jump if not equal* | ZF = 0 | Independe de signed/unsigned |
| `jg` | *jump if greater* | ZF = 0 e SF = OF | Signed |
| `jge` | *jump if greater or equal* | SF = OF | Signed |
| `jl` | *jump if less* | SF != OF | Signed |
| `jle` | *jump if less or equal* | ZF = 1 ou SF != OF | Signed |
| `ja` | *jump if above* | CF = 0 e ZF = 0 | Unsigned |
| `jae` | *jump if above or equal* | CF = 0 | Unsigned |
| `jb` | *jump if below* | CF = 1 | Unsigned |
| `jbe` | *jump if below or equal* | CF = 1 ou ZF = 1 | Unsigned |

> **Padrão para memorizar (sem decorar a tabela inteira):** instruções com **g**/**l** (*greater*/*less*) são para comparações **signed**. Instruções com **a**/**b** (*above*/*below*) são para comparações **unsigned**. Esse é literalmente o mecanismo pelo qual, ao ler Assembly, é possível inferir se uma variável em C era `int` ou `unsigned int`; a escolha entre `jg`/`ja`, por exemplo, é uma pista direta.

### 3.6 Exemplo comentado — `if (a == b)`

```c
if (a == b) {
    x = 1;
}
```

```asm
cmp eax, ebx     ; compara a (EAX) com b (EBX)
jne fim           ; se a != b, PULE o corpo do if
mov dword [x], 1  ; corpo do if: x = 1
fim:
```

Note o padrão: o compilador frequentemente **inverte** a condição do C. O código C diz "se a == b, faça X". O Assembly diz "se a != b, **pule** X", ou seja, o desvio condicional testa o **oposto** da condição C, para pular o corpo quando a condição original é falsa. Isso é extremamente comum e vale destacar como um padrão de leitura, não uma coincidência deste exemplo.

## 4. Reconhecendo `if` simples

### Padrão geral

```
cmp / test
j<condição-oposta> rótulo_fim
... corpo do if ...
rótulo_fim:
```

### Exemplo: `if (a > 10)`

```c
if (a > 10) {
    b = 1;
}
```

```asm
cmp eax, 10
jle fim          ; oposto de "maior que" é "menor ou igual" → pula se NÃO for maior
mov dword [b], 1
fim:
```

Processo de leitura, aplicando o que já vimos:

1. **Identificar a comparação**: `cmp eax, 10`, que compara EAX com o valor 10.
2. **Identificar o desvio condicional**: `jle`, que significa "salta se menor ou igual" (signed, pela tabela).
3. **Inverter mentalmente**: se o desvio é "pule se `<=`", a condição original do C é o oposto: "`>`". Ou seja, o corpo abaixo do `jle` só executa quando `EAX > 10`.
4. **Ler o corpo**: `mov dword [b], 1` é executado apenas nesse caso.

## 5. Reconhecendo `if / else`

### Padrão geral

```
cmp / test
j<condição-oposta> rótulo_else
... corpo do if ...
jmp rótulo_fim
rótulo_else:
... corpo do else ...
rótulo_fim:
```

Note o `jmp` extra no final do bloco `if`: ele é necessário para que, depois de executar o corpo do `if`, a execução **não "caia" também** no corpo do `else`; sem ele, a execução simplesmente continuaria sequencialmente para dentro do bloco `else`.

### Exemplo: `if (a > b) { x = 1; } else { x = 2; }`

```c
if (a > b) {
    x = 1;
} else {
    x = 2;
}
```

```asm
cmp eax, ebx
jle senao          ; se NÃO for a > b (ou seja, a <= b), pula para 'senao'
mov dword [x], 1    ; corpo do if
jmp fim              ; pula o bloco else
senao:
mov dword [x], 2    ; corpo do else
fim:
```

## 6. `if` vs. `while`: a mesma "forma", propósitos diferentes

Antes de avançar para laços em detalhe, vale um contraste direto, já que, olhando rápido, um `if` simples e um `while` podem parecer estruturalmente parecidos (ambos têm `cmp` + `j<condição>` + um bloco). A diferença real está em **para onde o fluxo aponta depois do bloco**.

```asm
; IF simples
cmp eax, 10
jle fim
mov dword [b], 1
fim:
; (fluxo segue em frente, não volta)
```

```asm
; WHILE
jmp verificacao
inicio_loop:
mov dword [b], 1
verificacao:
cmp eax, 10
jg inicio_loop
; (fluxo segue em frente, IGUAL ao if, mas antes disso pode ter voltado várias vezes)
```

A diferença crucial: no `if`, o rótulo de desvio (`fim`) está **depois** do bloco e nunca é revisitado: o fluxo é estritamente "para frente". No `while`, o rótulo de desvio de retorno (`inicio_loop`) está **antes** do bloco, e o `j<condição>` no final aponta **de volta** para ele, criando um ciclo. Em outras palavras:

> **Regra prática:** se o alvo de um `j<condição>` está **atrás** dele no código (endereço menor), é um laço. Se está **à frente** (endereço maior), é um desvio de `if`/`if-else`. Essa é, sozinha, a forma mais rápida de diferenciar os dois padrões ao correr o olho por um trecho de Assembly.

## 7. Reconhecendo `while`

### Padrão geral

O `while` testa a condição **antes** de cada iteração, inclusive antes da primeira. Um padrão comum de compilação:

```
jmp verificacao
inicio_loop:
... corpo do while ...
verificacao:
cmp / test
j<condição-original> inicio_loop
```

Repare que aqui o desvio no final do bloco usa a condição **original** (não invertida), porque estamos decidindo se **voltamos** para o início do corpo, não se pulamos ele.

### Exemplo: `while (i < 10) { i++; }`

```c
int i = 0;
while (i < 10) {
    i++;
}
```

```asm
mov dword [i], 0
jmp verificacao
inicio_loop:
inc dword [i]
verificacao:
cmp dword [i], 10
jl inicio_loop      ; se i < 10, volta para o início do corpo
```

Processo de leitura:

1. `i` é inicializado com 0.
2. `jmp verificacao`: o programa vai direto checar a condição, **antes** de executar o corpo pela primeira vez (isso é a essência do `while`, diferente do `do-while`, que veremos a seguir).
3. O corpo (`inc dword [i]`) só é alcançado se a verificação, na primeira ou em iterações seguintes, permitir.
4. `cmp dword [i], 10` seguido de `jl inicio_loop`: "se `i < 10` (signed), volte para o início do corpo". Isso repete o processo.
5. Quando `i` finalmente chega a 10, `jl` não desvia mais, e a execução simplesmente segue para a instrução após o `jl` (fora do laço).

## 8. Reconhecendo `do-while`

### Padrão geral

Diferente do `while`, o `do-while` executa o corpo **pelo menos uma vez**, testando a condição só **depois**:

```
inicio_loop:
... corpo do do-while ...
cmp / test
j<condição-original> inicio_loop
```

Note que não há o `jmp verificacao` inicial que existia no `while`; a execução simplesmente começa direto no corpo.

### Exemplo: `do { i++; } while (i < 10);`

```c
int i = 0;
do {
    i++;
} while (i < 10);
```

```asm
mov dword [i], 0
inicio_loop:
inc dword [i]
cmp dword [i], 10
jl inicio_loop
```

### `while` vs. `do-while`, lado a lado

Os dois exemplos acima (Seção 7 e esta seção) compilam praticamente a **mesma lógica em C** (`i` incrementado até chegar a 10), com a única diferença sendo `while` vs `do-while`. Vale colocar os dois Assemblys um do lado do outro:

```asm
; while (i < 10) { i++; }         ; do { i++; } while (i < 10);
mov dword [i], 0                   mov dword [i], 0
jmp verificacao                    inicio_loop:
inicio_loop:                       inc dword [i]
inc dword [i]                      cmp dword [i], 10
verificacao:                       jl inicio_loop
cmp dword [i], 10
jl inicio_loop
```

A única diferença real: a presença (ou ausência) do `jmp verificacao` logo após a inicialização. Tudo o mais é idêntico: mesmo corpo, mesma comparação, mesmo desvio de retorno. Isso confirma a regra prática:

> **Diferença estrutural chave entre `while` e `do-while`, em Assembly:** a ausência do `jmp` inicial "pulando para a verificação" é o sinal mais confiável de que se está diante de um `do-while`, e não de um `while`. Um erro de leitura comum é assumir "tem um `cmp` no fim de um bloco que aponta para trás, então é `do-while`", mas isso também é verdade para o `while`. **Sempre confira o início do bloco**, não só o fim, antes de decidir entre os dois.

## 9. Reconhecendo `for`

### A ideia central

Um `for` em C, como:

```c
for (inicializacao; condicao; incremento) {
    corpo;
}
```

é, conceitualmente, **açúcar sintático** para um `while` com a inicialização antes e o incremento no final do corpo:

```c
inicializacao;
while (condicao) {
    corpo;
    incremento;
}
```

Por isso, o Assembly gerado por um `for` costuma seguir **exatamente o mesmo padrão** do `while` (Seção 7); a diferença está em **onde**, no código C original, cada parte veio, não em como o Assembly se estrutura.

### Exemplo 1: `for (int i = 0; i < 5; i++) { soma += i; }`

```c
int soma = 0;
for (int i = 0; i < 5; i++) {
    soma += i;
}
```

```asm
mov dword [soma], 0
mov dword [i], 0        ; inicialização
jmp verificacao
inicio_loop:
mov eax, dword [i]
add dword [soma], eax   ; corpo: soma += i
inc dword [i]            ; incremento
verificacao:
cmp dword [i], 5
jl inicio_loop
```

Processo de leitura: a chave é identificar as **três partes** do `for` dentro da estrutura de `while` que já conhecemos.

1. **Inicialização**: `mov dword [i], 0`, antes do `jmp verificacao`.
2. **Corpo real**: `mov eax, [i]` / `add [soma], eax`.
3. **Incremento**: `inc dword [i]`, executado **dentro** do laço, logo após o corpo. Essa é a parte que só existe por causa do `for`; em um `while` equivalente, precisaria estar escrita manualmente no C.
4. **Condição**: `cmp dword [i], 5` / `jl inicio_loop`, no mesmo formato do `while`.

### Exemplo 2 — um `for` menos óbvio: decrescente, com `<=`

O exemplo anterior tem um incremento bem visível (`inc`) e uma condição direta (`jl`). Mas o padrão precisa ser reconhecido mesmo quando essas partes não são tão óbvias. Considere:

```c
for (int i = 10; i <= 0; i--) {
    ; corpo omitido para focar na estrutura
}
```

Antes de traduzir, repare: `i <= 0` começando de `10` e **decrescendo** já parece estranho (esse laço, matematicamente, nunca executaria, pois `10 <= 0` é falso de cara). Isso é proposital: leitura de Assembly às vezes exige notar quando a lógica original em C parece ter um bug, ou quando o próprio leitor entendeu mal a condição. Vamos usar uma versão que realmente executa, para focar na estrutura:

```c
for (int i = 10; i >= 0; i--) {
    ; corpo omitido
}
```

```asm
mov dword [i], 10        ; inicialização
jmp verificacao
inicio_loop:
; ... corpo omitido ...
dec dword [i]              ; incremento (aqui, decremento)
verificacao:
cmp dword [i], 0
jge inicio_loop            ; "i >= 0" -> jge (signed, "greater or equal")
```

O que muda em relação ao Exemplo 1:

- O "incremento" da Seção 9 aqui é, na verdade, um **decremento** (`dec`, não `inc`). O padrão estrutural (uma instrução isolada, logo antes do rótulo de verificação, mexendo na variável de controle) continua o mesmo; só a operação específica muda.
- A condição `i >= 0` gera `jge`, não `jl`, mas ainda está exatamente na mesma posição estrutural (fim do bloco, apontando de volta para `inicio_loop`).

> **Lição de leitura:** não procure por `inc` e `jl` especificamente como "assinatura" de um `for`; procure pela **posição estrutural** (uma instrução isolada de ajuste da variável de controle, seguida por uma comparação e um desvio condicional apontando para trás). A instrução exata (`inc`/`dec`, `jl`/`jle`/`jg`/`jge`) depende inteiramente da direção e dos limites do laço original em C.

## 10. Reconhecendo `switch`

`switch` é o mais variável dos quatro construtos: compiladores podem gerar Assembly bem diferente dependendo do número de casos e de quão "densos" (sequenciais) eles são. Vamos ver os dois padrões mais comuns.

### 10.1 Padrão 1 — cadeia de comparações (poucos casos, ou casos esparsos)

Quando há poucos `case`s, ou valores muito espalhados, o compilador costuma gerar uma sequência de comparações, parecida com vários `if/else if` encadeados:

```c
switch (x) {
    case 1: y = 10; break;
    case 2: y = 20; break;
    default: y = 0;
}
```

```asm
mov eax, dword [x]
cmp eax, 1
jne checa_case2
mov dword [y], 10
jmp fim_switch
checa_case2:
cmp eax, 2
jne caso_default
mov dword [y], 20
jmp fim_switch
caso_default:
mov dword [y], 0
fim_switch:
```

Note que essa estrutura é, na prática, **idêntica** a uma cadeia de `if/else if/else` (Seção 5), porque, do ponto de vista da CPU, é exatamente isso que está acontecendo. O `break` de cada `case`, em Assembly, corresponde ao `jmp fim_switch` que evita "cair" no próximo bloco.

**Um cuidado de leitura importante:** ao encontrar uma cadeia de `cmp`/`jne` desse tipo isoladamente, **não é possível ter certeza absoluta**, só olhando o Assembly, se o código C original era um `switch` ou uma cadeia de `if/else if`. Ambos compilam para exatamente essa mesma forma. A única diferença notável, quando existe, é sutil: em um `switch` real, é mais comum que todas as comparações sejam contra o **mesmo registrador/variável** (`eax`, sempre comparado com valores diferentes); em uma cadeia de `if/else if`, é mais comum ver condições envolvendo variáveis diferentes em cada ramo. Ainda assim, essa é uma heurística, não uma garantia; na dúvida, ambas as leituras (`switch` ou `if/else if` equivalente) são aceitáveis ao descrever o comportamento em pseudocódigo.

### 10.2 Padrão 2 — jump table (muitos casos sequenciais)

Quando os valores de `case` são densos e sequenciais (por exemplo, `0, 1, 2, 3...`), compiladores frequentemente geram uma **tabela de saltos** (*jump table*): um array de endereços na memória, onde o índice do array corresponde ao valor testado, e cada posição contém o endereço de código correspondente àquele `case`.

```asm
; conceito simplificado (a sintaxe exata de tabelas varia bastante)
mov eax, dword [x]
cmp eax, 3          ; verifica se x está dentro da faixa esperada (0 a 3, por exemplo)
ja caso_default      ; se x > 3 (unsigned), nenhum case bate, vai para default
jmp [tabela + eax*8]  ; salta para o endereço guardado na posição 'eax' da tabela
```

Aqui, em vez de uma cadeia de comparações, o programa calcula um endereço (usando o próprio valor de `x` como índice, exatamente como vimos no Módulo 5 com arrays) e desvia diretamente para lá, sem testar cada `case` individualmente.

> Não é necessário dominar a sintaxe exata de jump tables agora; o importante é **reconhecer que esse padrão existe** e não se confundir ao encontrá-lo: se aparecer algo como `jmp [algum_endereço + registrador*8]` sem uma cadeia de `cmp`s antecedendo cada possibilidade, é provável que se trate de uma tabela de saltos representando um `switch` com muitos casos dentro de uma faixa.

## 11. Loops aninhados: quando um bloco contém outro

Código real frequentemente tem laços dentro de laços, ou uma condição dentro de um laço. Isso não introduz nenhum mecanismo novo; apenas exige manter o controle de **múltiplos rótulos ao mesmo tempo**, e prestar atenção em qual `j<condição>` pertence a qual bloco.

### Exemplo: `for` dentro de `for`

```c
for (int i = 0; i < 2; i++) {
    for (int j = 0; j < 3; j++) {
        total++;
    }
}
```

```asm
mov dword [i], 0
jmp verifica_i
loop_i:
    mov dword [j], 0
    jmp verifica_j
    loop_j:
        inc dword [total]
        inc dword [j]
    verifica_j:
        cmp dword [j], 3
        jl loop_j          ; fecha o laço INTERNO (j)
    inc dword [i]
verifica_i:
    cmp dword [i], 2
    jl loop_i               ; fecha o laço EXTERNO (i)
```

(A indentação acima é só para facilitar a leitura neste material. Assembly de verdade não exige indentação, mas segui-la mentalmente ao ler ajuda a não perder o alinhamento entre rótulos.)

### Como não se perder

1. **Nomeie (mentalmente) cada par `jmp verificação` / `j<condição> volta`**: cada par completo é um laço independente. Neste exemplo, há dois pares: `verifica_i`/`loop_i` (externo) e `verifica_j`/`loop_j` (interno).
2. **Identifique onde um laço "contém" o outro**, observando os endereços: o laço interno inteiro (`loop_j` até seu `jl`) deve estar posicionado **entre** o início e o fim do laço externo, nunca cruzando essa fronteira.
3. **Repare que `inc dword [i]`** aparece **depois** do laço interno inteiro ter terminado, mas **antes** de `verifica_i`, ou seja, ele só executa uma vez por iteração do laço externo, depois que todas as iterações do laço interno já rodaram. Isso é consistente com o C: o incremento de `i` só acontece depois que o corpo completo (incluindo o `for` interno) termina.

> **Regra prática para loops aninhados:** conte os pares `jmp verificação` / `j<condição>`. Cada par é um laço. Depois, observe o aninhamento pela posição relativa dos rótulos: um laço "dentro" de outro sempre tem seu par inteiro (início e fim) posicionado entre o início e o fim do laço que o contém, nunca ultrapassando essa fronteira.

## 12. Erros comuns de leitura neste módulo

Reunindo os erros mais frequentes ao interpretar controle de fluxo, para revisão rápida:

- **Confundir `while` com `do-while`** por olhar só o fim do bloco (`cmp` + `j<condição>` apontando para trás) e ignorar se existe um `jmp` de verificação **antes** do corpo. Sempre confira o início do bloco, não só o fim (Seção 8).
- **Assumir que toda cadeia de `cmp`/`jne` é um `switch`**: pode perfeitamente ser uma cadeia de `if/else if` do C original; sem o código-fonte, os dois padrões são indistinguíveis com certeza (Seção 10.1).
- **Tentar identificar um `for` procurando literalmente por `inc` e `jl`**: a instrução de ajuste pode ser `dec`, e a condição pode ser `jge`, `jle`, `jg`, dependendo da direção e dos limites do laço original. O que importa é a **posição estrutural** do ajuste e da comparação, não a instrução exata (Seção 9, Exemplo 2).
- **Perder a referência em loops aninhados**, confundindo qual `j<condição>` fecha qual laço. Contar os pares `jmp verificação`/`j<condição> volta` e observar o aninhamento pela posição dos rótulos evita esse erro (Seção 11).
- **Esquecer que o desvio condicional de um `if` testa a condição oposta** à do C original: ao ler `jle`, lembrar que o C provavelmente dizia `if (... > ...)`, não `if (... <= ...)` (Seção 3).

## 13. Processo de leitura consolidado

Juntando tudo que vimos até aqui neste módulo, o processo para identificar qual construto de controle de fluxo está sendo lido:

1. **O alvo do `j<condição>` está atrás dele (endereço menor) ou à frente (endereço maior)?** Atrás → laço. À frente → `if`/`if-else` (Seção 6).
2. **Se for um laço: existe `jmp` para uma verificação logo no início, antes do corpo?** Se sim, é `while`/`for` (testa antes). Se não, é `do-while` (testa depois) (Seções 7 e 8).
3. **Se for `while`/`for`: existe uma instrução isolada de ajuste de variável (`inc`, `dec`, ou uma conta) logo antes do rótulo de verificação, sem relação com o "corpo lógico" do laço?** Se sim, é provavelmente um `for` com essa parte funcionando como incremento/decremento (Seção 9).
4. **Se for `if`: existe um `jmp` logo após o bloco, pulando para depois de um segundo bloco rotulado?** É um `if/else` (Seção 5). Se não, é um `if` simples (Seção 4).
5. **Existe uma cadeia de `cmp` seguidos de `jne`/`je`, cada um levando a um bloco terminado em `jmp fim`, todos comparando o mesmo registrador?** Pode ser um `switch` (padrão de cadeia), ou uma sequência de `if/else if` do próprio C; a única forma de ter certeza é ver o C original (Seção 10.1).
6. **Existe um `jmp` que usa um endereço calculado (`[tabela + registrador*escala]`)?** É um `switch` com jump table (Seção 10.2).
7. **Há múltiplos pares `jmp verificação`/`j<condição> volta`, um contido dentro do outro?** São laços aninhados; trate cada par independentemente, prestando atenção em qual instrução de ajuste/incremento pertence a qual laço (Seção 11).

## 14. Exercícios

### Nível 1 — Reconhecer instruções

1. Qual é a diferença fundamental entre `jmp` e `je`?
2. Por que compiladores costumam inverter a condição do C ao gerar o desvio condicional de um `if`?
3. Ao ver um `j<condição>` cujo alvo está **atrás** dele no código, o que isso sugere sobre o construto envolvido?

### Nível 5 — Interpretar uma função/bloco completo

Para o trecho abaixo, siga o processo de leitura da Seção 13 e responda:

```asm
mov dword [i], 0
jmp verificacao
inicio_loop:
mov eax, dword [i]
imul eax, eax
mov dword [quadrado], eax
inc dword [i]
verificacao:
cmp dword [i], 4
jl inicio_loop
```

4. Que construto de controle de fluxo (if, while, do-while, for) este trecho representa? Justifique com base nos sinais estruturais (não apenas no "parece").
5. Escreva o pseudocódigo/C aproximado que este trecho representa.

### Nível 6 — Reconstruir lógica de controle

```asm
cmp eax, ebx
jge maior_ou_igual
mov dword [resultado], 1
jmp fim
maior_ou_igual:
mov dword [resultado], 2
fim:
```

6. Reconstrua o código C aproximado (`if/else`) que gera este Assembly.

```asm
mov eax, dword [x]
cmp eax, 1
jne checa_b
mov dword [y], 100
jmp fim_cadeia
checa_b:
cmp eax, 2
jne fim_cadeia
mov dword [y], 200
fim_cadeia:
```

7. Este trecho pode ser descrito tanto como um `switch` quanto como uma cadeia de `if/else if`. Escreva as duas versões equivalentes em C.

### Nível 7 — Reconstruir pseudocódigo (com sinal e comparações)

```asm
mov eax, dword [x]
cmp eax, 0
jge nao_negativo
neg eax
nao_negativo:
mov dword [abs_x], eax
```

8. O que este trecho calcula, em termos gerais? (Dica: pense no que `neg` faz, Módulo 6 Parte 2, e em que situação ele é aplicado aqui.)

### Nível 8 — Interpretar um programa pequeno desconhecido (loop aninhado)

```asm
mov dword [i], 0
jmp verifica_i
loop_i:
mov dword [j], 0
jmp verifica_j
loop_j:
inc dword [contador]
inc dword [j]
verifica_j:
cmp dword [j], 2
jl loop_j
inc dword [i]
verifica_i:
cmp dword [i], 3
jl loop_i
```

9. Quantos laços existem neste trecho? Como identificar o aninhamento entre eles?
10. Ao final da execução, qual é o valor de `contador` (assumindo que começa em 0)? Justifique contando as iterações.

---

## 15. Respostas

1. `jmp` sempre desvia, incondicionalmente. `je` só desvia se `ZF = 1` (ou seja, se a comparação anterior indicou igualdade); caso contrário, a execução simplesmente segue para a instrução seguinte.
2. Porque a estrutura mais eficiente, em termos de instruções, é "pular o corpo do if quando a condição é falsa", ou seja, o desvio condicional testa quando **não** executar o bloco, e deixa a execução "cair" (fluir sequencialmente) para dentro do bloco quando a condição é verdadeira. Codificar dessa forma evita um `jmp` extra que seria necessário se a lógica fosse "se verdadeiro, pule para o corpo".
3. Sugere que se trata de um **laço** (o desvio está "voltando" para repetir um bloco de código já executado), e não de um `if`/`if-else` (onde o desvio sempre aponta para frente, nunca revisitando código anterior).
4. É um **for** (mais especificamente, o padrão Assembly de um `while`, mas com um "incremento" (`inc dword [i]`) presente dentro do corpo, logo antes da verificação, e não descrito por nenhuma lógica além do controle do laço). O sinal estrutural principal: existe um `jmp verificacao` logo no início, antes de qualquer execução do corpo, o que indica "testar antes de rodar pela primeira vez" (característico de `while`/`for`, não `do-while`). Além disso, `inc dword [i]` aparece isolado, sem relação com o cálculo do quadrado, sugerindo que é apenas o incremento de uma variável de controle de laço, típico de um `for`.
5. 
```c
for (int i = 0; i < 4; i++) {
    quadrado = i * i;
}
```
6. 
```c
if (a >= b) {
    resultado = 2;
} else {
    resultado = 1;
}
```
(Note que o `jge` no Assembly pula para o bloco de `resultado = 2` quando `a >= b`; o bloco padrão, sem desvio, atribui `resultado = 1`, que corresponde ao `else` no C, já que a condição testada no Assembly, `a >= b`, é o oposto do `else` do `if (a < b)` equivalente. Uma forma alternativa e igualmente válida de descrever este trecho seria `if (a < b) { resultado = 1; } else { resultado = 2; }`; ambas descrevem a mesma lógica.)
7. Como `switch`:
```c
switch (x) {
    case 1: y = 100; break;
    case 2: y = 200; break;
}
```
Como `if/else if`:
```c
if (x == 1) {
    y = 100;
} else if (x == 2) {
    y = 200;
}
```
Ambas geram Assembly estruturalmente equivalente ao trecho dado; sem o código-fonte original, não há como determinar com certeza qual delas foi realmente escrita.
8. Calcula o **valor absoluto** de `x`. Se `x >= 0` (verificado por `cmp eax, 0` / `jge`), o valor não é alterado. Se `x < 0` (ou seja, o `jge` **não** desviou), a instrução `neg eax` é executada, transformando o valor negativo em seu equivalente positivo (Módulo 6, Parte 2, Seção 5.4). Em C, isso corresponde a algo como `abs_x = (x >= 0) ? x : -x;`, ou uma implementação manual da função `abs()`.
9. Existem **dois laços**: um externo (controlado por `i`, par `verifica_i`/`loop_i`) e um interno (controlado por `j`, par `verifica_j`/`loop_j`). O aninhamento é identificado porque o par completo do laço interno (`loop_j` até seu `jl loop_j`) está posicionado inteiramente **entre** o início (`loop_i`) e o fim (`jl loop_i`) do laço externo, nunca ultrapassando essas fronteiras.
10. O laço externo (`i`) roda enquanto `i < 3`, ou seja, 3 vezes (`i = 0, 1, 2`). Para cada uma dessas 3 vezes, o laço interno (`j`) roda enquanto `j < 2`, ou seja, 2 vezes por iteração externa. `contador` é incrementado uma vez a cada iteração do laço interno. Total: `3 × 2 = 6`. Valor final de `contador`: **6**.

---

*Módulo anterior: [Módulo 7 — Flags](./07-flags.md)*
*Próximo módulo: [Módulo 9 — Stack](./09-stack.md)*
