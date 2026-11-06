# Laboratório – Análise de UDP no Wireshark

## Objetivo

Observar o comportamento do UDP na troca de mensagens entre Windows e Linux, relacionando a captura de rede com os sockets e o conteúdo recebido pelas aplicações.

## Resumo teórico

O UDP é um protocolo de transporte que envia datagramas sem handshake e não fornece confirmação de entrega, retransmissão ou ordenação. Seu cabeçalho tem **8 bytes**, divididos entre porta de origem, porta de destino, comprimento e checksum.

É utilizado em aplicações como DNS, DHCP, jogos e comunicação em tempo real. A menor sobrecarga não significa que será sempre mais rápido que o TCP.

O checksum detecta possíveis erros, mas não fornece criptografia ou autenticidade. No IPv4, seu uso é opcional; no IPv6, é obrigatório como regra geral. O estado **unverified** indica apenas que o Wireshark não verificou o cálculo.

## Ambiente

- Windows: `192.168.0.90`, utilizando PowerShell e porta UDP `56000`.
- VM Debian: `192.168.0.114`, utilizando Netcat e porta UDP `46000`.
- Wireshark: captura na interface `enp0s3`.

## Filtros de exibição no Wireshark

| Filtro | Finalidade |
|---|---|
| `udp.port == 53` | Consultas e respostas DNS sobre UDP |
| `udp.port == 46000` | Datagramas da troca de mensagens com o Netcat |
| `udp.port == 46000 || icmp` | Troca de mensagens e possíveis erros ICMP |
| `udp.port == 45678 || icmp` | Teste anterior com o receptor encerrado |
| `udp.port == 9999` | Tráfego que interferiu na tentativa inicial |

São filtros de exibição: selecionam o que aparece na lista, sem alterar os pacotes já capturados. Nos filtros com `|| icmp`, aparecem também mensagens ICMP de outras comunicações; é necessário conferir o datagrama citado dentro do erro.

## Comportamento observado

### Envio e recepção

O Windows enviou a mensagem `Teste UDP - laboratorio SOC` diretamente para a VM, sem estabelecimento prévio de conexão. O texto apareceu no terminal do Netcat, confirmando a entrega à aplicação neste teste.

| Campo | Valor observado |
|---|---|
| Origem | `192.168.0.90:56000` |
| Destino | `192.168.0.114:46000` |
| Dados | 27 bytes |
| Comprimento UDP | 35 bytes: 8 de cabeçalho + 27 de dados |

![Datagrama enviado pelo Windows](UDP_1.png)

Antes do recebimento, o socket do Netcat aparecia como `UNCONN`, vinculado a `0.0.0.0:46000`. Depois, passou a mostrar `ESTAB`, associado a `192.168.0.90:56000`.

Nesse Netcat, **ESTAB representa a associação local do socket a um remetente; não significa que ocorreu um handshake UDP**. A consulta com `ss -uanp` permitiu correlacionar o socket com o processo responsável.

![Socket UDP associado ao Windows](SS_UDP_NETCAT_TERMINAL.PNG)

### Resposta e conteúdo sem criptografia

A VM respondeu de `192.168.0.114:46000` para `192.168.0.90:56000`, invertendo os endereços e as portas. A mensagem foi recebida no PowerShell. **Essa resposta foi gerada pela aplicação, não por um mecanismo de ACK do UDP.**

![Resposta da VM capturada no Wireshark](UDP_2.png)

![Mensagem recebida no Windows](UDP_3.PNG)

Os textos ficaram legíveis na captura porque foram enviados **sem criptografia**. A representação hexadecimal não protege o conteúdo. O UDP não criptografa os dados por si só; essa proteção depende dos protocolos utilizados pela aplicação.

Ferramentas de análise também podem exibir esse conteúdo. No Wireshark, a opção **Follow → UDP Stream** permite organizar a visualização dos dados da conversa.

## Outras observações do laboratório

- **DNS:** a consulta foi enviada ao servidor configurado, na porta `53`. A resposta inverteu os endereços e as portas e manteve o identificador da consulta. Trata-se de uma resposta DNS, não de confirmação própria do UDP.
- **Porta sem receptor:** após encerrar o Netcat em um teste anterior na porta `45678`, a VM respondeu com ICMP **Destination unreachable — Port unreachable**. A máquina continuava acessível, mas não havia receptor naquela porta.
- **Interferência na porta 9999:** um broadcast de `192.168.0.111` para `255.255.255.255:9999`, identificado pelo Wireshark como TP-Link, fez o Netcat se associar a outro remetente. A correlação entre captura e socket explicou a interferência, sem evidência suficiente para classificá-la como incidente.

## Aprendizados para SOC N1

A análise mostrou a importância de correlacionar **IPs, portas, direção do tráfego, sockets e processos**. Um datagrama capturado comprova sua presença no ponto de captura; a mensagem exibida no terminal confirmou a entrega à aplicação.

Erros ICMP e ausência de resposta precisam ser interpretados com o contexto. Também foi possível reconhecer conteúdo em texto claro e distinguir respostas da aplicação dos mecanismos do protocolo de transporte.
