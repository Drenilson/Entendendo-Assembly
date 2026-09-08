# Módulo 11 — C → Assembly

> **Arquitetura:** x86-64
> **Sistema operacional de referência:** Linux
> **ABI:** System V AMD64
> **Sintaxe:** Intel

## 1. Introdução

Este módulo não introduz novos conceitos fundamentais de arquitetura. Ele reúne os conceitos já estudados, registradores, memória, instruções, flags, controle de fluxo, stack e funções, em torno de pequenos trechos de C e do Assembly correspondente, e introduz apenas algumas instruções auxiliares, necessárias para reconhecer formas alternativas de código gerado pelo compilador.

A ideia central deste módulo, e talvez a mais importante de todo o curso, é esta:

> **Não existe uma correspondência de um para um entre código C e código Assembly.**

O mesmo trecho de C pode gerar Assembly bem diferente dependendo do compilador, da versão dele, e principalmente do nível de otimização usado. E, na direção oposta, o mesmo Assembly às vezes pode ter vindo de mais de uma forma diferente de escrever o C original. Ler Assembly não é decodificar uma tradução exata, é reconstruir um comportamento plausível.

## 2. `int soma(int a, int b)`: revisitando com duas versões

Os Módulos 9 e 10 já mostraram esta função em detalhe:

```c
int soma(int a, int b) {
    int resultado = a + b;
    return resultado;
}
```

### 2.1 Versão sem otimização (a que o curso usou até aqui)

```asm
soma:
    push rbp
    mov rbp, rsp
    sub rsp, 16
    mov dword [rbp-4], edi
    mov dword [rbp-8], esi
    mov eax, dword [rbp-4]
    add eax, dword [rbp-8]
    mov dword [rbp-12], eax
    mov eax, dword [rbp-12]
    mov rsp, rbp
    pop rbp
    ret
```

### 2.2 Versão com otimização

Um compilador com otimizações ativadas tende a produzir algo radicalmente mais enxuto para exatamente a mesma função:

```asm
soma:
    lea eax, [rdi+rsi]
    ret
```

Duas instruções, contra as onze da versão anterior, com exatamente o mesmo comportamento observável. Isso é possível porque `lea` calcula uma expressão de endereçamento sem tocar a memória (Módulo 6, Parte 1).

> `lea eax, [rdi+rsi]` não significa acessar o endereço calculado. Significa apenas calcular `RDI + RSI` e colocar esse resultado em `EAX`, sem nenhuma leitura de memória envolvida.

Essa expressão pode, portanto, ser usada como uma calculadora: `rdi+rsi` soma os dois argumentos diretamente, sem precisar de variáveis locais, sem stack frame, e o resultado já sai pronto em `EAX`, exatamente onde o valor de retorno é esperado (Módulo 10).

> Este é o primeiro exemplo concreto da ideia central deste módulo: a mesma função em C, duas formas completamente diferentes de Assembly, o mesmo comportamento.

## 3. `if (x > 10)`: duas formas de decidir

```c
int maior_que_dez(int x) {
    if (x > 10) {
        return 1;
    }
    return 0;
}
```

### 3.1 Versão com desvio condicional (o padrão do Módulo 8)

```asm
maior_que_dez:
    push rbp
    mov rbp, rsp
    mov dword [rbp-4], edi
    cmp dword [rbp-4], 10
    jle nao_maior
    mov eax, 1
    jmp fim
nao_maior:
    mov eax, 0
fim:
    pop rbp
    ret
```

### 3.2 Versão sem desvio, usando `setcc` e `movzx`

Existe uma segunda forma comum, especialmente em código otimizado, que evita desvios por completo. Ela usa duas instruções que ainda não apareceram no curso, e que vale introduzir aqui, seguindo o mesmo formato usado desde o Módulo 6.

**`setcc` (a família `sete`, `setne`, `setg`, `setl`...)**

*O que faz:* escreve `1` ou `0` em um operando de 1 byte, dependendo exatamente da mesma combinação de flags que a instrução `jcc` correspondente testaria (Módulo 8, Seção 3). Em vez de desviar, ela simplesmente registra o resultado da comparação como um valor.

*Sintaxe:* `setg destino_de_1_byte` (e as variações `sete`, `setne`, `setl`, `setge`, `setle`, para citar as mais comuns, seguindo exatamente as mesmas letras e o mesmo significado de flags já vistos na tabela do Módulo 8).

**`movzx` (*move with zero extend*)**

*O que faz:* copia um valor menor para um destino maior, preenchendo os bits extras com zero. Isso resolve exatamente o problema visto no Módulo 3 (Seção 3.2): escrever em um registrador de 8 ou 16 bits, como `AL`, não limpa o restante do registrador. `movzx` existe para fazer essa limpeza de forma explícita, quando necessário.

*Sintaxe:* `movzx destino_maior, origem_menor`

> `setcc` sozinha já produz corretamente um byte `0` ou `1` em `AL`; ela não exige `movzx` depois por obrigação. A necessidade aparece quando esse resultado precisa ser interpretado como um valor maior, como um `int` de 32 bits, que é exatamente o caso do exemplo a seguir, já que a função retorna `int`.

### 3.3 Aplicando as duas instruções

```asm
maior_que_dez:
    cmp edi, 10
    setg al          ; AL = 1 se EDI > 10 (signed), senão AL = 0
    movzx eax, al     ; zera os bits 8-31 de EAX, preservando apenas o 0 ou 1 de AL
    ret
```

Passo a passo:

1. `cmp edi, 10` calcula `EDI - 10` e atualiza as flags, exatamente como em qualquer comparação (Módulo 7).
2. `setg al` verifica a mesma combinação de flags que `jg` verificaria (`ZF = 0` e `SF = OF`), e grava `1` em `AL` se verdadeiro, `0` caso contrário, sem nenhum desvio.
3. `movzx eax, al` garante que os bits acima do byte baixo de `EAX` estejam zerados, produzindo um `int` limpo (0 ou 1) como valor de retorno.

> Sem o `movzx`, `EAX` poderia conter lixo de uma instrução anterior nos bits 8 a 31, mesmo com `AL` correto. Isso conecta diretamente à regra do Módulo 3: escrever em `AL` não afeta o restante do registrador, então, se um valor limpo de 32 bits é necessário, algo precisa cuidar disso explicitamente, e `movzx` é essa ferramenta.

Assim como no exemplo de `soma`, a mesma lógica em C, duas formas bem diferentes de Assembly: uma com desvios, outra sem nenhum.

## 4. `for (int i = 0; i < 10; i++)`: revisão rápida

O padrão de reconhecimento de `for` já foi coberto em profundidade no Módulo 8 (Seção 9): inicialização antes de um `jmp` para a verificação, corpo, incremento isolado antes do rótulo de verificação, e comparação com desvio de volta para o início. Este módulo não repete esse conteúdo, e vai usá-lo diretamente na próxima seção, combinado com acesso a arrays.

## 5. `int array[5]`: arrays como parâmetro, e o detalhe que muitos ignoram

Quando um array é declarado como parâmetro de uma função em C, como em `int soma_array(int array[], int tamanho)`, o compilador o trata exatamente como um ponteiro para o primeiro elemento, `int *array`. O tamanho declarado entre colchetes, quando presente em um parâmetro, é apenas decorativo, e é ignorado pelo compilador. Essa regra da própria linguagem C, e não do Assembly, costuma passar despercebida até que se observe o Assembly gerado e se note que o parâmetro chega em um único registrador de 8 bytes (um endereço), exatamente como qualquer outro ponteiro.

```c
int soma_array(int array[], int tamanho) {
    int total = 0;
    for (int i = 0; i < tamanho; i++) {
        total += array[i];
    }
    return total;
}
```

```asm
soma_array:
    push rbp
    mov rbp, rsp
    sub rsp, 32

    mov qword [rbp-8], rdi      ; array (um ponteiro, 8 bytes) vindo em RDI
    mov dword [rbp-12], esi      ; tamanho vindo em ESI
    mov dword [rbp-16], 0        ; total = 0
    mov dword [rbp-20], 0        ; i = 0
    jmp verificacao

loop_inicio:
    mov rax, qword [rbp-8]          ; RAX = array (o endereço base)
    mov ecx, dword [rbp-20]          ; ECX = i (escrever em ECX zera a metade alta de RCX, Módulo 3)
    mov edx, dword [rax+rcx*4]        ; EDX = array[i]
    mov eax, dword [rbp-16]
    add eax, edx
    mov dword [rbp-16], eax           ; total += array[i]
    inc dword [rbp-20]                 ; i++

verificacao:
    mov eax, dword [rbp-20]
    cmp eax, dword [rbp-12]
    jl loop_inicio

    mov eax, dword [rbp-16]
    mov rsp, rbp
    pop rbp
    ret
```

Este exemplo combina praticamente tudo visto até aqui: `array` chega como ponteiro em `RDI` (Módulo 10), é guardado localmente na stack (Módulo 9), o acesso a `array[i]` usa endereçamento com escala (`[rax+rcx*4]`, Módulo 5), e a estrutura do laço segue exatamente o padrão de `for` do Módulo 8. Repare também que `mov ecx, dword [rbp-20]` aproveita a regra do Módulo 3 (escrever em um registrador de 32 bits zera a metade alta de 64 bits), o que torna seguro usar `RCX` inteiro como índice na expressão de endereçamento logo em seguida, mesmo `ECX` tendo sido escrito, não `RCX` diretamente.

## 6. `int *p`: um ponteiro como parâmetro

```c
void incrementa(int *p) {
    *p = *p + 1;
}
```

```asm
incrementa:
    push rbp
    mov rbp, rsp
    mov qword [rbp-8], rdi      ; p (o ponteiro) vindo em RDI

    mov rax, qword [rbp-8]        ; RAX = p (o endereço em si)
    mov eax, dword [rax]           ; EAX = *p (dereferência, Módulo 5)
    add eax, 1
    mov rdx, qword [rbp-8]          ; RDX = p novamente
    mov dword [rdx], eax             ; *p = EAX (escreve de volta no endereço)

    mov rsp, rbp
    pop rbp
    ret
```

Note os dois momentos de dereferência: `mov eax, dword [rax]` lê o valor atual apontado por `p`; `mov dword [rdx], eax` escreve o novo valor no mesmo endereço. Isso é exatamente o mecanismo de leitura e escrita em memória do Módulo 5, agora aplicado a um ponteiro recebido como argumento de função. Como a função é `void`, não há valor de retorno relevante em `EAX` ao final.

## 7. A lição central, revisitada

Os exemplos deste módulo demonstram a mesma ideia por ângulos diferentes:

- `soma`: a mesma função C, duas versões de Assembly completamente diferentes em tamanho e estrutura (Seção 2).
- `maior_que_dez`: a mesma condição, uma versão com desvio e outra sem nenhum (Seção 3).
- O Módulo 8 já havia mostrado, na Seção 10.1, que uma cadeia de comparações pode ter vindo tanto de um `switch` quanto de uma sequência de `if/else if`, sem forma de distinguir com certeza apenas pelo Assembly.

O objetivo de ler Assembly não é reconstruir o texto exato do código-fonte original, isso, em geral, nem é possível. O objetivo é reconstruir um comportamento correto e equivalente, descrito em pseudocódigo ou C aproximado. Duas pessoas lendo o mesmo trecho de Assembly podem produzir descrições em C ligeiramente diferentes na forma, e ambas estarem certas, desde que capturem o mesmo comportamento.

## 8. O método de leitura consolidado

Ao longo do curso, cada módulo apresentou seu próprio processo de leitura, focado no assunto daquele módulo. Chegado o módulo final, vale reunir tudo em um único método, aplicável a qualquer função Assembly desconhecida, do início ao fim:

1. **Identificar a função.** Localizar o rótulo de início e, quando presente, o padrão de prólogo (Módulo 9).
2. **Identificar as entradas.** Quais registradores trazem argumentos, seguindo a convenção do Módulo 10, e o que cada um provavelmente representa.
3. **Identificar os registradores mais usados.** Quais registradores aparecem repetidamente, e que papel cada um parece cumprir naquele trecho (Módulo 3).
4. **Identificar os acessos à memória.** Quais expressões entre colchetes aparecem, e o que cada uma provavelmente representa: uma variável local (`[rbp-N]`), um argumento pela stack (`[rbp+N]`), um elemento de array (`[base+índice*escala]`), ou uma dereferência simples de ponteiro (Módulo 5).
5. **Identificar as operações.** Quais instruções aritméticas ou lógicas aparecem, e o que estão calculando (Módulo 6).
6. **Identificar as comparações.** Onde há `cmp` ou `test`, e o que exatamente está sendo comparado (Módulo 7).
7. **Identificar os desvios.** Quais `jmp` e `j<condição>` existem, e para onde cada um aponta (Módulo 8).
8. **Mapear o fluxo de execução.** Reconstruir, com base nos desvios, o caminho (ou os caminhos possíveis) que a execução realmente percorre.
9. **Determinar o resultado.** O que fica em `EAX`/`RAX` ao final (para funções que retornam um valor), ou qual efeito colateral a função produz (para funções `void`, como escrever em um ponteiro).
10. **Descrever o comportamento em pseudocódigo ou C aproximado.** A etapa final, que resume tudo o que foi identificado nos passos anteriores em uma descrição legível e correta, sem se preocupar em reproduzir a sintaxe exata do código-fonte original.

## 9. Exemplo final integrado, aplicando o método

```c
int conta_pares(int *array, int tamanho) {
    int contador = 0;
    for (int i = 0; i < tamanho; i++) {
        if ((array[i] & 1) == 0) {
            contador++;
        }
    }
    return contador;
}
```

```asm
conta_pares:
    push rbp
    mov rbp, rsp
    sub rsp, 32

    mov qword [rbp-8], rdi
    mov dword [rbp-12], esi
    mov dword [rbp-16], 0
    mov dword [rbp-20], 0
    jmp verificacao

loop_inicio:
    mov rax, qword [rbp-8]
    mov ecx, dword [rbp-20]
    mov edx, dword [rax+rcx*4]
    and edx, 1
    cmp edx, 0
    jne nao_e_par
    inc dword [rbp-16]
nao_e_par:
    inc dword [rbp-20]

verificacao:
    mov eax, dword [rbp-20]
    cmp eax, dword [rbp-12]
    jl loop_inicio

    mov eax, dword [rbp-16]
    mov rsp, rbp
    pop rbp
    ret
```

Aplicando o método da Seção 8:

1. **Função:** `conta_pares`, com prólogo padrão completo (`push rbp` / `mov rbp, rsp` / `sub rsp, 32`).
2. **Entradas:** `RDI` (um ponteiro, guardado em `[rbp-8]`) e `ESI` (um inteiro, guardado em `[rbp-12]`).
3. **Registradores mais usados:** `RAX` (endereço base do array, depois valor de retorno), `ECX` (índice), `EDX` (valor lido do array e resultado do `and`).
4. **Acessos à memória:** `[rbp-8]` (o ponteiro `array`), `[rbp-12]` (`tamanho`), `[rbp-16]` (um contador), `[rbp-20]` (um índice de laço), e `[rax+rcx*4]` (um elemento do array, lido por índice).
5. **Operações:** `and edx, 1` (isola o bit menos significativo), `inc` em dois pontos diferentes (contador e índice).
6. **Comparações:** `cmp edx, 0` (testa se o bit isolado era zero) e `cmp eax, [rbp-12]` (testa a condição do laço).
7. **Desvios:** `jne nao_e_par` (pula o incremento do contador quando o bit não era zero), `jl loop_inicio` (mantém o laço), e o `jmp verificacao` inicial (padrão de `for`, Módulo 8).
8. **Fluxo de execução:** um laço que percorre `array` do índice `0` até `tamanho - 1`; a cada posição, testa o bit menos significativo do valor, e incrementa um contador quando esse bit é zero.
9. **Resultado:** o valor final de `[rbp-16]` (o contador), copiado para `EAX` antes do epílogo.
10. **Descrição em C aproximado:**

```c
int conta_pares(int *array, int tamanho) {
    int contador = 0;
    for (int i = 0; i < tamanho; i++) {
        if ((array[i] & 1) == 0) {
            contador++;
        }
    }
    return contador;
}
```

Neste caso, o C reconstruído coincide quase exatamente com o original, porque o Assembly foi gerado sem otimização, o que preserva a estrutura de forma bastante literal. Em código otimizado, como visto nas Seções 2 e 3, essa reconstrução tende a exigir mais interpretação, mas o método permanece o mesmo.

## 10. Onde este curso termina, e o que vem depois

Este módulo encerra a base proposta por este curso: um modelo mental completo da CPU, registradores, memória, instruções fundamentais, flags, controle de fluxo, stack e funções, unidos pela capacidade de ler Assembly x86-64 e descrever seu comportamento com confiança.

A partir desta base, tópicos mais avançados se tornam acessíveis, entre eles: o formato de executáveis ELF, usado por programas Linux; ferramentas de desmontagem e depuração, que permitem examinar binários reais; otimizações mais agressivas de compiladores, que produzem Assembly ainda mais distante da forma literal do código-fonte; e, mais adiante, engenharia reversa e análise de binários de forma geral, que se apoiam diretamente em tudo o que foi construído aqui. Nenhum desses tópicos faz parte deste curso, mas todos partem exatamente do ponto onde este material termina.

## 11. Erros comuns de leitura

- **Tentar reconstruir o código-fonte exato, palavra por palavra.** Como visto na Seção 7, isso frequentemente não é possível, e não é o objetivo. O alvo é o comportamento, não o texto original.
- **Esquecer que um array, como parâmetro de função, é apenas um ponteiro.** Isso leva a esperar, incorretamente, alguma informação sobre o tamanho do array chegando junto com ele; essa informação, quando existe, precisa ser passada separadamente, como um argumento próprio (Seção 5).
- **Confundir `setcc` com um desvio.** `setcc` nunca altera `RIP`; ela apenas grava `0` ou `1` em um byte, seguindo a mesma lógica de flags de um `jcc` correspondente, mas sem desviar a execução (Seção 3.2).
- **Esquecer o `movzx` depois de um `setcc`, e assumir que o restante do registrador de destino está limpo.** Como reforça a regra do Módulo 3, escrever em `AL` não afeta os demais bits do registrador; algo precisa zerá-los explicitamente, quando um valor de 32 bits limpo é necessário.

## 12. Tabela-resumo: onde cada assunto foi coberto

| Construto ou conceito | Módulo de referência |
|---|---|
| Registradores e suas subdivisões | Módulo 3 |
| Endereçamento e dereferência (`[base+índice*escala]`) | Módulo 5 |
| Instruções aritméticas e lógicas | Módulo 6 |
| `setcc`, `movzx` (introduzidas neste módulo) | Módulo 11 |
| Flags (ZF, SF, CF, OF) | Módulo 7 |
| `if`, `if/else`, `while`, `do-while`, `for`, `switch` | Módulo 8 |
| Stack, `RBP`, prólogo e epílogo | Módulo 9 |
| `call`, `ret`, convenção de argumentos, preservação de registradores | Módulo 10 |
| Arrays como ponteiros, correspondência C ↔ Assembly | Módulo 11 |

## 13. Exercícios

> Os níveis abaixo seguem a mesma escala de dificuldade usada nos exercícios de todos os módulos anteriores, do Nível 1 (interpretar uma instrução isolada) ao Nível 8 (interpretar um programa pequeno desconhecido). Nem todo nível se aplica a todo módulo, por isso a numeração pode saltar, aqui, por exemplo, direto do Nível 1 ao Nível 5.

### Nível 1 — Conceitual

1. Por que não existe uma correspondência de um para um entre C e Assembly?
2. O que `setcc` faz de diferente em relação a um `jcc` correspondente?
3. Por que `movzx eax, al` costuma aparecer logo depois de um `setcc`?
4. Por que um parâmetro declarado como `int array[]` chega a uma função exatamente como um ponteiro?

### Nível 5 — Interpretar uma variação

```asm
dobro_ou_nada:
    cmp edi, 0
    setg al
    movzx eax, al
    imul eax, edi
    imul eax, 2
    ret
```

5. Descreva, em C aproximado, o comportamento desta função. (Dica: pense no que `setg`/`movzx` produzem quando multiplicados por outro valor.)

### Nível 8 — Interpretar um programa pequeno desconhecido

> Este trecho usa uma variação estrutural do padrão de laço visto no Módulo 8: a verificação aparece no topo do laço, com um desvio de saída direto, e o retorno ao início ocorre por meio de um `jmp` incondicional ao final do corpo, em vez do padrão `jmp verificacao` inicial usado nos exemplos anteriores. É uma forma igualmente válida de compilar um `for`, e reconhecê-la reforça exatamente a lição da Seção 7: a mesma lógica pode aparecer em mais de uma forma estrutural.

```asm
maior_valor:
    push rbp
    mov rbp, rsp
    sub rsp, 32

    mov qword [rbp-8], rdi
    mov dword [rbp-12], esi
    mov rax, qword [rbp-8]
    mov eax, dword [rax]
    mov dword [rbp-16], eax     ; maior = array[0]
    mov dword [rbp-20], 1        ; i = 1

verificacao:
    mov eax, dword [rbp-20]
    cmp eax, dword [rbp-12]
    jge fim_loop

    mov rax, qword [rbp-8]
    mov ecx, dword [rbp-20]
    mov edx, dword [rax+rcx*4]
    cmp edx, dword [rbp-16]
    jle nao_atualiza
    mov dword [rbp-16], edx
nao_atualiza:
    inc dword [rbp-20]
    jmp verificacao

fim_loop:
    mov eax, dword [rbp-16]
    mov rsp, rbp
    pop rbp
    ret
```

Aplique o método da Seção 8 e responda:

6. Quais são as entradas desta função?
7. O que a variável em `[rbp-16]` representa ao longo da execução?
8. Por que o laço começa em `i = 1`, e não em `i = 0`?
9. Escreva o código C aproximado que esta função representa.

---

## 14. Respostas

1. Porque o compilador toma decisões (como alocação de registradores, otimizações, e a forma de organizar a stack) que não são determinadas de forma única pelo código-fonte. Diferentes compiladores, versões, ou níveis de otimização podem gerar Assembly diferente a partir do mesmo C, e o inverso também ocorre: o mesmo Assembly, em alguns casos, pode ter vindo de mais de uma forma diferente de código-fonte.
2. `setcc` não desvia a execução: ela apenas grava `0` ou `1` em um operando de 1 byte, com base na mesma combinação de flags que um `jcc` testaria. `jcc` muda `RIP` condicionalmente; `setcc` nunca muda `RIP`.
3. Porque `setcc` escreve apenas em um registrador de 8 bits (como `AL`), e, como visto no Módulo 3, escrever em um registrador de 8 bits não afeta o restante do registrador de 32 ou 64 bits. `movzx` zera explicitamente esses bits restantes, produzindo um valor de 32 bits limpo (0 ou 1) a partir do byte calculado por `setcc`.
4. Porque essa é uma regra da própria linguagem C: quando um array aparece como parâmetro de uma função, ele é tratado como um ponteiro para seu primeiro elemento, e qualquer tamanho declarado entre colchetes nesse contexto é ignorado pelo compilador.
5. 
```c
int dobro_ou_nada(int x) {
    int base = (x > 0) ? 1 : 0;
    return base * x * 2;
}
```
Ou, de forma equivalente e mais direta: `return (x > 0) ? x * 2 : 0;`, já que `base` vale `1` quando `x > 0` (preservando `x * 2`) e `0` caso contrário (zerando o resultado).
6. Um ponteiro em `RDI` (guardado em `[rbp-8]`) e um inteiro em `ESI` (guardado em `[rbp-12]`), o mesmo padrão de um array e seu tamanho, visto na Seção 5.
7. Representa o maior valor encontrado no array até o momento, sendo atualizada sempre que um valor maior é encontrado durante o laço.
8. Porque o valor em `[rbp-16]` já foi inicializado com `array[0]` antes do laço começar (`mov eax, dword [rax]` seguido de `mov dword [rbp-16], eax`, logo após o prólogo). Começar o laço em `i = 1` evita comparar `array[0]` consigo mesmo desnecessariamente.
9. 
```c
int maior_valor(int *array, int tamanho) {
    int maior = array[0];
    for (int i = 1; i < tamanho; i++) {
        if (array[i] > maior) {
            maior = array[i];
        }
    }
    return maior;
}
```

---

## 15. Considerações finais

Chegar até aqui significa ter percorrido, passo a passo, o caminho completo entre um único bit e uma função inteira em execução: dos registradores mais simples até a leitura de chamadas de função com argumentos, stack frames e fluxo de controle completo. É um percurso que exige disciplina e atenção a detalhes que parecem pequenos, um colchete, um sinal, um deslocamento de quatro bytes, além da paciência de repetir o mesmo processo de leitura muitas vezes, até que ele se torne automático.

Quem seguiu os onze módulos até este ponto, resolveu os exercícios propostos e aplicou o método de dez passos da Seção 8 a um programa desconhecido na Seção 9, já não olha mais para um trecho de Assembly como uma sequência arbitrária de símbolos. Passa a reconhecer registradores, memória, flags e desvios pelo que realmente são: a mesma lógica que qualquer programa em C expressa, só que revelada em sua forma mais literal e direta.

Essa é a base necessária para os próximos passos, sejam eles engenharia reversa, análise de binários, otimização de código, ou simplesmente a curiosidade de entender o que realmente acontece por trás de qualquer programa em execução. Parabéns por concluir esta primeira etapa, e bons estudos daqui em diante.

---

*Módulo anterior: [Módulo 10 — Funções](./10-funcoes.md)*
