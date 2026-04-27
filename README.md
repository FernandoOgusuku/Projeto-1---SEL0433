# Projeto-1---SEL0433

<div align="center">
  <img height="80" alt="Image" src="https://github.com/user-attachments/assets/8f68842b-2353-4898-a1ef-581c7dfbcb02" />
  <img height="80" alt="Image" src="https://github.com/user-attachments/assets/93cc5a3b-ca74-4bfc-9819-ecaee6ea8ae8" />
  <br><br>
  <b>Departamento de Engenharia Elétrica e de Computação</b><br>
  <b>EESC USP | SEL0433 - Aplicação de Microprocessadores</b><br>
  <b>Prof.: Pedro Oliveira</b>
  <br><br>
  <h1>Projeto 1: "Sistema de Dosagem Rotativa"</h1>
  <i>Implementação em Linguagem Assembly (Microcontrolador MCS-51)</i>
  <br><br>
  <img height="80" alt="Image" src="https://github.com/user-attachments/assets/1bf8fe88-9069-4698-b9a1-080933eb2702" />

</div>

<br>

<div align="right">
  <b>Turma:</b> 2ª feira 14h<br><br>
  <b>Alunos:</b><br>
  João Vitor Miranda Sousa – 14802702<br>
  Fernando Shoji Ogusuku – 15636682
</div>

---

## 1 Objetivo e Contextualização
Este projeto, estruturado na metodologia PBL (*Problem-Based Learning*), visa o desenvolvimento prático do firmware em *baixo nível* para um microcontrolador da família 8051. O cenário simulado é uma linha de produção de parafusos, onde o sistema deve controlar um dosador rotativo acionado por motor DC. 

A aprendizagem foca na manipulação de registradores (GPR e SFR), controle de fluxo, uso da pilha (*Stack*), sub-rotinas, I/O digital e, fundamentalmente, na aplicação de interrupções de hardware para contagem precisa de eventos externos (pulsos do motor).

---

## 2 Requisitos do Sistema
De acordo com o roteiro do projeto, o sistema deve cumprir integralmente os seguintes requisitos:
1. **Contagem de Voltas:** O motor deve realizar a contagem de suas voltas utilizando uma interrupção interna (Timer 1).
2. **Display:** Exibição da contagem (0 a 9) em um display de 7 segmentos.
3. **Ciclo Automático:** Ao atingir exatamente 10 voltas, o sistema reinicia a contagem automaticamente para 0.
4. **Inversão de Rotação:** O botão `SW0` comanda o sentido do motor. Ao ser alterado, o motor deve inverter imediatamente e zerar a contagem atual.
5. **Indicação Visual de Sentido:** O Ponto Decimal (DP) do display deve refletir o sentido de rotação atual (Aceso para um lado, Apagado para o outro).

---

## 3 Lógica de Implementação e Arquitetura
Para assegurar a robustez do sistema, estruturamos o código separando responsabilidades. O **Loop Principal** gerencia apenas a interface (leitura da chave e atualização do display), enquanto as tarefas de tempo crítico e verificação de limite ficaram a cargo do **Hardware de Interrupção**.

### 3.1 Mapeamento de Memória e Prevenção de Colisão
Para garantir que o fluxo intenso de interrupções não sobrescreva os nossos dados de contagem, o mapa de memória e os registos de controlo foram definidos conforme a tabela abaixo:

| Endereço / Reg. | Nome Lógico | Justificativa da Aplicação |
| :--- | :--- | :--- |
| `2FH` | `SP` (Stack Pointer) | Inicialização do topo da pilha. Mover o `SP` protege os registos dos Bancos 0 a 3. |
| `40H` | Variável de Processo | Armazena o número de voltas atuais (0 a 9). Escolhido por estar distante do *Stack Pointer*. |
| `F0` | Flag de Usuário | Bit do registo `PSW` utilizado como variável de controlo de estado direcional (1 = horário, 0 = anti-horário). |

### 3.2 Configuração do Temporizador (TMOD = 60H)
Configuramos o **Timer 1 no Modo 2 (C/T = 1, M1=1, M0=0)**. Neste modo, ele funciona como um **Contador de Eventos Externos com Auto-Reload de 8 bits**.

**Lógica Matemática:** Inicializamos o contador (`TL1` e `TH1`) com o valor máximo `FFH` (255 em decimal). Ao menor movimento do motor, é gerado um único pulso no pino P3.5. Como o contador já está no limite, esse pulso causa um *overflow* instantâneo, ativando a flag `TF1`, que por sua vez dispara o Vetor de Interrupção `001BH`. O hardware então recarrega automaticamente o valor `FFH`, deixando o sistema pronto para o próximo pulso.

---

## 4 Análise do Código Fonte (Passo a Passo)
Abaixo, detalhamos os principais blocos lógicos construídos na Entrega Final.

### 4.1 Vetores de Interrupção e Proteção de Memória
O código desvia da rotina inicial e aloca o espaço exato na ROM para o vetor do Timer 1. Logo após, inicializamos o *Stack Pointer* e nossa variável de processo em uma área segura.

```assembly
ORG 0000H            ; Vetor de Reset
SJMP MAIN            ; Salta para o programa principal
ORG 001BH            ; Vetor de Interrupcao do Timer 1
SJMP INT_TIMER1      ; Salta para a rotina de servico
MAIN:
    MOV SP, #2FH     ; Protege a area de sub-rotinas
    MOV 40H, #00H    ; Zera o contador na memoria RAM segura
```

### 4.2 Rotina de Serviço de Interrupção (ISR)

Quando o motor gira, o hardware executa esta rotina. Usamos `PUSH` e `POP` para preservar o contexto. A regra de limite é aplicada: se a RAM em `40H` chegar a 10, chamamos a rotina de reset.

```assembly
INT_TIMER1:
    PUSH ACC             ; Salva o Acumulador na pilha
    PUSH PSW             ; Salva a Program Status Word
    INC 40H              ; Incrementa a variavel de processo
    MOV A, 40H
    CJNE A, #10, FIM_INT ; Se nao for 10, sai da interrupcao.
    ACALL RESET_TIMER    ; Se for 10, zera o ciclo
FIM_INT:
    POP PSW              
    POP ACC              
    RETI                 ; Retorna da Interrupcao
```

### 4.3 Lógica do Ponto Decimal (Bit 7)

O bit de estado `F0` é injetado diretamente no bit 7 (Ponto Decimal) do padrão numérico lido da tabela, atendendo à exigência visual.

```assembly
    MOV A, 40H           ; Carrega a contagem atual
    MOVC A, @A+DPTR      ; Busca o padrao (0 a 9)
    MOV C, F0            
    MOV ACC.7, C         ; Se F0=1, Ponto apaga. Se F0=0, acende.
    MOV P1, A            ; Envia ao display
```

### 4.4 Sub-Rotina de Reset Controlado

Sempre que a direção muda ou o limite de 10 voltas é atingido, o Timer é temporariamente parado, a variável `40H` é zerada, e o hardware é recarregado para ficar na iminência do próximo pulso.

```assembly
RESET_TIMER:
    CLR TR1              ; Para o temporizador (Requisito da entrega)
    MOV 40H, #00H        ; Zera a variavel de processo
    MOV TL1, #0FFH       ; Recarrega TL1 com FFH
    SETB TR1             ; Reinicia a contagem
    RET
```

## 5 Código Fonte Completo

Este é o firmware final integrado desenvolvido para o projeto em linguagem Assembly do microcontrolador 8051:

```assembly
; SEL0433 - APLICACAO DE MICROPROCESSADORES
; Projeto 1: Sistema de dosagem rotativa
; Entrega Final - Integracao da mudanca de direcao e limites
; Alunos: Joao Vitor Miranda Sousa | 14802702
; Fernando Shoji Ogusuku | 15636682

ORG 0000H            ; Vetor de Reset (Endereco inicial)
SJMP MAIN            ; Salta para o programa principal para desviar das interrupcoes

ORG 001BH            ; Vetor de Interrupcao do Timer 1 (Dispara quando TF1 vai a 1)
SJMP INT_TIMER1      ; Salta para a rotina de servico de interrupcao

ORG 0030H            ; Inicio da memoria de programa segura
MAIN:
    ; 1) Inicializacao da Pilha (Stack)
    MOV SP, #2FH     ; Protege a area de memoria para sub-rotinas (Stack usara 30H em diante)

    ; 2) Inicializacao da Variavel de Processo (Contador de Voltas)
    ; CORRECAO: Variavel movida para 40H para nao colidir com a Pilha!
    MOV 40H, #00H    ; Zera o contador de voltas na memoria RAM (Vai de 0 a 9)

    ; 3) Configuracao inicial do Motor e Variavel de Estado (F0)
    SETB P3.0        ; Motor girando para um sentido (P3.0 = 1)
    CLR P3.1         ; (P3.1 = 0)
    SETB F0          ; F0 = 1 (Estado inicial acompanha chave SW0 solta)

    ; 4) Configuracao do display
    MOV DPTR, #TABELA ; Aponta para a tabela de digitos na ROM

    ; =========================================================================
    ; 5) Configuracao do Timer 1 (Modo Contador, Auto-Reload)
    ; TMOD = 60H (0110 0000B): Timer 1 como Contador (C/T=1), Modo 2.
    ; =========================================================================
    MOV TMOD, #01100000B
    MOV TH1, #0FFH   ; Valor de recarga automatica
    MOV TL1, #0FFH   ; Valor inicial (estoura e interrompe no primeiro pulso)

    ; Habilita Interrupcoes
    SETB ET1         ; Habilita a interrupcao exclusiva do Timer 1
    SETB EA          ; Habilita a chave geral de interrupcoes do sistema
    SETB TR1         ; Liga o Timer 1 para "escutar" os pulsos

LOOP:
    ACALL VERIFICA_CHAVE ; Checa botao e muda sentido se necessario

    ; Atualiza o display de 7 segmentos com o contador
    MOV A, 40H           ; Carrega a contagem atual (Variavel de processo segura em 40H)
    MOVC A, @A+DPTR      ; Busca o padrao na tabela (0 a 9)

    ; Inserir o estado de F0 no bit 7 (Ponto Decimal - DP) do padrao
    MOV C, F0
    MOV ACC.7, C         ; Se F0=1, Ponto apaga. Se F0=0, Ponto acende.

    MOV P1, A            ; Envia padrao numerico + Estado do DP para o display

    SJMP LOOP            ; Loop infinito

; ---------------------------------------------------------
; SUB-ROTINA 1: Verifica a chave SW0 (P2.0)
; ---------------------------------------------------------
VERIFICA_CHAVE:
    JB P2.0, CHAVE_SOLTA ; Se for 1 (solto), pula

CHAVE_PRESSIONADA:
    JNB F0, FIM_VERIFICA ; Se estados sao iguais (ambos 0), nao faz nada
    ACALL MUDA_DIRECAO   ; Se diferentes, inverte o motor
    SJMP FIM_VERIFICA

CHAVE_SOLTA:
    JB F0, FIM_VERIFICA  ; Se estados sao iguais (ambos 1), nao faz nada
    ACALL MUDA_DIRECAO   ; Se diferentes, inverte o motor

FIM_VERIFICA:
    RET

; ---------------------------------------------------------
; SUB-ROTINA 2: Muda a direcao do motor
; ---------------------------------------------------------
MUDA_DIRECAO:
    CPL F0               ; Inverte estado da variavel base
    CPL P3.0             ; Inverte pinos do motor
    CPL P3.1
    ACALL RESET_TIMER    ; REQUISITO: Reinicia a contagem ao mudar de sentido
    RET

; ---------------------------------------------------------
; SUB-ROTINA DEDICADA 3: Reset do Timer e Variavel
; Atende ao requisito de parar o timer, zerar a contagem e reiniciar
; ---------------------------------------------------------
RESET_TIMER:
    CLR TR1              ; Para o temporizador (Requisito da entrega)
    MOV 40H, #00H        ; Zera a variavel de processo (Substitui o zerar TL1)
    MOV TL1, #0FFH       ; Recarrega TL1 com FFH para garantir interrupcao no proximo pulso
    SETB TR1             ; Reinicia a contagem de forma controlada (Requisito da entrega)
    RET

; ---------------------------------------------------------
; ROTINA DE INTERRUPCAO: Timer 1 
; Atende ao requisito de evitar polling no Loop Principal.
; NOTA DE PROJETO: Como o Timer 1 so interrompe por overflow (FFH -> 00H), 
; TL1 e mantido no limite e a contagem real e feita na RAM (40H).
; ---------------------------------------------------------
INT_TIMER1:
    PUSH ACC             ; Salva o Acumulador
    PUSH PSW             ; Salva a Program Status Word

    INC 40H              ; Incrementa a contagem de voltas
    MOV A, 40H
    
    ; Requisito: Comparacao do contador com 10 (Usando RAM em vez de TL1 direto)
    CJNE A, #10, FIM_INT ; Compara com 10. Se nao for, sai.

    ; Requisito: Chama sub-rotina dedicada ao atingir limite
    ACALL RESET_TIMER    

FIM_INT:
    POP PSW              ; Restaura PSW
    POP ACC              ; Restaura Acumulador
    RETI                 ; Retorna para o Loop Principal

; ---------------------------------------------------------
; TABELA DO DISPLAY DE 7 SEGMENTOS (Anodo Comum - 0 a 9)
; ---------------------------------------------------------
TABELA:
    DB 0C0H  ; 0
    DB 0F9H  ; 1
    DB 0A4H  ; 2
    DB 0B0H  ; 3
    DB 099H  ; 4
    DB 092H  ; 5
    DB 082H  ; 6
    DB 0F8H  ; 7
    DB 080H  ; 8
    DB 090H  ; 9

END
```

## 6 Guia de Simulação e Testes

Para validar a solução matemática e de hardware apresentada, siga os passos no EdSim51. Os resultados da execução e a resposta do sistema aos comandos externos podem ser observados nas figuras abaixo.

1. Cole o código fonte na janela de texto do EdSim51.
2. Clique no botão **Assm** (Assemble) para compilar o código.
3. No painel inferior, ative a caixa de seleção **Motor Enabled**.
4. Clique no botão **Run**.

<br>

<p align="center">
  <img src="https://github.com/user-attachments/assets/5d5dfc66-69af-4298-84aa-040e722352f3" height="400">
</p>

<p align="center">
  <em>Figura 1: Interface geral do simulador EdSim51 durante a execução estável do firmware.</em>
</p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/2260a50b-d3b4-49f2-bc6d-049b5d9c8a96" height="400">
</p>

<p align="center">
  <em>Figura 2: Evidência de teste reativo: Ponto Decimal (DP) aceso indicando a mudança de estado da flag F0 e o motor operando em sentido invertido após o acionamento de SW0.</em>
</p>

<br>

**Comportamentos observados e validados:**

* **Limite Natural:** Ao atingir o valor 9 (Figura 1), o próximo pulso retorna o display a 0 via interrupção (Vetor 001BH), garantindo continuidade sem travamentos.
* **Inversão Reativa:** Ao pressionar `SW0` (P2.0), o sistema responde instantaneamente: o motor inverte a rotação, o contador na RAM é zerado e o Ponto Decimal sinaliza o novo sentido (Figura 2).

## 7 Conclusão

A conclusão deste projeto evidencia o sucesso na implementação de um firmware robusto para o controlo de um sistema de dosagem rotativa, utilizando a arquitetura do microcontrolador 8051. Através da metodologia de Aprendizagem Baseada em Problemas (PBL), o desenvolvimento ocorreu de forma evolutiva: partindo do controlo básico de I/O digital e mapeamento de displays de 7 segmentos nos primeiros *checkpoints*, até culminar na gestão complexa de periféricos internos nesta entrega final.

O principal marco técnico e arquitetural do sistema foi a transição de um modelo de varredura contínua (*polling*) para uma arquitetura orientada a eventos (*Event-Driven*), suportada por interrupções de hardware. A configuração do Timer 1 no Modo 2 (Auto-Reload de 8 bits) provou ser a estratégia mais eficiente para a contagem precisa dos pulsos mecânicos provenientes do motor. Esta abordagem libertou a Unidade Lógica e Aritmética (ALU) de ciclos de espera ociosos, permitindo que o *Loop Principal* se dedicasse exclusivamente à monitorização da chave de segurança (`SW0`). Como resultado, o sistema garantiu uma resposta imediata e determinística na inversão de marcha do motor, sem qualquer perda na integridade da contagem de voltas.

Adicionalmente, o projeto reforçou a importância vital da gestão rigorosa de memória em sistemas embarcados de baixo nível. A alocação consciente do *Stack Pointer* (`SP`) para fora da zona de conflito (iniciando em `2FH`) e o armazenamento da variável de processo em endereços seguros da RAM (como o `40H`) preveniram a sobreposição de dados críticos pelos endereços de retorno das interrupções e sub-rotinas (*Stack Overflow/Overlap*). Este tipo de precaução é o que distingue um protótipo de um firmware preparado para a fiabilidade exigida num ambiente industrial real.

Por fim, a lógica de *reset* automático da contagem ao atingir o limite estipulado (10 voltas), aliada ao reinício da contagem e à sinalização visual (ponto decimal) durante a alteração do sentido de rotação, cumpriram integralmente todos os requisitos funcionais definidos no guião. Esta prática consolidou o domínio dos alunos sobre a linguagem Assembly, manipulação de registos (*SFRs*) e mapeamento de *hardware*, competências que são essenciais para o desenvolvimento de soluções robustas em sistemas digitais e microprocessadores.

---

## 8 Referências

1. OLIVEIRA, Pedro. **Slides, Apostilas e Notas de Aula da Disciplina SEL0433 - Aplicação de Microprocessadores**. Escola de Engenharia de São Carlos (EESC-USP). [Principal Referência Teórica e Técnica].
2. OLIVEIRA, Pedro. **Códigos – Conceitos Iniciais (Ambiente EdSim51)**. Arquivo digital (.zip) de suporte e exemplos práticos da disciplina. Disponibilizado via e-Disciplinas USP.
3. EDSIM51. *Simulator for the 8051 Microcontroller*. Documentação oficial e manual de instruções do simulador. Disponível em: <https://www.edsim51.com/>.
