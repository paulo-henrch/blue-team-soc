# LAB — Auditoria local de acesso a arquivos no Windows 10

## Objetivo

Gerar um evento de auditoria de acesso a arquivo no Windows 10 e usar o Visualizador de Eventos para identificar a conta, o arquivo, o processo e a operação registrada.

## Ambiente

- Estação local com Windows 10
- Arquivo de teste: `Segurança.txt`, na Área de Trabalho sincronizada pelo OneDrive
- Ferramentas nativas: `auditpol`, propriedades de segurança do arquivo e Visualizador de Eventos

A auditoria foi configurada diretamente na estação com o `auditpol` e com uma regra no arquivo. O lab foi realizado sem GPO, Active Directory, Windows Server ou máquina virtual.

## O que fiz no lab

Usei o `auditpol` para habilitar a auditoria de sucesso na subcategoria **Sistema de Arquivos** do Windows 10. Depois, configurei uma entrada de auditoria na SACL do `Segurança.txt`, pelas propriedades de segurança do arquivo.

Abri o arquivo `Segurança.txt` e li seu conteúdo. Em seguida, no Visualizador de Eventos, consultei o log **Segurança**, filtrei pelo ID **4663** e identifiquei o registro pelo caminho do arquivo. O evento selecionado foi gerado pelo `FileCoAuth.exe`, processo do OneDrive.

## Evidência no Visualizador de Eventos

A captura mostra um evento **4663**, da categoria **File System**, com a palavra-chave **Sucesso da Auditoria**.

![Evento 4663 no Visualizador de Eventos com ReadAttributes registrado para Segurança.txt](evento-4663.png)

| Campo | Valor observado |
| --- | --- |
| Data e hora | 18/11/2026 às 22:22:20, no horário da estação |
| Conta | Usuário local da estação |
| Objeto | `Segurança.txt`, na Área de Trabalho sincronizada pelo OneDrive |
| Processo | `FileCoAuth.exe`, sob `C:\Program Files\Microsoft OneDrive\` |
| ID do processo | `0x26a0` |
| Acesso registrado | `ReadAttributes` |
| Máscara de acesso | `0x80` |

O evento 4663 registra o uso efetivo de um direito de acesso. Neste registro, `ReadAttributes` indica leitura de atributos do arquivo. A leitura do conteúdo seria evidenciada por uma operação `ReadData` registrada para o mesmo arquivo. [Microsoft Learn — Evento 4663](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4663)

## SOC N1 — Triagem do evento

Cruzei os campos **Requerente**, **Nome do objeto**, **Nome do processo** e **Acessos**. Às 22:22:20, o `FileCoAuth.exe` usou o direito `ReadAttributes` em `Segurança.txt` sob a conta local da estação.

O caminho do executável e a localização do arquivo, em uma pasta sincronizada, são compatíveis com a atividade do OneDrive nesse ambiente. Minha conclusão para a triagem foi que o evento registra uma **leitura de atributos**. Para corroborar no log a leitura de conteúdo que realizei, eu buscaria um evento com `ReadData` para o mesmo arquivo e correlacionaria o horário e o processo.
