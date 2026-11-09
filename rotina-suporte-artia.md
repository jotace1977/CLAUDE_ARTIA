# Rotina Suporte → Artia (fase de testes — cliente ARTIA)

Especificação operacional e estado da rotina que lê a caixa `suporte@j2rconsultoria.com.br`,
cria/atualiza atividades no Artia, controla saldo de horas (CF/CT/ABAP/GS) e espelha apontamentos.
**A rotina diária ainda NÃO foi agendada** — execuções são manuais nesta fase.

## Referências no Artia (organização J2R Consultoria 165411)

| Item | ID |
|---|---|
| Grupo de trabalho "02. Suporte em Andamento" | 6681502 |
| Pasta de triagem "01. Chamados Recebidos (Triagem)" | 6760487 |
| Pasta cliente ARTIA (domínio `artia.com`) | 6797865 |
| Projeto "Contrato 1 - TAXONE_Artia (Teste)" | 6797875 |
| Pasta "01-Novembro / 2026" | 6797876 |
| Pasta "02-Dezembro / 2026" | 6797877 |
| Pasta "03-Janeiro / 2027" | 6797878 |
| Responsável padrão: Flávia Rossi (user) | 310191 |
| Campo personalizado "Solicitante" (hash) | d1ae7b3d993d3a867e01089edb5c98b0bf398993 |

Tipos de atividade: CF = 718290 · CT = 718289 · ABAP = 718291 · GS = 718292

Equipes (Etapa 3): Consultor Funcional 16064 (Igor Quirino) · Gerente Suporte 16065 (Roger Munhoz) ·
ABAP 16066 (vazia) · Consultor Técnico 16067 (vazia)

Atividades de saldo em "01-Novembro / 2026":

| Cargo | Atividade | Estimado | Real (09/11) |
|---|---|---|---|
| CF | 34022714 | 6h | 0h |
| CT | 34022715 | 12h | 0h |
| ABAP | 34022717 | 5h | 0h |
| GS | 34022718 | 2h | 0h |

> Observação: as atividades de saldo são identificadas pelo **título** (CF/CT/ABAP/GS); elas não têm
> "Tipo de Atividade" preenchido. As atividades criadas pela rotina recebem o Tipo de Atividade (CF no início).

## Regras implementadas

**Etapa 1**
- Cliente = domínio do remetente → pasta com o nome da empresa no grupo 6681502.
- 1 contrato (projeto) na pasta → cria na subpasta do mês de recebimento; início = dia 1, término = último dia do mês.
- Mais de 1 contrato → cria na triagem 6760487 e notifica "aguarda triagem".
- Resposta (RE:/RES:) de assunto já existente → acrescenta o e-mail à descrição, não cria atividade.
- Título = assunto; Descrição = corpo + rodapé `[Rotina Suporte]` (marcador que identifica atividades da rotina);
  Responsável = Flávia Rossi; Solicitante = nome de exibição do remetente.

**Etapa 2 – saldo por cargo** (cargo = Tipo de Atividade da atividade da rotina)
`saldo = estimado(cargo) − real(cargo) − Σ reservas das outras atividades da rotina do mesmo cargo`
(reserva = estimado − real espelhado da atividade da rotina).
- saldo ≥ 2h → estimativa 2h · 0 < saldo < 2h → estimativa = saldo · saldo ≤ 0 → sem estimativa
  + comentário "CONTRATO NÃO POSSUI SALDO DE HORAS PARA EXECUÇÃO DA ATIVIDADE." (e na notificação).
- Estimativa alterada manualmente (≠ 2h) → passa a reservar o novo valor.
- As atividades de saldo nunca têm o esforço estimado alterado.

**Etapa 3 – espelhamento de apontamentos**
- Apontamento em atividade da rotina → identifica a equipe do autor → duplica (data, início, duração,
  observação) na atividade de saldo do cargo, limitado ao que falta para o real chegar ao estimado.

## Histórico de execuções

### 09/11/2026 (manual)
- Último e-mail processado: recebido em 2026-11-09T13:39:22Z (processar a partir deste horário na próxima execução).
- Criada atividade 34023217 "Erro de parâmetro Taxone (Teste rotina claude)" em "01-Novembro / 2026",
  tipo CF, 2h estimadas (saldo CF 6h → 4h livres após a reserva), solicitante Igor de Paula Quirino.
- Notificação Teams: **não enviada** — o host do webhook do Power Automate está bloqueado pela política de rede do ambiente.
- Etapa 3: nenhum apontamento nas atividades da rotina até o momento.
- E-mails de outros clientes (Logicalis, FIAP/Alura) e o informativo interno foram ignorados nesta fase de teste.
