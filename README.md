# doc-extract-gate

Extração de nota fiscal em que a parte interessante não é a extração. É o que o fluxo faz quando o
modelo não tem certeza.

Um extrator cego deduz o campo que falta, erra, e ninguém percebe até o número aparecer numa
planilha três semanas depois. Este se recusa a deduzir: para e pergunta a uma pessoa só o campo que
não conseguiu ler.

## O contrato

Um arquivo cai numa pasta vigiada do Google Drive. Um modelo de visão extrai seis campos:
fornecedor, número da nota, data de emissão, vencimento, total e moeda.

Cada campo volta como três coisas, nunca uma:

```json
{
  "dueDate": {
    "value": "2026-03-14",
    "confidence": 1.0,
    "evidence": "Due date: 14/03/2026"
  }
}
```

`evidence` é o texto exato da página de onde o valor saiu. Campo que volta com confiança alta e sem
texto que o sustente também reprova. Confiança sozinha não é evidência.

## O portão

Um limite na configuração decide.

- Todos os campos passam: a linha é gravada, o arquivo vai pra `processed` e ninguém é chamado.
- Algum campo fica abaixo: nada é gravado. Sai um formulário pelo Telegram só com os campos que
  falharam. O fluxo espera, junta a resposta e só então grava a linha, registrando quais campos uma
  pessoa tocou.

Essa última coluna é o ponto. A taxa de escalonamento vira um número que dá pra acompanhar, em vez
de uma impressão.

## Duas execuções reais

| | Execução A, nota limpa | Execução B, sem vencimento impresso |
|---|---|---|
| Campos com confiança 1,00 | 6 de 6 | 5 de 6 |
| `dueDate` | 1,00, evidência conferida | 0,00, sem evidência |
| Portão | passou | parou |
| Pessoa envolvida | ninguém | um campo |
| Do início ao fim | 4,3 segundos | espera a resposta |

A execução B é o produto. A página trazia a data de emissão e prazo de pagamento de 21 dias, sem
vencimento impresso. O modelo podia ter feito a conta e devolvido uma data errada com confiança.
Devolveu confiança zero e nenhuma evidência.

## Na falha, para

Campo faltando, confiança que não é um número entre 0 e 1 ou resposta ilegível do modelo param a
execução. Nada parcial chega ao destino.

Um teto de custo é conferido antes da chamada ao modelo, em código e não no prompt. Cada chamada
fica registrada numa tabela de custo, com tokens e nome do modelo.

## Como rodar

O `workflow.json` importa em qualquer instância do n8n. Ele está sanitizado: credenciais e ids de
pasta, de tabela e de chat são marcadores.

1. Importe o workflow.
2. Ligue as suas credenciais: Google Drive, OpenRouter (ou troque o nó pela API da Anthropic),
   Telegram e as data tables do n8n.
3. Aponte o `Watch Invoice Folder` pra uma pasta do Drive e defina a pasta de destino em
   `Move to Processed`.
4. Crie duas data tables: `invoice_records` e a tabela de custo.
5. Em `Config`, defina o limite de confiança e o teto de custo.

## Formato

23 nós: gatilho do Drive, download, leitura dos metadados, configuração, checagem do teto, chamada
de visão, validação da saída, registro de custo, avaliação do portão, desvio pelo limite, formulário
do Telegram e junção no caminho com pessoa, convergência, gravação da linha e movimento do arquivo.

O caminho com pessoa e o caminho limpo convergem num único insert, então existe exatamente um lugar
onde a linha é gravada.

## Situação

Rodou na minha instância do n8n em setembro de 2026. Os números acima vêm das execuções 794 e 797,
não de estimativa. Nenhum cliente rodou em produção ainda.

Stack: n8n, OpenRouter (visão), Google Drive, Telegram, data tables do n8n.
