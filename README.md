# Atividade 06 – Cadeia de custódia e análise de MACtimes

> **Professor Esp. Marcelino Dias da Silva Junior** · UNINOVE · Disciplina: **Auditoria Forense de Sistemas Digitais**
> **Aula relacionada:** Cadeia de custódia e análise de MACTimes (retoma o caso da estação FIN-014 da aula de relatórios)

## 🎯 Objetivo
Montar a **linha do tempo** de um incidente usando os **MACtimes** dos arquivos e registrar a **cadeia de custódia** da evidência, conferindo o hash a cada entrega.

## 📚 Conceito em 1 minuto
| MACtime | Linux (`stat`) | O que registra |
|---|---|---|
| **M** – modify | `Modify` | Última alteração do **conteúdo** |
| **A** – access | `Access` | Último **acesso** (leitura) |
| **C** – change | `Change` | No Linux, última alteração dos metadados do arquivo (por exemplo, permissões ou proprietário). Não é a data de criação. |

- No Linux, esses horários são armazenados nos metadados do sistema de arquivos, normalmente associados ao inode. No Windows, a data de criação é um horário distinto; não se deve tratar `ctime` do Linux e criação do Windows como equivalentes.
- O SHA-256 de um arquivo é calculado sobre seus bytes; os MACtimes mantidos externamente pelo sistema de arquivos não são lidos pelo `sha256sum`. Um pacote `tar` pode serializar metadados, como horários, dentro do próprio pacote, e esses bytes também entram no hash do pacote.
- **Cadeia de custódia:** quem, quando, onde, como e por que manuseou a evidência, da coleta até o juízo.

## Pré-requisitos

Os comandos abaixo devem ser executados na pasta do exercício, que deve conter `preparar.sh`. Esse script deve disponibilizar a evidência em `saida/FIN-014`. Esses arquivos não estão incluídos neste repositório; sem eles, não é possível reproduzir a linha do tempo nem gerar um pacote de evidência.

## 🧪 Parte 1 – Lendo os MACtimes

Esta atividade usa uma evidência de treinamento. Em uma investigação real, preserve a mídia original e faça a análise em uma imagem ou cópia forense verificada, usando bloqueador de escrita quando aplicável.

```bash
bash preparar.sh
stat saida/FIN-014/home/jsilva/Documentos/pagamentos.xlsx
```

Veja como cada ação muda um MACtime diferente:

```bash
cp saida/FIN-014/home/jsilva/Documentos/ferias.txt saida/teste.txt
stat -c 'M=%y | A=%x | C=%z  %n' saida/teste.txt
chmod 600 saida/teste.txt
stat -c 'M=%y | A=%x | C=%z  %n' saida/teste.txt
echo "nova linha" >> saida/teste.txt
stat -c 'M=%y | A=%x | C=%z  %n' saida/teste.txt
```

📌 O `chmod` altera o **C**. Escrever no arquivo altera o **M** e o **C**. O **A** pode não mudar a cada leitura, pois seu registro depende das opções de montagem do sistema de arquivos.

## 🧪 Parte 2 – Linha do tempo do incidente

Confirme que a evidência existe antes de executar os comandos:

```bash
test -d saida/FIN-014 || { echo "Evidência ausente: saida/FIN-014" >&2; exit 1; }
```

Liste todos os arquivos **ordenados pela data de modificação** (incluindo os ocultos):

```bash
find saida/FIN-014 -type f -printf '%TY-%Tm-%Td %TH:%TM  M  %p\n' | sort
```

E pela data de **acesso**:

```bash
find saida/FIN-014 -type f -printf '%AY-%Am-%Ad %AH:%AM  A  %p\n' | sort
```

Junte tudo numa linha do tempo única:

```bash
{ find saida/FIN-014 -type f -printf '%TY-%Tm-%Td %TH:%TM  modificado  %p\n'
  find saida/FIN-014 -type f -printf '%AY-%Am-%Ad %AH:%AM  acessado    %p\n'
} | sort | grep "2026-08-14" | tee saida/linha_do_tempo.txt
```

Os horários exibidos por `find` usam o fuso local do sistema. Registre o fuso ao documentar a análise e não misture horários locais com UTC.

### Resultado da execução – 14/08/2026

| Horário | Evento identificado |
|---|---|
| 22h10 | Não foi possível identificar o arquivo ou a ação: a evidência `saida/FIN-014` não está neste repositório. |
| 22h17 | `pagamentos.xlsx` foi modificado, conforme informado no enunciado. |
| 22h34 | Não foi possível identificar o arquivo ou a ação sem a evidência. |
| 22h39 | `pagamentos.xlsx` foi acessado, conforme informado no enunciado. |

## 🧪 Parte 3 – Resultado da cadeia de custódia

A coleta e o cálculo do hash não foram executados neste workspace porque `saida/FIN-014` não existe. Portanto, nenhum pacote ou registro foi criado. Com a cópia de trabalho da evidência disponível, execute a coleta na pasta do exercício:

```bash
set -euo pipefail
test -d saida/FIN-014 || { echo "Evidência ausente: saida/FIN-014" >&2; exit 1; }
tar czf saida/EV-01_FIN-014.tar.gz -C saida FIN-014
HASH=$(sha256sum saida/EV-01_FIN-014.tar.gz | awk '{print $1}')
if [[ ! -f saida/cadeia_custodia.csv ]]; then
  printf 'data_hora_utc;acao;entregue_por;recebido_por;sha256\n' > saida/cadeia_custodia.csv
fi
printf '%s;COLETA;-;%s;%s\n' "$(date -u '+%F %T')" "$(whoami)" "$HASH" >> saida/cadeia_custodia.csv
printf 'SHA-256 da coleta: %s\n' "$HASH"
```

Ao receber uma cópia, coloque-a no caminho `saida/EV-01_FIN-014.tar.gz` e compare-a com o hash informado pelo responsável anterior. O registro de entrega só é acrescentado quando a comparação confere; em caso de divergência, registra-se a recusa e não se aceita a evidência:

```bash
set -euo pipefail
read -r -p 'SHA-256 informado pelo entregador: ' HASH_ESPERADO
read -r -p 'Entregue por: ' ENTREGUE_POR
read -r -p 'Recebido por: ' RECEBIDO_POR
CONF=$(sha256sum saida/EV-01_FIN-014.tar.gz | awk '{print $1}')
if [[ "$CONF" != "$HASH_ESPERADO" ]]; then
  printf '%s;RECUSA_HASH_DIVERGENTE;%s;%s;%s\n' "$(date -u '+%F %T')" "$ENTREGUE_POR" "$RECEBIDO_POR" "$CONF" >> saida/cadeia_custodia.csv
  echo "HASH DIVERGENTE - NAO RECEBER!" >&2
  exit 1
fi
printf '%s;ENTREGA PARA ANALISE;%s;%s;%s\n' "$(date -u '+%F %T')" "$ENTREGUE_POR" "$RECEBIDO_POR" "$CONF" >> saida/cadeia_custodia.csv
column -s';' -t saida/cadeia_custodia.csv
```

## ❓ Perguntas
1. Qual é a sequência de eventos da noite de 14/08? Relacione cada horário a um arquivo.
2. O arquivo `pagamentos.xlsx` foi modificado às 22h17 e acessado às 22h39. O que isso sugere?
3. Por que abrir um arquivo da evidência original "só para olhar" pode destruir uma prova de horário?
4. Na Parte 3, o que o recebedor deve fazer se o hash **não** conferir?
5. Quais campos um formulário de cadeia de custódia precisa ter, no mínimo?

### Respostas
1. A tabela da Parte 2 resume o que pode ser afirmado com os dados disponíveis. Os eventos das 22h10 e 22h34 e os arquivos correspondentes dependem da evidência ausente neste repositório.
2. Indica que o conteúdo de `pagamentos.xlsx` foi modificado às 22h17 e que houve um acesso registrado às 22h39. Esses horários, por si só, não identificam quem realizou as ações, por que foram feitas ou exatamente o que foi alterado.
3. A leitura pode atualizar o horário de acesso (`atime`), conforme as opções de montagem do sistema de arquivos, modificando um metadado relevante para a análise. Preserve o original e analise uma cópia forense, documentando os procedimentos.
4. Deve recusar o recebimento, não assinar a transferência como aceita, preservar os registros, comunicar e documentar a divergência. Só deve aceitar após esclarecer a diferença e confirmar que o hash da cópia recebida corresponde ao hash informado.
5. No mínimo: identificação e descrição da evidência; data e hora, com fuso, de cada movimentação; ação e motivo; local de origem e destino; identificação e assinatura de quem entrega e recebe; condição e lacre da evidência; e hash de integridade. Cada transferência deve ser registrada para manter o histórico contínuo de custódia.
