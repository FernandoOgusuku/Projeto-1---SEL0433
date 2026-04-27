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
