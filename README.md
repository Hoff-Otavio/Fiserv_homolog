# Roteiro de Certificação — Fiserv e-SiTef (Cartão + PIX)

> Ambiente de homologação: `https://esitef-homologação.softwareexpress.com.br/e-sitef/api/`
> Data de início: 2026-04-10

---

## Cartões de Teste

| Bandeira | Número | Vencimento | CVV | CPF |
|---|---|---|---|---|
| VISA | `4000 0000 0000 0044` | mês/ano atual | 123 | — |
| VISA (19 dígitos) | `4563 4700 0000 0000 004` | mês/ano atual | 123 | — |
| MASTERCARD | `5000 0000 0000 0001` | mês/ano atual | 123 | — |
| MASTERCARD (19 dígitos) | `5390 0000 0000 0000 009` | mês/ano atual | 123 | — |
| AMEX | `3764 0000 0000 016` | mês/ano atual | 123 | — |
| HIPERCARD | `6062 8200 0000 001` | mês/ano atual | 123 | — |
| DINERS | `3634 5600 0000 01` | mês/ano atual | 123 | — |
| ELO | `6362 9700 0045 7013` | mês/ano atual | 123 | — |
| SODEXO Cultura | `6060 7001 2476 5016` | mês/ano atual | 123 | 581.935.914-37 |
| SODEXO Alimentação | `6033 8900 0000 0007` | mês/ano atual | 123 | 581.935.914-37 |
| SODEXO Refeição | `6060 7133 3914 2012` | mês/ano atual | 123 | 581.935.914-37 |

> **OBS:** Todos os cartões usam **vencimento = mês e ano atuais** e **CVV = 123**

---

## Alertas do Módulo

>  **getStatus — Retry de 3 tentativas NÃO implementado**
>
> A documentação da Fiserv exige:
> - Timeout de 60 segundos aguardando resposta
> - Após timeout: chamar `getStatus` (até **3 tentativas**)
> - Após 3 falhas consecutivas: **bloquear nova tentativa de pagamento** do mesmo pedido
>
> **O que o módulo faz hoje:** O `CronJob/CheckPaymentStatus.php` chama `updateStatusOrderById()` **uma única vez** por pedido, sem loop de retentativas, sem contagem de tentativas e sem bloqueio de reenvio.
>
> **Arquivo afetado:** `Core/CronJob/CheckPaymentStatus.php` e `Core/Model/Notifications/Topics/Payment.php`
>
> **Ação necessária antes da certificação:** Implementar loop com máximo de 3 tentativas e bloqueio de novo pagamento em caso de falha total.

---

## Legenda

- `[ ]` — Não testado
- `[x]` — Aprovado
- `[!]` — Falhou / problema encontrado
- `[~]` — Testado parcialmente / com ressalvas

---

## BLOCO 1 — Pagamento com Cartão (Obrigatório)

### 1.1 Pagamento à vista — APROVADO

| # | Bandeira | Valor | NSU e-SiTef | Order ID | Merchant USN | Status | Data |
|---|---|---|---|---|---|---|---|
| 1 | VISA | R$ 250,00 | 260410138563214 | 000000001 | 1 | `[x]` | 10/04/2026 |
| 2 | MASTERCARD | R$ 250,00 | 260410138564154 | 000000004 | 4 | `[!]` | 10/04/2026 |
| 3 | AMEX | R$ 250,00 | — | 000000005/006 | — | `[!]` | 10/04/2026 |
| 4 | DINERS | R$ 250,00 | 260410138565254 | 000000007 | 7 | `[x]` | 10/04/2026 |

**Fluxo esperado:** Pagamento processado → pedido com status "Aprovado" → fatura gerada automaticamente.

>  **Observação:** O módulo está usando fluxo de pré-autorização (`POST /v1/preauthorizations`) mesmo para pagamento à vista. O comprovante retornado traz `PRE-AUT. SIM.`. Verificar se a Fiserv aceita esse fluxo na certificação ou se é necessário usar `POST /v1/payments` direto.

---

### 1.2 Pagamento com status NEGADO (Obrigatório)

| # | Bandeira | Valor | NSU e-SiTef | Order ID | Status | Data |
|---|---|---|---|---|---|---|
| 5 | VISA | R$ 22.000,00 (2200000 centavos) | 260410138566284 | 000000009 | `[x]` | 10/04/2026 |

**Fluxo esperado:** Pagamento recusado pela autorizadora → pedido com status "Negado" → sem fatura.

---

### 1.3 Cancelamento / Estorno com Cartão (Obrigatório)

| # | Descrição | Valor | NSU e-SiTef | Order ID | Status | Data |
|---|---|---|---|---|---|---|
| 6 | Estorno de pagamento aprovado | R$ 250,00 | — | 000000001 | `[!]` | 10/04/2026 |

**Fluxo esperado:** Pedido aprovado → solicitar estorno → API recebe cancel → pedido muda para "Estornado" / "Cancelado".

---

### 1.4 Pagamento Parcelado (Opcional)

| # | Descrição | Parcelas | Valor | NSU e-SiTef | Order ID | Status | Data |
|---|---|---|---|---|---|---|---|
| 7 | Parcelado **sem juros** | 3x | — | | | `[ ]` | |
| 8 | Parcelado **com juros** | 3x | — | | | `[ ]` | |

---

### 1.5 Pré-autorização e Captura (Opcional)

| # | Descrição | Valor | NSU e-SiTef | Order ID | Status | Data |
|---|---|---|---|---|---|---|
| 9 | Pré-autorização | R$ 10,00 | | | `[ ]` | |
| 10 | Captura da pré-autorização | R$ 10,00 | | | `[ ]` | |

**OBS:** Captura é **obrigatória** se pré-autorização for certificada.

---

## BLOCO 2 — Consulta de Status / Timeout (Obrigatório)

>  **Atenção:** Ver alerta acima sobre implementação incompleta do retry.

| # | Descrição | Valor | NSU e-SiTef | Order ID | Status | Data |
|---|---|---|---|---|---|---|
| 11 | Timeout + getStatus → transação **APROVADA** | R$ 10,00 | | | `[ ]` | |
| 12 | Timeout + getStatus → transação **NEGADA** | R$ 22.000,00 (2200000 centavos) | | | `[ ]` | |

**Fluxo esperado (caso 11):**
1. Transação enviada → sem resposta (timeout de 60s)
2. Aplicação chama `getStatus` (até 3x)
3. Resposta indica APROVADA → pedido atualizado, fatura gerada
4. Cliente não consegue enviar novo pagamento para o mesmo pedido

**Fluxo esperado (caso 12):**
1. Transação enviada → sem resposta (timeout de 60s)
2. Aplicação chama `getStatus` (até 3x)
3. Resposta indica NEGADA → pedido atualizado, sem fatura
4. Cliente não consegue enviar novo pagamento para o mesmo pedido

---

## BLOCO 3 — PIX (Obrigatório)

### 3.1 PIX Confirmado

| # | Descrição | Valor | NSU e-SiTef | Order ID | Status | Data |
|---|---|---|---|---|---|---|
| 13 | PIX **confirmado** | R$ 10,00 (1000 centavos) | 260410138567540 | 000000011 | `[x]` | 10/04/2026 |

**Fluxo esperado:** QR Code gerado → pagamento PIX realizado → webhook de notificação recebido → pedido aprovado → fatura gerada.

> **Adendo:** Confirmação realizada via painel admin de homologação. Status `CON` detectado automaticamente pelo cron `checkPaymentStatus`.

---

### 3.2 PIX Negado

| # | Descrição | Valor | NSU e-SiTef | Order ID | Status | Data |
|---|---|---|---|---|---|---|
| 14 | PIX **negado na autorizadora** | R$ 5,00 (500 centavos) | 260410138569670 | 000000013 | `[!]` | 10/04/2026 |

**Fluxo esperado:** QR Code gerado → autorizadora nega → pedido com status "Negado".

---

### 3.3 PIX Erro

| # | Descrição | Valor | NSU e-SiTef | Order ID | Status | Data |
|---|---|---|---|---|---|---|
| 15 | PIX **erro** | R$ 6,00 (600 centavos) | 260410138570710 | 000000015 | `[~]` | 10/04/2026 |

**Fluxo esperado:** QR Code gerado → erro no processamento → pedido com status de erro.

> **Adendo:** QR Code gerado com sucesso, requisição aceita (status `PEN`). O sandbox retorna `PEN` para qualquer valor de PIX — a simulação de "erro" requer ação no painel da Fiserv, que não está disponível neste ambiente.

---

### 3.4 Estorno PIX (Obrigatório)

| # | Descrição | Valor | NSU e-SiTef | Order ID | Status | Data |
|---|---|---|---|---|---|---|
| 16 | Estorno de pagamento PIX aprovado | — | — | — | `[~]` | 10/04/2026 |

**Fluxo esperado:** PIX aprovado → solicitar estorno → API processa devolução → pedido com status "Estornado".

> **Adendo:** Bloqueado no ambiente de homologação. O método `capture()` do módulo (`Core/Model/Custom/Payment.php:437`) só permite faturamento online quando `paymentResponse.status == 'CON'`. Como o sandbox da Fiserv retorna `PEN` para todos os PIX e não há acesso ao painel para confirmar, o PIX nunca chega ao estado necessário para o estorno via admin. Comportamento esperado em produção (com webhook real de confirmação).

---

## BLOCO 4 — Tokenização (Opcional)

| # | Descrição | NSU e-SiTef | Order ID | Status | Data |
|---|---|---|---|---|---|
| 17 | Armazenamento de cartão (tokenização) | | | `[ ]` | |
| 18 | Pagamento com cartão armazenado (token) | | | `[ ]` | |

---

## BLOCO 5 — 3DS 2.0 (Obrigatório para Débito)

### Fluxo de Autenticação 3DS

| # | Status Esperado | Valor (centavos) | Trans ID | Data | Status |
|---|---|---|---|---|---|
| 19 | **CON** (Autenticado) | 10000 | | | `[ ]` |
| 20 | **Challenge** (Desafio) | 10004 | | | `[ ]` |
| 21 | **NEG** (Negado) | 10001 | | | `[ ]` |

### Efetivação com dados 3DS

| # | Descrição | NSU e-SiTef | Valor | Order ID | Status | Data |
|---|---|---|---|---|---|---|
| 22 | Pagamento enviando `eci`, `reference_id`, `cavv` da autenticação 3DS | | | | `[ ]` | |

**Payload obrigatório na efetivação:**
```json
"external_authentication": {
    "eci": "xxx",
    "reference_id": "string",
    "cavv": "string"
}
```

---

## Evidências Obrigatórias

- [ ] Prints do fluxo completo de pagamento (todas as telas)
- [ ] Prints do fluxo de Consulta de Status — transação **Aprovada**
- [ ] Prints do fluxo de Consulta de Status — transação **Negada**

---

## Progresso Geral

| Bloco | Total | Concluídos | Parciais | Falhos | Pendentes |
|---|---|---|---|---|---|
| 1 — Cartão (obrigatórios) | 6 | 3 | 0 | 2 | 1 |
| 2 — Timeout / getStatus | 2 | 0 | 0 | 0 | 2 |
| 3 — PIX | 4 | 1 | 2 | 1 | 0 |
| 4 — Tokenização | 2 | 0 | 0 | 0 | 2 |
| 5 — 3DS 2.0 | 4 | 0 | 0 | 0 | 4 |
| **Total** | **18** | **4** | **2** | **3** | **9** |

---

## Resumo da Sessão de Testes — 10/04/2026

### O que passou
| # | Teste | Order | e-SiTef USN |
|---|---|---|---|
| 1 | Pagamento à vista VISA | 000000001 | 260410138563214 |
| 4 | Pagamento à vista DINERS | 000000007 | 260410138565254 |
| 5 | Pagamento NEGADO (VISA R$ 22.000,00) | 000000009 | 260410138566284 |
| 13 | PIX confirmado (R$ 10,00) | 000000011 | 260410138567540 |

### O que falhou
| # | Teste | Order | Motivo | Bug |
|---|---|---|---|---|
| 2 | Pagamento à vista MASTERCARD | 000000004 | Rede — "Operacao nao permitida" (code 134) | BUG-002 |
| 3 | Pagamento à vista AMEX | 000000005/006 | authorizer_id "3" inválido (code 5) | BUG-004 |
| 6 | Estorno cartão VISA | 000000001 | "Authenticity error" — NSU errado no cancel | BUG-005 |
| 14 | PIX negado | 000000013 | Cancel rejeitado pelo gateway (PIX ainda PEN) | BUG-006 |

### O que ficou parcial
| # | Teste | Order | Motivo |
|---|---|---|---|
| 15 | PIX erro (R$ 6,00) | 000000015 | QR Code gerado (PEN), simulação de erro requer painel Fiserv |
| 16 | Estorno PIX | — | Bloqueado: PIX não chega a CON sem painel Fiserv |

### O que não foi testado 
- Timeout / getStatus (Blocos 2)
- Tokenização (Bloco 4)
- 3DS 2.0 (Bloco 5)

### Bugs a corrigir antes da certificação
| Bug | Descrição | Severidade |
|---|---|---|
| BUG-001 | `installment_type: "4"` hardcoded para crédito | Alta |
| BUG-002 | MASTERCARD rejeitada pelo adquirente REDE | Alta |
| BUG-003 | Pagamento à vista usa fluxo de pré-autorização | Média |
| BUG-004 | AMEX com `authorizer_id: "3"` inválido | Alta |
| BUG-005 | Estorno de cartão usa NSU da pré-autorização em vez da captura | Alta |
| BUG-006 | Cancel/estorno de PIX pendente falha no gateway | Alta |
| — | getStatus sem retry de 3 tentativas (pendente implementação) | Alta |

---

## Erros Encontrados

### BUG-001 — `installment_type: "4"` hardcoded para crédito
- **Severidade:** Alta
- **Arquivo:** `Core/Model/Custom/Payment.php` linhas 545, 627, 769
- **Descrição:** O campo `installment_type` está fixado com valor `"4"` para todos os pagamentos com cartão de crédito. Na API Fiserv e-SiTef, `"4"` corresponde a PIX/pré-datado, não crédito à vista. O adquirente Bin aceitou no sandbox (permissivo), mas o REDE rejeitou.
- **Impacto:** Pagamentos via adquirentes mais restritivos (Rede, Cielo) podem ser recusados.
- **Correção:** Mapear corretamente o `installment_type` por tipo de transação (`1` = à vista crédito, `2` = parcelado emissor, `3` = parcelado lojista).

---

### BUG-002 — MASTERCARD rejeitada pelo adquirente REDE (homologação)
- **Severidade:** Alta
- **Teste:** #2 — Pagamento à vista MASTERCARD
- **Order ID:** 000000004 | **e-SiTef USN:** 260410138564154
- **Descrição:** MASTERCARD é roteada para `authorizer_id: "2"` (REDE). A REDE retornou `code 134 — "Error on card query"` com `authorizer_code: 255 — "Operacao nao permitida"`.
- **Causa provável:** Conta de homologação não tem MASTERCARD habilitado via Rede, ou `authorizer_id` incorreto para o ambiente de teste. VISA (`authorizer_id: "1"` → Bin) funcionou normalmente.
- **Correção:** Verificar junto à Fiserv qual `authorizer_id` deve ser usado para MASTERCARD no ambiente de homologação, ou habilitar MASTERCARD na conta Rede de teste.

---


### BUG-003 — Fluxo de pagamento à vista usa pré-autorização
- **Severidade:** Média
- **Arquivo:** `Core/Model/Custom/Payment.php`
- **Descrição:** Pagamentos à vista estão sendo processados via `POST /v1/preauthorizations` com `transaction_type: "preauthorization"`, em vez de `POST /v1/payments`. O comprovante retorna `PRE-AUT. SIM.`.
- **Impacto:** Pode ser questionado pela Fiserv na certificação, pois o roteiro distingue pagamento à vista de pré-autorização como casos separados.
- **Correção:** Avaliar se o fluxo de pagamento configurado como "authorize" no admin deve usar `/v1/payments` direto para à vista.

---

### BUG-004 — AMEX com `authorizer_id: "3"` inválido para este merchant
- **Severidade:** Alta
- **Teste:** #3 — Pagamento à vista AMEX
- **Order IDs:** 000000005, 000000006
- **Descrição:** AMEX é mapeada para `authorizer_id: "3"` no código, porém a API retornou `code 5 — "invalid authorizerId value"`. O ID não existe para este merchant de homologação.
- **Padrão identificado:** Apenas `authorizer_id: "1"` (Bin) funcionou até agora. MASTERCARD (`"2"`) e AMEX (`"3"`) falharam por motivos diferentes — Rede recusou a operação, AMEX sequer tem o ID habilitado.
- **Correção:** Os `authorizer_id` precisam ser configuráveis por bandeira no admin, ou alinhados com a Fiserv quais IDs estão disponíveis nessa conta de homologação.

---


### BUG-005 — Estorno usa NSU da pré-autorização em vez da captura
- **Severidade:** Alta
- **Teste:** #6 — Cancelamento / Estorno
- **Order ID:** 000000001
- **Descrição:** Ao emitir memorando de crédito, o módulo disparou corretamente a captura da pré-autorização (esitef_usn: `260410138566454`), mas enviou o cancel usando o NSU da **pré-autorização original** (`260410138563214`). A API retornou `code 115 — "Authenticity error"` porque o NSU não corresponde a uma transação cancelável naquele estado.
- **Arquivo afetado:** `Core/Model/Custom/Payment.php` método `refund()` e `Core/Observer/RefundObserverBeforeSave.php`
- **Correção:** O estorno deve usar o `esitef_usn` da **captura** (armazenado na transação de capture), não o da pré-autorização inicial.

---

### BUG-006 — Cancelamento/estorno de PIX falha com "Authenticity error"
- **Severidade:** Alta
- **Testes:** #8 — PIX Negado (orders 000000012, 000000013)
- **Descrição:** Dois comportamentos distintos identificados:
  - **Order 000000012:** `OrderCancelPlugin` bloqueou o cancel no gateway porque `canCreditmemo()` retorna `false` (sem invoice). O pedido foi cancelado só no Magento.
  - **Order 000000013:** O cancel foi enviado ao gateway com o NSU correto (`260410138569670`), mas a API retornou `code 115 — "Authenticity error"`. Causa: o PIX foi aceito apenas no Magento (via invoice manual), mas o gateway ainda registrava a transação como `PEN`. A API Fiserv rejeita o cancel de transações não confirmadas no gateway.
- **Arquivo afetado:** `Core/Plugin/OrderCancelPlugin.php` e `Core/Model/Core.php::doCancel()`
- **Correção:** O fluxo de cancelamento de PIX precisa: (1) verificar o status real no gateway antes de tentar cancelar; (2) para PIX em `PEN`, usar o endpoint correto de cancelamento de PIX pendente (diferente do cancel de crédito aprovado).

---

