# Revisão Prática — Laboratórios de Redes TCP/IP

## Objetivo

Documentar a revisão dos três laboratórios realizados durante os estudos de Redes TCP/IP, como parte da preparação para a transição de carreira para Cibersegurança, com foco em Blue Team e SOC N1.

A revisão foi realizada por meio de perguntas e respostas, relacionando os conceitos estudados com situações que podem ocorrer no dia a dia de um analista de segurança.

**Laboratórios revisados:**
- Laboratório 1 — Three-Way Handshake TCP
- Laboratório 2 — SSH e análise de logs de autenticação
- Laboratório 3 — ARP e análise de tráfego na rede local

---

## Laboratório 1 — Three-Way Handshake TCP

### Questão 1 — Estabelecimento de uma conexão TCP

**Pergunta:**

Durante o Three-Way Handshake, o cliente envia um SYN, o servidor responde com SYN/ACK e o cliente envia ACK. Por que são necessárias essas três mensagens? O que o servidor está confirmando quando envia o SYN/ACK?

**Minha análise:**

Não é à toa que o nome é Three-Way Handshake, aperto de mão de três vias.

SYN é o início da conexão, a abertura da conexão, e vem de "Synchronize". É o pedido para iniciar a sincronização.

Depois, o servidor/destino responde com SYN/ACK, informando que recebeu o segmento SYN e aceitou.

Por último, é enviado um ACK, querendo dizer que aceitou o SYN/ACK.

### Questão 2 — Retransmissão TCP

**Cenário:**

1. O cliente envia um SYN.
2. O servidor não responde.
3. Alguns segundos depois, o cliente envia outro SYN.

**Pergunta:**

Por que o cliente enviou outro SYN? Quais poderiam ser as causas de o servidor não ter respondido?

**Minha análise:**

O TCP é orientado à conexão, ou seja, a transmissão só começa após a conexão ter sido estabelecida pelo handshake.

Se o cliente enviou o SYN e o servidor não respondeu, o envio de outro SYN acontece porque o TCP possui um parâmetro de tempo. Como ele não recebeu o SYN/ACK, tenta novamente.

A causa pode ser um problema de rede ou o host estar indisponível. Penso assim.

### Questão 3 — Diferença entre TCP e TLS

**Cenário:**

1. Cliente envia SYN.
2. Servidor responde SYN/ACK.
3. Cliente envia ACK.
4. A conexão TCP é estabelecida.
5. O cliente começa a enviar dados.

**Pergunta:**

Se o TCP já estabeleceu a conexão através do Three-Way Handshake, por que, ao acessar um site HTTPS, ainda precisamos realizar o handshake TLS?

**Minha análise:**

Porque, nesse caso, o handshake só estabeleceu a conexão. Ele não criptografou a transmissão das informações.

O TLS é necessário para criptografar o fluxo de informações entre o cliente e o servidor. O handshake TCP não faz isso.

### Questão 4 — Análise de tráfego HTTPS criptografado

**Pergunta:**

Se o tráfego HTTPS está criptografado pelo TLS, o Wireshark ainda consegue visualizar os endereços IP de origem e destino e as portas TCP utilizadas? Por que essas informações continuam visíveis mesmo com a criptografia TLS?

**Minha análise:**

Sim, é possível visualizar, pois o TLS não criptografa o cabeçalho do TCP.

---

## Laboratório 2 — SSH e Análise de Logs

### Questão 5 — Conexão TCP versus autenticação SSH

**Cenário:**

Durante uma análise no Wireshark, foi identificada uma conexão TCP estabelecida na porta 22 entre:

- Cliente SSH: `192.168.0.90`
- Servidor SSH: `192.168.0.114`

**Pergunta:**

Se o Three-Way Handshake TCP foi concluído com sucesso, isso significa que o usuário conseguiu realizar o login no servidor SSH?

**Minha análise:**

Não. Tanto que fiz um teste sem informar o usuário e a senha no terminal do MobaXterm, e a conexão parou.

O handshake é uma coisa, e conseguir realizar o login é outra. São coisas diferentes.

### Questão 6 — Investigação de autenticação SSH

**Cenário:**

Um analista SOC identifica no Wireshark várias conexões TCP bem-sucedidas para a porta 22 de um servidor.

**Pergunta:**

Você pode afirmar, apenas com essa captura, que alguém conseguiu acessar o servidor via SSH? Qual evidência adicional procuraria no Linux para confirmar se houve uma autenticação bem-sucedida?

**Minha análise:**

Não posso afirmar que conseguiu, pois teria que analisar o log no Linux.

Realizei esse teste e capturei os registros de uma autenticação bem-sucedida e de outra com erro.

### Questão 7 — Identificação de possível ataque de força bruta

**Cenário:**

Durante a investigação de um servidor SSH, foram identificados registros como:

```log
Invalid user teste from 192.168.0.90
Failed password for invalid user teste from 192.168.0.90
```

**Pergunta:**

Se um analista SOC identificar centenas desses registros em poucos minutos, todos originados do mesmo IP, qual tipo de atividade suspeita ele poderia estar investigando? O que diferenciaria esse comportamento de um usuário legítimo que simplesmente digitou a senha errada?

**Minha análise:**

Se viessem de um mesmo endereço IP, suspeitaria de um ataque de força bruta.

O que diferenciaria seria a quantidade de tentativas e o volume em um curto espaço de tempo.

Um ataque de força bruta tentaria várias vezes, e um usuário não tentaria 50 ou 100 vezes.

---

## Laboratório 3 — ARP e Comunicação na Rede Local

### Questão 8 — Funcionamento do protocolo ARP

**Cenário:**

Uma VM Debian possui o endereço IP `192.168.0.114` e precisa se comunicar com o gateway `192.168.0.1`, mas não conhece seu endereço MAC.

**Pergunta:**

Como o protocolo ARP resolve esse problema? Explique o que acontece na rede, desde a solicitação até a resposta do gateway.

**Minha análise:**

O ARP vai enviar uma mensagem do tipo broadcast perguntando quem possui o endereço IP `192.168.0.1`.

Se eu estivesse em um ambiente corporativo, o switch seria responsável por espalhar essa mensagem para todos que estão conectados a ele.

O ARP tem o objetivo de descobrir o endereço MAC a partir do endereço IP, que é o endereço físico da placa de rede.

Por isso, o switch trabalha na camada 2, que é a camada de enlace.

### Questão 9 — Broadcast ARP na rede local

**Cenário:**

Três dispositivos estão conectados ao mesmo switch e pertencem à mesma VLAN:

- Computador A: `192.168.0.10`
- Computador B: `192.168.0.20`
- Gateway: `192.168.0.1`

O computador A precisa descobrir o MAC do gateway e envia um ARP Request em broadcast.

**Pergunta:**

O computador B também receberá essa requisição ARP? Se receber, ele responderá à requisição ou simplesmente a ignorará?

**Minha análise:**

Ele vai receber, sim, pois todos recebem. Porém, não irá responder.

### Questão 10 — ARP e comunicação com redes externas

**Cenário:**

- IP da VM: `192.168.0.114`
- Máscara: `255.255.255.0` (`/24`)
- Gateway: `192.168.0.1`
- Destino: `8.8.8.8`

Comando executado:

```bash
ping 8.8.8.8
```

**Pergunta:**

Como o destino `8.8.8.8` está fora da rede local, o Debian utilizará o ARP para descobrir o endereço MAC do `8.8.8.8` ou o endereço MAC do gateway `192.168.0.1`? Por quê?

**Minha análise:**

Não, não vai descobrir o MAC do `8.8.8.8`, e sim o do gateway, que é o ponto de saída da rede.

### Questão 11 — ARP Spoofing e interceptação de tráfego

**Cenário:**

Uma VM Debian possui a seguinte entrada em sua tabela ARP:

```text
192.168.0.1 → AA:AA:AA:AA:AA:AA
```

Esse é o MAC legítimo do gateway. Outro dispositivo começa a enviar mensagens ARP afirmando:

```text
192.168.0.1 → BB:BB:BB:BB:BB:BB
```

**Pergunta:**

Se a VM aceitar essa informação e atualizar sua tabela ARP, o que poderá acontecer com o tráfego destinado à internet?

**Minha análise:**

Se a VM aceitar, o tráfego agora passará por esse outro gateway, que pode ser um gateway falso, uma máquina ou um dispositivo de rede.

Dessa forma, em vez de estar conectado ao meu gateway padrão, estarei conectado à máquina de um hacker, por exemplo, e meu tráfego agora passará por ele.

---

## Conclusão da revisão

Foram revisados três laboratórios de Redes TCP/IP, com foco na compreensão dos protocolos e na aplicação dos conceitos em situações relacionadas à segurança da informação.

As atividades envolveram análise de conexões TCP, interpretação de tráfego SSH, investigação de logs de autenticação, funcionamento do ARP e identificação de possíveis comportamentos maliciosos.

O objetivo desta documentação é registrar meu entendimento atual sobre os assuntos estudados, acompanhar minha evolução técnica e desenvolver o raciocínio investigativo necessário para atuar como analista SOC N1.

**Ferramentas utilizadas nos laboratórios:**

- Wireshark
- Debian Linux
- VirtualBox
- MobaXterm
- Terminal Linux

**Área de estudo:** Redes TCP/IP e Cibersegurança — Blue Team / SOC N1.
