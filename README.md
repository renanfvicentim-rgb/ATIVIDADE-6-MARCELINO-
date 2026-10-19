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
| **C** – change | `Change` | Última alteração dos **metadados** (permissão, dono). No Windows, o "C" é a data de criação |

- Os MACtimes ficam no **inode** do arquivo e **não** entram no cálculo do hash.
- **Cadeia de custódia:** quem, quando, onde, como e por que manuseou a evidência, da coleta até o juízo.

## 🧪 Parte 1 – Lendo os MACtimes

```bash
cd /workspaces/*/labs/06-cadeia-de-custodia-mactimes
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

📌 O `chmod` mudou só o **C**, e escrever no arquivo mudou o **M** e o **C**.

## 🧪 Parte 2 – Linha do tempo do incidente

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

📌 Reconstrua a sequência da noite de **14/08/2026**: o que aconteceu às 22h10, 22h17, 22h34 e 22h39?

## 🧪 Parte 3 – Cadeia de custódia com hash

Empacote a evidência e registre a **coleta**:

```bash
tar czf saida/EV-01_FIN-014.tar.gz -C saida FIN-014
HASH=$(sha256sum saida/EV-01_FIN-014.tar.gz | cut -d' ' -f1)
echo "data_hora_utc;acao;entregue_por;recebido_por;sha256" > saida/cadeia_custodia.csv
echo "$(date -u '+%F %T');COLETA;-;$(whoami);$HASH" >> saida/cadeia_custodia.csv
```

A cada **entrega**, quem recebe **confere o hash** antes de assinar:

```bash
CONF=$(sha256sum saida/EV-01_FIN-014.tar.gz | cut -d' ' -f1)
[ "$CONF" = "$HASH" ] && echo "HASH CONFERE" || echo "HASH DIVERGENTE - NAO RECEBER!"
echo "$(date -u '+%F %T');ENTREGA PARA ANALISE;$(whoami);perito.analista;$CONF" >> saida/cadeia_custodia.csv
column -s';' -t saida/cadeia_custodia.csv
```

## ❓ Perguntas
1. Qual é a sequência de eventos da noite de 14/08? Relacione cada horário a um arquivo.
2. O arquivo `pagamentos.xlsx` foi modificado às 22h17 e acessado às 22h39. O que isso sugere?
3. Por que abrir um arquivo da evidência original "só para olhar" pode destruir uma prova de horário?
4. Na Parte 3, o que o recebedor deve fazer se o hash **não** conferir?
5. Quais campos um formulário de cadeia de custódia precisa ter, no mínimo?
