# Módulo 10 — Funções

> **Arquitetura:** x86-64
> **Sistema operacional de referência:** Linux
> **ABI:** System V AMD64
> **Sintaxe:** Intel

## 1. Introdução

O Módulo 9 deixou duas promessas em aberto: como os argumentos de uma função realmente chegam até ela (mencionamos `EDI` e `ESI` de passagem, sem justificar), e o que exatamente `ret` faz. Este módulo cumpre as duas, e completa o quadro necessário para ler qualquer função Assembly simples de ponta a ponta: como ela é chamada, como recebe seus dados, como devolve um resultado, e como garante que a função que a chamou não seja afetada por essa chamada.

O objetivo aqui não é decorar a ABI inteira, com todos os seus casos especiais, mas sim aprender o suficiente para ler chamadas de função comuns com confiança.

## 2. Ordem no arquivo não é ordem de execução

Antes de ver `call` e `ret` em detalhe, vale resolver uma armadilha comum, e bastante razoável de cair: dentro de uma única função, as instruções realmente executam de cima para baixo, na ordem em que aparecem (a menos que um desvio, como os do Módulo 8, mude isso). Mas isso **não se estende** ao arquivo inteiro quando há várias funções escritas uma após a outra.

### 2.1 O que realmente acontece na fronteira entre duas funções

```asm
funcao_b:
    ; corpo de funcao_b
    ret

funcao_a:
    ; corpo de funcao_a
    call funcao_b
    ; ...
    ret
```

Mesmo `funcao_b` aparecendo **primeiro** no arquivo, ela não executa "primeiro" só por isso. Uma função só começa a executar quando algo desvia explicitamente para o endereço do seu rótulo, seja um `call` (o caso normal), seja um `jmp` (mais raro, para casos específicos). Se nada no programa jamais executar `call funcao_b`, o corpo de `funcao_b` simplesmente nunca roda, independentemente de sua posição no arquivo.

Repare também que `funcao_b` termina em `ret`. Como será detalhado na próxima seção, `ret` devolve a execução para quem chamou, ela não "cai" para dentro do próximo rótulo escrito logo abaixo no arquivo. Então, mesmo que a CPU chegasse a executar `funcao_b` inteira, ao encontrar `ret`, ela retornaria para o ponto de chamada, não continuaria descendo para `funcao_a`.

> **Regra a fixar:** posição no arquivo é apenas uma escolha de organização de quem escreveu (ou do compilador que gerou) o código. Execução é guiada inteiramente por desvios explícitos (`call`, `jmp`, `j<condição>`) e pelo fim natural de cada instrução, nunca pela ordem visual do texto.

### 2.2 Então, o que decide o que executa primeiro?

Todo programa precisa de um **ponto de entrada**: um endereço específico onde a CPU começa a buscar instruções assim que o programa é iniciado (voltando ao ciclo fetch-decode-execute do Módulo 2, é preciso que `RIP` receba algum valor inicial vindo de algum lugar). Em um programa Linux compilado a partir de C, esse ponto de entrada tem um nome convencional, `_start`, mas ele não é escrito diretamente pelo programador: é inserido automaticamente pelo processo de compilação, e sua função é preparar o ambiente de execução e, então, chamar explicitamente a função `main` do programa, através de um `call main` (ou equivalente).

A partir daí, `main` chama outras funções, na ordem que a lógica do programa exigir, através de instruções `call` explícitas, exatamente como qualquer outra função faria. A ordem em que essas funções foram escritas no código-fonte original, ou em que aparecem no Assembly gerado, não tem relação necessária com a ordem em que são efetivamente chamadas durante a execução.

Isso é exatamente o que vamos aplicar ao exemplo integrado da Seção 8: uma das duas funções ali só executa porque a outra a chama explicitamente com `call`, nunca por estar posicionada antes ou depois no arquivo.

## 3. `call` — chamando uma função

### O que faz

`call` faz duas coisas em sequência: empilha o endereço da instrução **seguinte** a ela mesma (o **endereço de retorno**), e depois desvia `RIP` para o início da função chamada, exatamente como um `jmp` faria.

### Sintaxe

```asm
call rotulo_da_funcao
```

### Por que empilhar o endereço de retorno

Pense no problema que isso resolve: quando a função chamada terminar, como ela sabe para onde voltar? A resposta é: o endereço de retorno fica guardado na stack, exatamente no formato que já vimos em `push` (Módulo 6 e Módulo 9), esperando para ser lido quando a função terminar.

### Exemplo com endereços concretos

Seguindo o mesmo estilo do Módulo 9, vamos acompanhar números reais. Suponha que `RSP = 0x4000` no momento em que a instrução `call` abaixo é executada, e que essa instrução `call` esteja no endereço `0x1000`, ocupando 5 bytes (um tamanho comum para essa instrução), portanto a instrução seguinte está em `0x1005`.

```asm
0x1000:  call soma        ; chama a função 'soma'
0x1005:  mov  ecx, eax    ; instrução seguinte, o "endereço de retorno"
```

Ao executar `call soma`:

1. O endereço `0x1005` (o endereço de retorno) é empilhado: `RSP` passa a valer `0x3FF8`, e o valor `0x1005` é escrito nesse endereço.
2. `RIP` passa a apontar para o início da função `soma`, seja lá onde essa função estiver localizada na memória, sua posição no arquivo-fonte não influencia esse endereço, que é resolvido pelo assembler/linker.

```
0x4000  ┌──────────────────────────  ← RSP estava aqui, antes do call
        │   (memória de outra função)  │
0x3FF8  ├──────────────────────────  ← RSP aponta aqui, depois do call
        │   0x1005  (endereço de       │
        │   retorno, empilhado)        │
        └──────────────────────────
```

> **`call` é, na prática, um `push` do endereço de retorno seguido de um `jmp`.** Fixar essa equivalência ajuda bastante a entender o que vem a seguir: `ret`.

### Um segundo exemplo: chamando uma função definida antes, no arquivo

Para reforçar a Seção 2, considere que `soma` esteja definida **antes** de quem a chama:

```asm
soma:
    ; corpo de soma (será visto na íntegra na Seção 8)
    ret

principal:
    ...
    call soma        ; funciona normalmente, mesmo 'soma' estando ANTES no arquivo
    ...
```

Não há nenhuma diferença de comportamento entre chamar uma função definida antes ou depois, no arquivo. O assembler resolve o rótulo `soma` para um endereço fixo de memória durante a montagem do programa, e `call soma` sempre desvia para esse endereço, independentemente de onde ele esteja em relação à instrução `call` no texto-fonte. Isso é conceitualmente parecido com o que já vimos no Módulo 8 sobre rótulos de desvio (`jmp fim`, por exemplo): o nome é só um apelido para um endereço, resolvido pelo assembler.

## 4. `ret` — retornando de uma função

### O que faz

`ret` desempilha um valor (exatamente como `pop` faria) e o coloca em `RIP`. Como o valor que está no topo da stack, nesse momento, é o endereço de retorno empilhado pelo `call` correspondente, o efeito é a execução "voltar" para logo depois de onde a função foi chamada.

### Sintaxe

```asm
ret
```

### Continuando o exemplo numérico

Supondo que, ao final da função `soma`, a stack esteja exatamente como no diagrama da Seção 3 (o epílogo do Módulo 9 já devolveu `RSP` e `RBP` ao estado correto, sobrando apenas o endereço de retorno no topo):

```
0x3FF8  ┌────────────────────────── ← RSP aponta aqui, antes do ret
        │   0x1005  (endereço de       │
        │   retorno)                   │
        └──────────────────────────
```

Ao executar `ret`:

1. O valor no topo (`0x1005`) é lido e colocado em `RIP`.
2. `RSP` volta a `0x4000` (o valor de antes do `call`, na Seção 3).
3. A execução continua na instrução `mov ecx, eax`, em `0x1005`, exatamente onde havia parado.

> **`ret` é, na prática, um `pop` para `RIP`.** Essa simetria com `call` (que é um `push` seguido de `jmp`) é o que fecha o ciclo completo de uma chamada de função, e é também o motivo pelo qual `ret` nunca "cai" para o próximo rótulo do arquivo: ele não segue a ordem textual, ele lê exatamente o endereço que foi empilhado pelo `call` correspondente, esteja esse endereço onde estiver.

## 5. Argumentos: a convenção de registradores

Chegou a hora de formalizar o que os Módulos 3 e 9 já adiantaram. A System V AMD64 ABI define que os primeiros seis argumentos inteiros (ou ponteiros) de uma função são passados por registradores específicos, nesta ordem fixa:

| Posição do argumento | Registrador (64 bits) | Registrador (32 bits, para `int`) |
|---|---|---|
| 1º | `RDI` | `EDI` |
| 2º | `RSI` | `ESI` |
| 3º | `RDX` | `EDX` |
| 4º | `RCX` | `ECX` |
| 5º | `R8` | `R8D` |
| 6º | `R9` | `R9D` |

Se uma função tiver mais de seis argumentos inteiros, os excedentes são passados pela stack, no formato de deslocamento positivo a partir de `RBP` que já vimos no Módulo 9 (Seção 8). Isso é relativamente raro em código C comum, então não vamos detalhar essa mecânica agora.

> **Fora do escopo deste curso, por ora:** argumentos de ponto flutuante (`float`, `double`) usam um conjunto de registradores completamente diferente (`XMM0` a `XMM7`), que não faz parte do que este curso cobre até aqui. Ao encontrar uma função lidando com esses tipos, a lógica de "os primeiros argumentos vão em registradores específicos" continua valendo, só que com registradores diferentes.

### Exemplo 1: `soma(a, b)` sendo chamada

```c
int resultado = soma(3, 4);
```

```asm
mov edi, 3            ; primeiro argumento (a)
mov esi, 4            ; segundo argumento (b)
call soma             ; empilha endereço de retorno, desvia para 'soma'
mov [resultado], eax  ; usa o valor de retorno, já em EAX
```

### Exemplo 2: uma chamada com quatro argumentos

```c
int total = calcular(10, 20, 30, 40);
```

```asm
mov edi, 10       ; 1º argumento
mov esi, 20       ; 2º argumento
mov edx, 30       ; 3º argumento
mov ecx, 40       ; 4º argumento
call calcular
mov [total], eax
```

Repare que a ordem dos registradores usados (`EDI, ESI, EDX, ECX`) segue exatamente a ordem da tabela, mesmo esta função tendo mais argumentos que o primeiro exemplo. Reconhecer essa sequência fixa é o que permite, ao ler qualquer chamada, saber instantaneamente qual registrador corresponde a qual posição de argumento, sem precisar consultar a função em si.

## 6. Valor de retorno: `EAX`/`RAX`

Valores inteiros de retorno são colocados em `RAX`. Para valores de até 32 bits (o caso mais comum, correspondendo a `int` em C), utiliza-se a parte baixa, `EAX`. Para valores menores, utilizam-se `AX` ou `AL`, conforme o tamanho, exatamente as subdivisões de registrador já vistas no Módulo 3. Quem chamou a função lê esse valor assim que `ret` devolve o controle.

## 7. Preservação de registradores: quem promete o quê

Esta é a última peça necessária, e talvez a mais sutil: quando uma função é chamada, ela pode usar registradores livremente para seus próprios cálculos internos. Mas isso levanta uma pergunta: e se a função que chamou já estava usando aquele mesmo registrador para guardar algo importante? A convenção resolve isso dividindo os registradores em dois grupos.

### 7.1 *Caller-saved* (preservados por quem chama)

Registradores que uma função chamada **pode livremente sobrescrever**, sem aviso prévio. Se quem chama precisa manter o valor desses registradores depois da chamada, é responsabilidade de quem chama salvá-los antes (por exemplo, empilhando-os com `push`, e restaurando depois com `pop`).

```
RAX, RCX, RDX, RSI, RDI, R8, R9, R10, R11
```

> Repare que os registradores de argumento (`RDI`, `RSI`, `RDX`, `RCX`, `R8`, `R9`) e o de retorno (`RAX`) estão justamente neste grupo. Faz sentido: eles existem exatamente para carregar dados de uma função para outra, então não haveria motivo para a função chamada precisar preservá-los.

**Exemplo do problema que isso pode causar, se ignorado:**

```asm
mov ecx, 99            ; ECX guarda um valor importante para o código atual
call alguma_funcao     ; alguma_funcao pode usar ECX livremente, sem avisar
add eax, ecx           ; PERIGO: ECX pode não valer mais 99 aqui
```

Se `alguma_funcao` usar `ECX` internamente (o que ela tem todo o direito de fazer, por ser *caller-saved*), o valor `99` pode ter sido sobrescrito. Para evitar isso, quem escreve (ou o compilador que gera) o código precisa salvar `ECX` antes da chamada, caso ainda precise dele depois:

```asm
mov ecx, 99
push rcx               ; salva ECX/RCX antes da chamada
call alguma_funcao
pop rcx                ; restaura o valor original
add eax, ecx
```

### 7.2 *Callee-saved* (preservados por quem é chamado)

Registradores que, se a função chamada quiser usar internamente, ela é **obrigada a preservar**: salvar o valor original (tipicamente empilhando-o no início da função) e restaurá-lo antes de retornar (desempilhando-o no final).

```
RBX, RBP, RSP, R12, R13, R14, R15
```

> Note que `RBP` e `RSP` já se encaixam naturalmente aqui: o próprio prólogo/epílogo do Módulo 9 (`push rbp` no início, `pop rbp` no final) é exatamente esse padrão de preservação sendo aplicado ao `RBP` da função que chamou.

### 7.3 Por que isso importa para leitura

Ao ler uma função, encontrar `push rbx` logo no início (além do `push rbp` já esperado do prólogo padrão) é um sinal de que a função pretende usar `RBX` internamente, e está seguindo a convenção de preservá-lo. Espera-se encontrar o `pop rbx` correspondente antes do epílogo:

```asm
minha_funcao:
    push rbp
    mov rbp, rsp
    push rbx         ; RBX será usado aqui dentro, então seu valor original é preservado

    ; ... uso de RBX no corpo da função ...

    pop rbx          ; RBX restaurado ao valor que tinha antes da função
    mov rsp, rbp
    pop rbp
    ret
```

## 8. Exemplo integrado completo

Vamos juntar tudo, desde a chamada até o retorno. Seguindo a Seção 2, vamos deixar explícito, em comentário, que algo externo a este trecho (o ponto de entrada do programa, eventualmente chamando `main`) é quem inicia a execução chamando `principal`. A partir daí, o próprio `principal` é quem decide chamar `soma`, através de um `call` explícito, não por uma questão de posição no arquivo.

```c
int soma(int a, int b) {
    int resultado = a + b;
    return resultado;
}

int principal() {
    int x = soma(3, 4);
    return x;
}
```

```asm
; (assumindo que algo externo, como o ponto de entrada do programa, já
;  chamou 'principal' neste momento; vamos acompanhar a execução a partir daqui)

principal:
    push rbp
    mov rbp, rsp
    sub rsp, 16
    mov edi, 3
    mov esi, 4
    call soma                     ; desvia explicitamente para 'soma', definida abaixo
    mov dword [rbp-4], eax        ; x recebe o valor de retorno
    mov eax, dword [rbp-4]
    mov rsp, rbp
    pop rbp
    ret

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

Note que, aqui, `soma` foi colocada **depois** de `principal` no arquivo, o que combina melhor com a ordem em que a execução realmente acontece, mas isso é só uma escolha de organização para facilitar a leitura. Como vimos na Seção 3 (segundo exemplo), a ordem no arquivo poderia perfeitamente ser invertida sem alterar em nada o comportamento, desde que o `call soma` continue apontando corretamente para o rótulo `soma`, onde quer que ele esteja.

### 8.1 Seguindo o fluxo completo, passo a passo

1. Algo externo (fora deste trecho) chama `principal`, o que empilha um endereço de retorno próprio, não mostrado aqui.
2. `principal` monta seu próprio stack frame (prólogo padrão).
3. `mov edi, 3` / `mov esi, 4`: os argumentos de `soma` são colocados nos registradores corretos, seguindo a convenção da Seção 5.
4. `call soma`: o endereço de retorno (a instrução `mov dword [rbp-4], eax`) é empilhado, e a execução salta para `soma`, mesmo ela estando escrita depois, no arquivo.
5. `soma` monta seu **próprio** stack frame, empilhado sobre o de `principal` (o `RBP` de `principal` foi salvo pelo `push rbp` de `soma`, exatamente como visto no Módulo 9).
6. `soma` copia `EDI` e `ESI` para variáveis locais, calcula a soma, e coloca o resultado em `EAX`.
7. `soma` desfaz seu stack frame (epílogo) e executa `ret`, que lê o endereço de retorno empilhado no passo 4 e devolve a execução para `principal`, exatamente para a instrução seguinte ao `call`, não para o que estivesse escrito logo abaixo de `soma` no arquivo.
8. De volta em `principal`, o valor de `EAX` (o resultado de `soma`) é copiado para a variável local `x`.
9. `principal` prepara seu próprio valor de retorno e executa seu próprio epílogo e `ret`, devolvendo o controle para quem o chamou no passo 1.

### 8.2 Uma observação sobre este exemplo: uma redundância que vale a pena notar

Vale reexaminar o corpo de `soma` com mais atenção, porque ele expõe algo útil sobre como ler Assembly. Observe a sequência completa novamente:

```asm
mov dword [rbp-4], edi
mov dword [rbp-8], esi
mov eax, dword [rbp-4]
add eax, dword [rbp-8]
mov dword [rbp-12], eax
mov eax, dword [rbp-12]
```

Repare que `EAX` não tem um papel fixo aqui: primeiro ele recebe o valor de `a` (vindo de `[rbp-4]`), depois participa da soma como acumulador (`add eax, [rbp-8]`), e por fim se torna o portador do valor de retorno da função. O mesmo registrador é reaproveitado para três propósitos diferentes ao longo de poucas linhas, o que reforça uma ideia importante: registradores não têm um significado fixo, apenas um papel momentâneo, definido pela instrução que os usa naquele instante (Módulo 3).

Agora, olhando as duas últimas linhas com atenção: depois de `add eax, dword [rbp-8]`, `EAX` já contém o resultado final da soma. As duas linhas seguintes escrevem esse valor em `[rbp-12]` e, na sequência imediata, leem esse mesmo endereço de volta para `EAX`. O valor sai de `EAX`, passa pela memória, e volta para `EAX`, sem nenhuma mudança no meio do caminho. Nada, numericamente, seria diferente se essas duas linhas fossem simplesmente removidas.

Essa sequência é exatamente o tipo de passo que um compilador com otimizações ativadas tende a eliminar. O exemplo deste módulo foi mantido nesse formato mais literal de propósito, porque corresponde a como uma variável local declarada explicitamente em C (`int resultado = a + b;`) tende a aparecer em código gerado sem otimização: a variável `resultado` existe de fato como uma posição na stack, e seu valor é gravado ali antes de ser devolvido, ainda que isso signifique um vaivém aparentemente desnecessário entre registrador e memória.

Vale usar este exemplo para praticar uma pergunta que ajuda bastante ao ler qualquer trecho de Assembly: **cada instrução aqui está fazendo algo necessário, ou existe uma sequência que só existe por causa de como o código foi gerado, sem otimização?** Saber reconhecer esse tipo de redundância, sem se confundir com ela, é parte do processo de leitura: o comportamento lógico da função (somar `a` e `b`, e devolver o resultado) não muda, independentemente de quantos passos intermediários o código usa para chegar até ele. Esse mesmo tipo de raciocínio, separar o que é essencial do que é apenas um efeito colateral da forma como o código foi gerado, torna-se cada vez mais valioso à medida que o código analisado cresce em tamanho ou passa a vir de um compilador otimizado.

## 9. Prólogo e epílogo não são obrigatórios

Vale um esclarecimento importante antes de prosseguir. O termo *stack frame*, como usado no Módulo 9, se refere de forma geral a qualquer porção da stack que pertence a uma chamada de função específica. Essa porção não deixa de existir só porque uma função não usa o padrão de prólogo e epílogo ensinado até aqui. O que é opcional não é o uso da stack em si, mas sim **a forma específica de organizá-la**: montar um frame fixo baseado em `RBP`, com `push rbp` / `mov rbp, rsp` no início e o inverso no final.

### 9.1 O frame baseado em `RBP` é uma escolha, não uma exigência

O padrão `push rbp` / `mov rbp, rsp` / `sub rsp, N`, seguido do inverso no epílogo, existe para dar à função uma referência fixa e estável (`RBP`) durante toda a sua execução, como visto no Módulo 9. Isso é extremamente útil quando a função usa `push`/`pop` internamente, ou chama outras funções, situações em que `RSP` fica se movendo e um ponto fixo facilita tanto a geração quanto a leitura do código.

Mas nada na arquitetura x86-64, nem na ABI, exige que toda função monte esse frame especificamente dessa forma. Uma função pode perfeitamente reservar espaço na stack, ou até não precisar de espaço algum, sem nunca estabelecer um `RBP` próprio, referenciando tudo diretamente através de `RSP`.

### 9.2 Duas formas comuns de dispensar o frame baseado em `RBP`

**Reservando espaço via `RSP`, sem `RBP` fixo:**

Uma função pode reservar espaço para variáveis locais apenas com `sub rsp, N` no início e `add rsp, N` no final, referenciando essas variáveis como deslocamentos diretos de `RSP` (`[rsp+4]`, por exemplo), sem nunca copiar esse valor para `RBP`:

```asm
funcao_sem_rbp:
    sub rsp, 16
    mov dword [rsp], edi      ; variável local, referenciada via RSP diretamente
    ; ... corpo da função ...
    add rsp, 16
    ret
```

Aqui, a stack **continua sendo usada**: existe espaço reservado especificamente para esta chamada, então um stack frame, no sentido amplo, existe. O que não existe é o prólogo/epílogo baseado em `RBP` ensinado no Módulo 9. Essa técnica costuma ser chamada de *frame pointer omission*, e é comum em código otimizado, já que libera `RBP` para ser usado como um registrador de uso geral comum, em vez de ficar reservado apenas como referência fixa.

**Não reservando espaço algum, por não precisar:**

Quando uma função não tem variáveis locais nem chama outras funções, ela pode dispensar até mesmo o `sub rsp`, porque simplesmente não há nada para reservar:

```c
int quadrado(int x) {
    return x * x;
}
```

```asm
quadrado:
    mov eax, edi
    imul eax, eax
    ret
```

Aqui, o único uso da stack durante toda a chamada é o endereço de retorno empilhado pelo próprio `call`, que pertence ao mecanismo de chamada em si, não a um espaço reservado por esta função. Este é o caso mais comum entre funções-folha (*leaf functions*, funções que não chamam nenhuma outra), especialmente sob otimização.

> Nos dois casos, o conceito de stack frame, no sentido de "espaço associado a esta chamada", continua válido. O que muda é apenas a forma de organizá-lo: com um `RBP` fixo servindo de referência (o padrão ensinado neste curso, mais fácil de reconhecer), com deslocamentos diretos de `RSP` sem `RBP` fixo, ou, no caso mais simples, sem nenhuma necessidade de espaço reservado além do que o próprio `call` já usa.

### 9.3 O que isso significa para a leitura

Ao encontrar uma função que não começa com `push rbp` / `mov rbp, rsp`, isso não é um sinal de código incompleto, corrompido, ou de que algo foi omitido por engano. Significa que o compilador optou por uma das formas alternativas vistas na Seção 9.2: referenciar dados diretamente via `RSP`, ou simplesmente não precisar de espaço reservado. Ao ler esse tipo de função, o mesmo raciocínio de sempre continua se aplicando, apenas trocando `[rbp+N]`/`[rbp-N]` por `[rsp+N]` quando for o caso, e lembrando que a ausência de espaço reservado significa que a função opera inteiramente com o que já recebeu em registradores.

O padrão com prólogo e epílogo completos, usado como base de ensino neste curso, continua sendo o mais importante de reconhecer primeiro, por ser o mais comum em código não otimizado e por deixar a estrutura da função mais explícita. Mas vale manter em mente que ele representa uma escolha de organização entre outras possíveis, não uma regra obrigatória da arquitetura, e muito menos uma condição para que a stack esteja, ou não, sendo usada.

## 10. Processo de leitura para chamadas de função

1. **Não presumir a ordem de execução pela posição no arquivo.** Localizar primeiro qual função é efetivamente o ponto de partida do trecho analisado (Seção 2).
2. **Localizar `call rotulo`**: identifica o nome da função sendo chamada, onde quer que ela esteja definida no arquivo.
3. **Olhar as instruções imediatamente antes do `call`**: que registradores (`EDI`, `ESI`, `EDX`, `ECX`, `R8D`, `R9D`) estão sendo preenchidos, e em que ordem? Isso revela os argumentos, na ordem correta (Seção 5).
4. **Olhar a instrução imediatamente depois do `call`**: se ela usa `EAX`/`RAX`, é provável que esteja consumindo o valor de retorno da função chamada.
5. **Dentro da função chamada**: aplicar o processo de leitura do Módulo 9 (Seção 10) para identificar prólogo, corpo e epílogo.
6. **Verificar se há `push`/`pop` de registradores *callee-saved* (`RBX`, `R12`-`R15`) além do par `RBP` do prólogo padrão**: isso indica que a função usa esses registradores internamente, e está seguindo a convenção de preservá-los.

## 11. Erros comuns de leitura

- **Assumir que a ordem no arquivo determina a ordem de execução.** Como visto na Seção 2, uma função só executa quando algo desvia explicitamente para ela; sua posição no texto-fonte é irrelevante para isso.
- **Esperar que `ret` "caia" para o próximo rótulo do arquivo.** `ret` sempre retorna para o endereço empilhado pelo `call` correspondente, nunca simplesmente continua para a próxima instrução escrita abaixo.
- **Esquecer que `call` também empilha algo.** Ao contar quantos `push`s existem antes de um `pop` ou `ret`, é fácil esquecer que o próprio `call` já colocou o endereço de retorno na stack, antes mesmo do prólogo da função começar.
- **Assumir que qualquer registrador pode ser usado livremente dentro de uma função.** Registradores *callee-saved* (Seção 7.2) exigem que a função os preserve; ignorar isso ao ler código pode levar a interpretar mal por que certos `push`/`pop` aparecem sem relação aparente com o prólogo/epílogo padrão.
- **Confundir a ordem dos argumentos.** A ordem `RDI, RSI, RDX, RCX, R8, R9` é fixa; inverter mentalmente essa ordem ao ler uma chamada leva a atribuir os valores errados a cada parâmetro.
- **Esperar que argumentos de ponto flutuante sigam a mesma convenção de registradores inteiros.** Como mencionado na Seção 5, eles usam um conjunto próprio (`XMM0` em diante), fora do escopo deste curso por ora.
- **Esperar que toda função monte um frame baseado em `RBP`.** Como visto na Seção 9, uma função pode usar a stack diretamente via `RSP`, sem nunca estabelecer um `RBP` fixo, ou até não precisar de espaço reservado algum. Nenhum desses casos representa um erro ou uma omissão no código: apenas uma forma diferente de organizar o mesmo espaço de stack.

## 12. Tabela-resumo

| Elemento | Papel |
|---|---|
| Ordem no arquivo | Não determina ordem de execução; apenas desvios explícitos (`call`, `jmp`) fazem isso |
| Ponto de entrada (`_start` → `main`) | Onde a execução do programa realmente começa |
| `call rotulo` | Empilha o endereço de retorno, desvia para `rotulo` (como um `push` + `jmp`) |
| `ret` | Desempilha o endereço de retorno para `RIP` (como um `pop` para `RIP`) |
| `RDI, RSI, RDX, RCX, R8, R9` | Registradores dos 6 primeiros argumentos inteiros, nesta ordem |
| `EAX`/`RAX` | Valor de retorno |
| *Caller-saved* | `RAX, RCX, RDX, RSI, RDI, R8-R11`; podem ser sobrescritos pela função chamada sem aviso |
| *Callee-saved* | `RBX, RBP, RSP, R12-R15`; devem ser preservados pela função que os usa internamente |
| Frame baseado em `RBP` | Uma entre várias formas de organizar o uso da stack; opcional, mesmo quando a stack continua sendo usada (Seção 9) |

## 13. Exercícios

### Nível 1 — Conceitual

1. Por que a posição de uma função no arquivo não garante nada sobre quando (ou se) ela será executada?
2. Por que `call` precisa empilhar um endereço antes de desviar, enquanto `jmp` não precisa?
3. Qual é a diferença entre um registrador *caller-saved* e um *callee-saved*?
4. Se uma função usa `R12` internamente, o que se espera encontrar no prólogo e no epílogo dela?
5. Por que uma função sem variáveis locais e sem chamadas internas pode dispensar prólogo e epílogo completos?

### Nível 5 — Interpretar uma chamada completa

```asm
mov edi, 10
mov esi, 20
mov edx, 30
call soma3
mov dword [rbp-4], eax
```

6. Quantos argumentos estão sendo passados para `soma3`, e quais são seus valores?
7. Supondo que `soma3` simplesmente some os três valores e retorne o resultado, o que estará em `[rbp-4]` depois deste trecho?

### Nível 6 — Reconstruir uma função com preservação de registrador

```asm
processa:
    push rbp
    mov rbp, rsp
    push rbx

    mov ebx, edi        ; RBX usado como "área de trabalho" nesta função
    add ebx, 100
    mov eax, ebx

    pop rbx
    mov rsp, rbp
    pop rbp
    ret
```

8. Por que existe um `push rbx` logo após `push rbp`, e um `pop rbx` logo antes do epílogo? O que aconteceria de errado se essas duas linhas fossem removidas?
9. Escreva o código C aproximado que esta função representa.

### Nível 7 — Ordem de execução vs. ordem no arquivo

```asm
auxiliar:
    push rbp
    mov rbp, rsp
    mov eax, edi
    imul eax, 2
    pop rbp
    ret

principal:
    push rbp
    mov rbp, rsp
    mov edi, 5
    call auxiliar
    mov dword [rbp-4], eax
    mov rsp, rbp
    pop rbp
    ret
```

10. Mesmo `auxiliar` estando definida antes de `principal` no arquivo, qual das duas funções executa primeiro quando o programa roda, supondo que algo externo chame `principal`? Justifique usando o que foi visto na Seção 2.

### Nível 8 — Identificar redundância

```asm
dobro_mais_um:
    push rbp
    mov rbp, rsp
    sub rsp, 8
    mov dword [rbp-4], edi
    mov eax, dword [rbp-4]
    add eax, eax
    add eax, 1
    mov dword [rbp-8], eax
    mov eax, dword [rbp-8]
    mov rsp, rbp
    pop rbp
    ret
```

11. Quais duas instruções deste trecho formam uma sequência redundante, no mesmo padrão discutido na Seção 8.2? Explique por quê.
12. Reescreva o corpo da função (entre o prólogo e o epílogo), removendo essa redundância, sem alterar o resultado final.

---

## 14. Respostas

1. Porque uma função só começa a executar quando algo desvia explicitamente para o endereço do seu rótulo (por meio de `call` ou `jmp`). Sem esse desvio explícito, o corpo da função nunca é alcançado, independentemente de estar escrito antes ou depois de outras funções no arquivo. A CPU segue endereços e desvios, não a ordem visual do texto-fonte.
2. Porque, ao final da função chamada, é preciso saber para onde voltar. `jmp` é usado para desvios permanentes dentro do mesmo fluxo (como em `if`/`while`), onde não há necessidade de "retornar"; `call` é usado especificamente para chamar uma função que, ao terminar, deve devolver o controle para logo depois de onde foi chamada, e o endereço de retorno precisa estar guardado em algum lugar para isso ser possível.
3. Um registrador *caller-saved* pode ser livremente sobrescrito pela função chamada, sem que ela precise avisar ou restaurá-lo, então é responsabilidade de quem chama salvar seu valor antes, caso precise dele depois. Um registrador *callee-saved* deve ser preservado pela própria função que o utiliza internamente: se ela quiser usá-lo, deve salvar o valor original (normalmente com `push`) e restaurá-lo (com `pop`) antes de retornar.
4. Espera-se encontrar `push r12` logo no início (após o `push rbp` do prólogo padrão), guardando o valor original de `R12` antes de a função sobrescrevê-lo, e um `pop r12` correspondente antes do epílogo, restaurando esse valor para a função que a chamou.
5. Porque, sem variáveis locais para armazenar e sem chamadas internas que desloquem `RSP`, não há necessidade de reservar espaço na stack nem de um `RBP` fixo como referência: o argumento já chega em um registrador, o cálculo pode ser feito inteiramente em registradores, e o único uso da stack durante toda a chamada é o endereço de retorno empilhado pelo próprio `call` (Seção 9).
6. Três argumentos: `10`, `20` e `30`, passados em `EDI`, `ESI` e `EDX`, respectivamente (Seção 5).
7. `[rbp-4]` receberá `60` (a soma de `10 + 20 + 30`), pois esse é o valor de retorno de `soma3`, colocado em `EAX` e depois copiado para `[rbp-4]`.
8. `push rbx` salva o valor original de `RBX` (pertencente à função que chamou `processa`) antes de a função sobrescrevê-lo para seu próprio uso interno. `pop rbx` restaura esse valor original antes de retornar. Se essas duas linhas fossem removidas, `RBX` sairia desta função com um valor diferente do que tinha antes de ela ser chamada, o que violaria a convenção *callee-saved* e poderia corromper silenciosamente o funcionamento de quem chamou `processa`, caso essa função dependesse do valor de `RBX` depois da chamada.
9. 
```c
int processa(int valor) {
    int trabalho = valor;
    trabalho = trabalho + 100;
    return trabalho;
}
```
10. `principal` executa primeiro, porque é ela quem foi chamada por algo externo ao trecho mostrado. `auxiliar` só passa a executar no momento em que `principal` alcança a instrução `call auxiliar`, independentemente de `auxiliar` estar definida antes no arquivo. A posição no arquivo é apenas uma escolha de organização; a ordem real de execução é definida inteiramente pelos desvios explícitos (`call`, neste caso), como visto na Seção 2.
11. As duas últimas instruções antes do epílogo, `mov dword [rbp-8], eax` e `mov eax, dword [rbp-8]`, formam a redundância. Depois de `add eax, eax` seguido de `add eax, 1`, `EAX` já contém o valor final que a função deve retornar; escrevê-lo em `[rbp-8]` e lê-lo de volta para `EAX` não muda nada, apenas passa o mesmo valor pela memória sem necessidade.
12. 
```asm
mov dword [rbp-4], edi
mov eax, dword [rbp-4]
add eax, eax
add eax, 1
```
(As instruções envolvendo `[rbp-8]` podem ser removidas, já que `EAX` já contém o valor final de retorno logo após `add eax, 1`.)

---

*Módulo anterior: [Módulo 9 — Stack](./09-stack.md)*
*Próximo módulo: [Módulo 11 — C → Assembly](./11-c-para-assembly.md)*
