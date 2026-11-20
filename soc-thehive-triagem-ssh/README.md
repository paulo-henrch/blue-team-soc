# Laboratório SOC N1 — Triagem de falhas de autenticação SSH no TheHive

## Objetivo

Subi o TheHive em uma VM no Google Cloud (GCP) usando Docker Compose. Meu objetivo com este laboratório é praticar a detecção, a análise e a triagem de eventos de segurança para atuar como analista SOC N1 no Blue Team.

## Cenário simulado

Nesta etapa, o TheHive era a ferramenta disponível para registrar o caso. Simulei o recebimento de um alerta do Wazuh sobre múltiplas falhas de autenticação SSH e abri manualmente um caso no TheHive para iniciar a triagem.

## Abertura do caso no TheHive

Abri o **caso #1 — [LAB] Múltiplas falhas de autenticação SSH — srv-app-01**, com severidade inicial **Medium**.

| Campo do alerta simulado | Valor |
|---|---|
| Host de destino | `srv-app-01` |
| Serviço | SSH |
| IP de origem | `198.51.100.27` |
| Usuário informado nas tentativas | `admin` |
| Falhas de autenticação | 12 |
| Janela do alerta | 3 minutos |

Na descrição do caso, registrei que a triagem deveria verificar se houve autenticação bem-sucedida e se a origem era esperada para aquele servidor.

![Caso #1 na lista de casos do TheHive][lista-de-casos]

![Descrição e severidade inicial do caso na aba General][detalhes-do-caso]

## Triagem do alerta

Criei três tarefas no TheHive para seguir minha linha de raciocínio na triagem do alerta e registrei o resultado de cada consulta simulada em **Activity**. As três ficaram com o status **Completed**.

![As três tarefas de triagem concluídas no TheHive][tarefas-concluidas]

### 1. Validar origem do acesso SSH

> A consulta apontou que o endereço IP está fora da lista autorizada para SSH no srv-app-01.

### 2. Verificar autenticação bem-sucedida na janela do alerta

> Na janela consultada, o SIEM registrou 12 falhas de autenticação SSH e nenhum evento de sucesso para o IP e o usuário analisados

### 3. Validar a conta admin e seus privilégios

> A conta admin existe localmente no srv-app-01 e possui privilégios efetivos, podendo executar comandos como root ou via sudo

## Minha análise

Na análise inicial, diferenciei o estabelecimento da conexão TCP da autenticação SSH. A conclusão do handshake estabelece a conexão TCP com o host `srv-app-01`. No cenário simulado, os logs indicam que a comunicação com o host chegou à etapa de autenticação SSH.

Na janela consultada, os logs simulados do `srv-app-01`, exibidos no SIEM, registraram falhas de autenticação e nenhum evento de sucesso para o IP e o usuário analisados.

Obs. A conexão TCP e a autenticação SSH são etapas diferentes da comunicação. A conexão permite que o cliente prossiga para a tentativa de autenticação.

O volume de falhas em uma janela curta levantou a hipótese de brute force. Também considerei a possibilidade de um acesso legítimo com erro de senha. Para avaliar o contexto, consultei se o IP estava autorizado para SSH naquele servidor no horário do alerta e validei, no cenário simulado, a existência da conta e seus privilégios.

Cadastrei o IP de origem `198.51.100.27` como observável no caso para vincular o endereço analisado ao registro da triagem.

## Conclusão e encerramento do caso

Após concluir as tarefas, registrei em **Comments** a decisão de encaminhar o caso ao SOC N2, conforme o playbook do laboratório. Em seguida, preenchi o **Summary** e encerrei o caso para concluir o exercício.

Classifiquei o caso como **True Positive** porque a consulta simulada ao SIEM confirmou as 12 falhas de autenticação SSH apontadas pelo alerta. Selecionei **No** no campo **Impact** porque a triagem N1 não identificou impacto no escopo analisado. A hipótese de brute force permaneceu em aberto.

### Resumo registrado no TheHive

> Triagem N1 concluída no laboratório. O SIEM simulado registrou 12 falhas de autenticação SSH em três minutos no srv-app-01, com origem 198.51.100.27 e usuário admin. Na janela consultada, não foram encontrados eventos de autenticação bem-sucedida para esse IP e usuário. A origem estava fora da lista autorizada para SSH no servidor, e a validação simulada indicou que admin é uma conta local com privilégios elevados.
>
> Conforme o playbook do laboratório, registrei a decisão de encaminhar o caso ao SOC N2. A hipótese de brute force ficou em aberto. O fechamento no TheHive marca o fim do exercício de triagem N1.

![Caso encerrado no TheHive com status True Positive, Impact No, resumo e comentário de escalonamento][encerramento-do-caso]

[lista-de-casos]: <./image(20261120-161234).png>
[detalhes-do-caso]: <./image(20261120-161324).png>
[tarefas-concluidas]: <./image(20261120-161419).png>
[encerramento-do-caso]: <./image(20261120-161833).png>