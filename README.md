# Projeto-1---SEL0433

<div align="center">
  <img src="logo_eesc.jpg" height="80" alt="Logo EESC"> &nbsp;&nbsp;&nbsp;&nbsp;
  <img src="sel.jpg" height="80" alt="Logo SEL">
  <br><br>
  <b>Departamento de Engenharia Elétrica e de Computação</b><br>
  <b>EESC USP | SEL0433 - Aplicação de Microprocessadores</b><br>
  <b>Prof.: Pedro Oliveira</b>
  <br><br>
  <h1>Projeto 1: "Sistema de Dosagem Rotativa"</h1>
  <i>Implementação em Linguagem Assembly (Microcontrolador MCS-51)</i>
  <br><br>
  <img src="usp.jpg" height="80" alt="Logo USP">
</div>

<br>

<div align="right">
  <b>Turma:</b> 2ª feira 14h<br><br>
  <b>Alunos:</b><br>
  João Vitor Miranda Sousa – 14802702<br>
  Fernando Shoji Ogusuku – 15636682
</div>

---

## 📌 Objetivo e Contextualização
Este projeto, estruturado na metodologia PBL (*Problem-Based Learning*), visa o desenvolvimento prático do firmware em *baixo nível* para um microcontrolador da família 8051. O cenário simulado é uma linha de produção de parafusos, onde o sistema deve controlar um dosador rotativo acionado por motor DC. 

A aprendizagem foca na manipulação de registradores (GPR e SFR), controle de fluxo, uso da pilha (*Stack*), sub-rotinas, I/O digital e, fundamentalmente, na aplicação de interrupções de hardware para contagem precisa de eventos externos (pulsos do motor).

---

## ⚙️ Requisitos do Sistema
De acordo com o roteiro do projeto, o sistema deve cumprir integralmente os seguintes requisitos:
1. **Contagem de Voltas:** O motor deve realizar a contagem de suas voltas utilizando uma interrupção interna (Timer 1).
2. **Display:** Exibição da contagem (0 a 9) em um display de 7 segmentos.
3. **Ciclo Automático:** Ao atingir exatamente 10 voltas, o sistema reinicia a contagem automaticamente para 0.
4. **Inversão de Rotação:** O botão `SW0` comanda o sentido do motor. Ao ser alterado, o motor deve inverter imediatamente e zerar a contagem atual.
5. **Indicação Visual de Sentido:** O Ponto Decimal (DP) do display deve refletir o sentido de rotação atual (Aceso para um lado, Apagado para o outro).

---

## 🏗️ Lógica de Implementação e Arquitetura
Para assegurar a robustez do sistema, estruturamos o código separando responsabilidades. O **Loop Principal** gerencia apenas a interface (leitura da chave e atualização do display), enquanto as tarefas de tempo crítico e verificação de limite ficaram a cargo do **Hardware de Interrupção**.

### Mapeamento de Memória e Prevenção de Colisão
Para garantir que o fluxo intenso de interrupções não sobrescreva os nossos dados de contagem, o mapa de memória e os registos de controlo foram definidos conforme a tabela abaixo:

| Endereço / Reg. | Nome Lógico | Justificativa da Aplicação |
| :--- | :--- | :--- |
| `2FH` | `SP` (Stack Pointer) | Inicialização do topo da pilha. Mover o `SP` protege os registos dos Bancos 0 a 3. |
| `40H` | Variável de Processo | Armazena o número de voltas atuais (0 a 9). Escolhido por estar distante do *Stack Pointer*. |
| `F0` | Flag de Usuário | Bit do registo `PSW` utilizado como variável de controlo de estado direcional (1 = horário, 0 = anti-horário). |

### Configuração do Temporizador (TMOD = 60H)
Configuramos o **Timer 1 no Modo 2 (C/T = 1, M1=1, M0=0)**. Neste modo, ele funciona como um **Contador de Eventos Externos com Auto-Reload de 8 bits**.

**Lógica Matemática:** Inicializamos o contador (`TL1` e `TH1`) com o valor máximo `FFH` (255 em decimal). Ao menor movimento do motor, é gerado um único pulso no pino P3.5. Como o contador já está no limite, esse pulso causa um *overflow* instantâneo, ativando a flag `TF1`, que por sua vez dispara o Vetor de Interrupção `001BH`. O hardware então recarrega automaticamente o valor `FFH`, deixando o sistema pronto para o próximo pulso.

---

## 🔍 Análise do Código Fonte (Passo a Passo)

### Vetores de Interrupção e Proteção de Memória
O código desvia da rotina inicial e aloca o espaço exato na ROM para o vetor do Timer 1. Logo após, inicializamos o *Stack Pointer* e nossa variável de processo em uma área segura.

```assembly
ORG 0000H            ; Vetor de Reset
SJMP MAIN            ; Salta para o programa principal
ORG 001BH            ; Vetor de Interrupcao do Timer 1
SJMP INT_TIMER1      ; Salta para a rotina de servico
MAIN:
    MOV SP, #2FH     ; Protege a area de sub-rotinas
    MOV 40H, #00H    ; Zera o contador na memoria RAM segura
