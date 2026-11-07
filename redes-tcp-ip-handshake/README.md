# Laboratório – Abertura e fechamento de conexão TCP no Wireshark

## Objetivo

Observar o estabelecimento e o encerramento de conexões TCP e compreender os campos Sequence Number (Seq), Acknowledgment Number (Ack), o tamanho dos dados (Len) e as flags SYN, ACK e FIN.

## Ambiente e procedimento

- VM Debian: 192.168.0.114, interface enp0s3.
- Wireshark para captura e análise.
- Requisição HTTPS gerada com curl, forçando IPv4 e HTTP/1.1:

```bash
curl -4 --http1.1 -I https://www.google.com
```

A captura foi iniciada antes do comando, que terminou normalmente. Os prints de abertura e fechamento foram obtidos em conexões distintas; por isso, os IPs de destino e as portas de origem podem variar.

## 1. Abertura – three-way handshake

![Abertura da conexão TCP](TCP_Abertura.png)

Na captura de abertura, o cliente utiliza a porta 48764 e se conecta ao servidor 142.251.157.119 na porta 443.

| Pacote | Sentido | Flags | Seq | Ack | Len |
|---|---|---|---|---|---|
| 30 | Cliente → servidor | SYN | 0 | — | 0 |
| 31 | Servidor → cliente | SYN, ACK | 0 | 1 | 0 |
| 32 | Cliente → servidor | ACK | 1 | 1 | 0 |

1. O cliente envia SYN para iniciar a conexão e anunciar sua sequência inicial.
2. O servidor confirma o SYN do cliente e anuncia sua própria sequência inicial.
3. O cliente confirma o SYN do servidor, concluindo o handshake.

Cada lado tem sua própria sequência. O SYN consome uma posição, mesmo com Len=0. Um ACK sem dados não consome posições.

### Números relativos e números reais

O número de sequência inicial (ISN) é escolhido independentemente por cada lado, com mecanismos que buscam torná-lo imprevisível; não é necessariamente zero. Pode ser entendido, neste estudo inicial, como um valor gerado de forma pseudoaleatória, e não como uma contagem de pacotes.

O Wireshark exibe números relativos por padrão: representa o ISN de cada lado como zero para facilitar a leitura. O campo **Sequence Number (raw)** mostra o valor real do cabeçalho.

Em uma das conexões analisadas, o SYN do cliente apresentou **Seq (raw)=4000480093**, confirmado pelo servidor com **Ack (raw)=4000480094**. Em outra conexão, o SYN apresentou **617544573**, confirmado com **617544574**. Na visualização relativa, a relação é a mesma: Seq=0 → Ack=1.

## 2. Entendendo Seq, Ack e Len

| Campo | Significado |
|---|---|
| Seq | Posição na sequência de quem envia; em um segmento de dados sem SYN, identifica o primeiro byte transportado. |
| Ack | Próxima posição esperada do outro lado; confirma cumulativamente as posições anteriores recebidas em ordem. É significativo quando a flag ACK está ativa. |
| Len do TCP | Quantidade de bytes de dados transportados pelo TCP, sem os cabeçalhos. |
| Length da lista | Tamanho do quadro mostrado na captura, incluindo os cabeçalhos capturados. |

**TCP numera bytes, não pacotes.** Se um segmento tem Seq=500 e Len=100, seus dados ocupam as posições 500 até 599. Recebendo tudo em ordem, o outro lado confirma com Ack=600.

```text
Última posição dos dados = Seq + Len − 1
Próxima posição esperada = Seq + Len
```

Essas contas simples consideram dados sem SYN ou FIN e sem a volta do contador de 32 bits. SYN e FIN acrescentam uma posição cada.

### Exemplo didático: envio de 800 bytes após o handshake

Supondo quatro envios de 200 bytes, com confirmação após cada envio:

| Envio do cliente | Posições dos dados | Confirmação do servidor |
|---|---|---|
| Seq=1, Len=200 | 1 até 200 | Ack=201 |
| Seq=201, Len=200 | 201 até 400 | Ack=401 |
| Seq=401, Len=200 | 401 até 600 | Ack=601 |
| Seq=601, Len=200 | 601 até 800 | Ack=801 |

Na prática, não é obrigatório receber um ACK para cada segmento antes de enviar o próximo. Uma confirmação pode abranger vários segmentos.

### Dados TLS depois do handshake

Na captura estudada, após o handshake TCP, o cliente enviou um TLS Client Hello com SNI www.google.com, Seq=1, Ack=1 e Len=1574. Esses dados ocupam as posições 1 até 1574.

Foram observadas duas confirmações:

- Ack=1401: confirma as posições 1 até 1400.
- Ack=1575: confirma todas as posições até 1574, incluindo as já confirmadas.

A captura local mostrou um envio grande, mas as confirmações avançaram em etapas. A segmentação após o ponto de captura, por mecanismos de offload, é uma explicação possível; os prints isolados não comprovam essa causa.

Embora o Wireshark identifique a mensagem como TLS, ela é transportada como dados pelo TCP. No handshake TCP dos prints, Len=0 porque ainda não há dados transportados nesses três segmentos.

## 3. Fechamento da conexão

![Fechamento da conexão TCP](TCP_Fechamento.png)

Nesta conexão, o cliente utiliza a porta 60016 e o servidor 142.251.152.119 utiliza a porta 443.

| Pacote | Sentido | Flags | Seq | Ack | Len | Interpretação |
|---|---|---|---|---|---|---|
| 78 | Cliente → servidor | FIN, ACK | 1777 | 5980 | 0 | Cliente informa que terminou seu envio. |
| 79 | Servidor → cliente | ACK | 5980 | 1777 | 0 | Confirma posições anteriores ao FIN; ainda não confirma o FIN na posição 1777. |
| 80 | Servidor → cliente | FIN, ACK | 5980 | 1778 | 0 | Confirma o FIN do cliente e encerra seu próprio envio. |
| 81 | Cliente → servidor | ACK | 1778 | 5981 | 0 | Confirma o FIN do servidor. |

O FIN do cliente ocupa a posição 1777 e é confirmado com Ack=1778. O FIN do servidor ocupa a posição 5980 e é confirmado com Ack=5981.

**A flag ACK, sozinha, não informa qual evento está sendo confirmado: é preciso conferir o valor do Ack.** O pacote 79 não confirma o FIN do cliente; essa confirmação aparece no pacote 80.

### Relação com o encerramento em quatro vias

O modelo didático é:

1. Cliente envia FIN, ACK: terminou de enviar.
2. Servidor envia ACK: confirma o FIN.
3. Servidor envia FIN, ACK: também terminou de enviar.
4. Cliente envia ACK: confirma o FIN do servidor.

Cada sentido é encerrado independentemente. Após receber o FIN do cliente, o servidor ainda pode enviar dados antes de enviar seu próprio FIN. O servidor também pode iniciar o fechamento.

Na captura, a confirmação do FIN do cliente e o FIN do servidor estão combinados no pacote 80. Embora haja quatro pacotes TCP visíveis no trecho, o pacote 79 confirma dados anteriores; não corresponde à segunda etapa do modelo didático.

FIN e ACK podem estar ativos no mesmo cabeçalho: o Wireshark não divide um segmento real em dois. A presença de FIN, ACK é normal em uma conexão estabelecida.

## Filtros utilizados

Todos os pacotes TCP envolvendo a VM:

```wireshark
tcp && ip.addr == 192.168.0.114
```

Somente TCP originado na VM:

```wireshark
tcp && ip.src == 192.168.0.114
```

SYN inicial:

```wireshark
tcp.flags.syn == 1 && tcp.flags.ack == 0
```

Segmentos com FIN, incluindo FIN, ACK:

```wireshark
ip.addr == 192.168.0.114 && tcp.flags.fin == 1
```

O filtro de FIN esconde os ACKs sem FIN. Para acompanhar a conversa completa, selecionar um pacote e usar **Conversation Filter → TCP**, ou aplicar `tcp.stream == N`, substituindo N pelo índice da conversa.

## Aprendizados

- A abertura usa SYN, SYN/ACK e ACK.
- Cada lado possui sua própria sequência; Seq e Ack se referem a sentidos diferentes.
- Seq representa posições e avança pela quantidade de dados, não pela quantidade de pacotes.
- Ack é cumulativo e indica a próxima posição esperada.
- Len do TCP não inclui cabeçalhos.
- SYN e FIN consomem uma posição; ACK sem dados não consome posição.
- O fechamento deve ser interpretado pelas flags e pelos números Seq/Ack em conjunto.
- Para análise de tráfego no SOC, é necessário considerar o contexto da conversa antes de concluir que um comportamento é anormal.
