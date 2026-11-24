# Rotina HubSpot → Artia: projetos dos negócios "Fechado ganho"

Especificação operacional da rotina que roda **todo dia às 01h47 (horário de Brasília)** e cria no Artia
um projeto para cada novo negócio ganho no HubSpot. Esta é a fonte de verdade da rotina: a sessão agendada
lê este arquivo e executa os passos abaixo.

## Referências

**HubSpot (portal 48517337)**
- Pipeline "Pipeline de vendas" = `default`. Etapa "Fechado ganho" = `dealstage = closedlost`
  (o id interno é `closedlost`, mas o rótulo no portal é "Fechado ganho"; "Fechado perdido" é `1006939286`).
- Data de entrada na etapa: `hs_v2_date_entered_closedlost`.
- **Corte:** só considerar negócios com `hs_v2_date_entered_closedlost >= 2026-11-16`.

| Campo (rótulo) | Propriedade | Objeto |
|---|---|---|
| Título do negócio | `dealname` | negócio |
| Oferta (Serviço ou Produto) | `oferta__produto_` | negócio |
| Tipo (aba financeira) | `tipo__sugar_` (BPO, Subscrição, Suporte, Time Material, Turn Key) | negócio |
| Nº Proposta J2R | `no_proposta_j2r` | negócio |
| Data início contrato | `data_inicio_contrato_suporte_bpo_outsourcing` | negócio |
| Data final do contrato | `data_final_do_contrato_suporte_bpo_outsourcing` | negócio |
| Valor | `amount` | negócio |
| Data de fechamento | `closedate` | negócio |
| Tanque (Horas Projetadas) | `tanque` | negócio |
| CF1 / CF2 - Consultor Funcional | `cf1__consultor_funcional` / `cf2__consultor_funcional` | negócio |
| CT - Consultor Técnico TAX One | `ct__consultor_tecnico_tax_one` | negócio |
| CTA - Consultor Técnico ABAP | `cta__consultor_tecnico_abap` | negócio |
| GP - Gerente de Projetos | `gp__gerente_de_projetos` | negócio |
| GS - Gerente de Serviços | `gs__gerente_de_servicos` | negócio |
| Empresa (nome) | `name` | empresa associada |
| Tipo da empresa | `type` ("Cliente base" = rótulo "Cliente base - J2R") | empresa associada |
| ERP | `erp` (empresa; se vazio, usar `erp__empresas` do negócio) | empresa associada |

**Artia (organização J2R Consultoria 165411)**

| Item | ID |
|---|---|
| Grupo "01. Projetos em Andamento" | 6671087 |
| Grupo "02. Suporte em Andamento" | 6681502 |
| Campo de projeto "ERP" (hash) | da03fcbb82650fd76644b717d47a8f6def18fde4 |
| Campo de projeto "Horas Orçadas" (hash) | 3133e13f2a120b8279b0cacf043821b47324c306 |
| Campo de projeto "Tanque de Horas?" (Sim/Não) (hash) | a255a604d540d570ce325cf12566e28484141b3c |

Tipos de projeto: TM 707853 · TK 708502 · BPO 708508 · Suporte Tax One 708505 · Suporte DFe 708506 ·
Projeto de Implantação 714182 · Alocações Outsourcing 708504 · Desenv. Fábrica Software 708503.

## Passo a passo

1. **Buscar negócios**: `pipeline = default`, `dealstage = closedlost`, `hs_v2_date_entered_closedlost >= 2026-11-16`
   (paginando até o fim). Para cada um, buscar a empresa associada (nome, `type`, `erp`).
2. **Classificar pela Oferta** (comparação sem acento e sem maiúsculas, usando `oferta__produto_`):
   - contém "suporte" ou "bpo" → **Suporte** (grupo 6681502);
   - senão, contém "implantacao" → **Projetos** (grupo 6671087);
   - senão → ignorar (ex.: Subscrição, Desenvolvimento, Assessment, Outsourcing).
3. **Duplicados** — pular o negócio se:
   - já existe projeto no Artia (nos dois grupos) cuja descrição contém `[HubSpot deal <id>]`; ou
   - já existe projeto em qualquer um dos dois grupos cujo nome contém o mesmo Nº de proposta
     (normalizado, ex.: `PS_182_2026`); ou
   - outro negócio da mesma execução, com o mesmo Nº de proposta, já gerou projeto (ex.: os dois negócios
     PS_182_2026 da QUALICORP geram um único projeto — fica o de criação mais antiga).
4. **Pendências** (não cria, reporta e tenta de novo na próxima madrugada):
   - sem empresa associada; sem Nº de proposta;
   - grupo Suporte sem Data início ou Data final do contrato.
5. **Cliente no Artia**: procurar cliente pelo nome da empresa (sem acento/caixa; aceitar quando um nome contém o
   outro). Se não existir, criar com o nome da empresa do HubSpot.
6. **Pasta do cliente** (filha direta do grupo):
   - procurar pasta cujo nome (normalizado, sem "LTDA", "S.A.", "S/A", pontuação) contenha ou esteja contido no nome
     da empresa;
   - se a empresa **não** é "Cliente base", ou se não encontrar pasta: criar pasta com o nome da empresa.
7. **Projeto** (dentro da pasta do cliente):
   - Título: `<Nº Proposta normalizado> - <dealname>`. Nº normalizado = `PS_<número>_<ano>`; se a proposta não
     trouxer o ano, usar o ano do fechamento (ex.: `PS_091_2026 - VIVARA - Implantação DF-e Governance Plus Licença`).
   - Cliente: o cliente do passo 5. Nº do projeto: Nº de proposta normalizado.
   - Tipo de projeto (de `tipo__sugar_`): Turn Key→TK · Time Material→TM · BPO→BPO ·
     Suporte→Suporte DFe se Oferta/título contém "DF", senão Suporte Tax One · Subscrição→sem tipo.
     Se vazio: grupo Projetos→Projeto de Implantação; grupo Suporte→BPO se a Oferta contém "BPO", senão
     Suporte DFe/Suporte Tax One pela mesma regra do "DF".
   - ERP: mapear o ERP da empresa para as opções do Artia (tabela abaixo); se não houver correspondência, não preencher
     e reportar.
   - Tanque de Horas?: "Sim" se a Oferta ou o título contém "tanque de horas" ou se `tanque` > 0; senão "Não".
   - Descrição: `Criado automaticamente a partir do HubSpot. Oferta: <oferta>. [HubSpot deal <id>]`.
   - **Grupo Suporte**: Início estimado = Data início contrato; Término estimado = Data final do contrato.
   - **Grupo Projetos**: Horas Orçadas = CF1 + CF2 + CT + CTA + GP + GS (+ Tanque, se preenchido).
8. **Lançamento financeiro**: receita realizada (`kind = accomplished`, `classification = revenue`) no projeto,
   título "Valor do contrato - <Nº proposta>", data = data de fechamento, valor = `amount`.
   Sem valor → não lança e reporta.
9. **Atividades de cargo** — horas por cargo vindas de Horas Projetadas, só os cargos com horas > 0:
   `CF` = CF1 + CF2 (uma única atividade) · `CT` = CT · `ABAP` = CTA · `GP` = GP · `GS` = GS.
   - **Grupo Suporte (sempre)**: criar uma pasta por mês entre Início e Término estimado, nome `Mês / AA`
     (ex.: "Dezembro / 26"); em cada pasta, uma atividade por cargo com o esforço estimado do cargo (as horas
     são replicadas em todos os meses), início = 1º dia do mês, término = último dia do mês.
     Os títulos CF/CT/ABAP/GS são os mesmos que a rotina Suporte→Artia usa como atividades de saldo.
   - **Grupo Projetos**: só quando "Tanque de Horas?" = Sim, criar no projeto uma atividade por cargo com o esforço.
10. **Relatório** da execução: projetos criados (cliente, grupo, título), duplicados ignorados, pendências.
    Registrar a execução no Histórico abaixo (commit na branch da rotina).

### Mapa de ERP (HubSpot → Artia)

| HubSpot | Artia |
|---|---|
| SAP 4 HANA | SAP S/4HANA |
| SAP ECC, SAP 6.0, SAP ECC R/3 6.0 | SAP ECC 6.0 |
| SAP B1 | SAP B1 |
| ORACLE - EBS, Oracle (ORACLE - 12) | Oracle EBS r12 |
| ORACLE - CLOUD, ORACLE - FUSION - 13, FUSION | Oracle Cloud Fusion |
| ORACLE NETSUITE, NET SUITE | Oracle Net Suite |
| PROTHEUS, TOTVS - MICROSIGA, MICROSIGA | TOTVS Protheus |
| TOTVS - DATASUL | TOTVS Datasul |
| MEGA SISTEMAS, MEGA | Mega |
| PRÓPRIO | PRÓPRIO |
| DYNAMICS AX | AX Dynamics 2012 |

## Histórico de execuções

### 24/11/2026 (manual, 1ª execução)
- HubSpot: datas do contrato do negócio "Loja Elétrica - Contrato Suporte Tax One e Módulos RT" preenchidas a pedido
  (início 01/12/2026, fim 31/05/2027).
- Negócios avaliados (entrada em "Fechado ganho" desde 16/11/2026): 6.
- **Criados**
  - Cliente "Loja Elétrica Ltda." (70633) e pastas "Loja Elétrica Ltda." em 01. Projetos (6812125) e 02. Suporte (6812126)
    — a empresa é "Cliente base", mas não havia pasta.
  - Projeto 6812127 "PS_198_2026 - Loja Elétrica - Implantação Módulo RT" (Projeto de Implantação, ERP SAP S/4HANA,
    Horas Orçadas 640 = CF 480 + CTA 64 + GP 96, Tanque = Não) + receita realizada R$ 130.519,30 (6604500).
  - Projeto 6812128 "PS_209_2026 - Loja Elétrica - Contrato Suporte Tax One e Módulos RT" (Suporte Tax One, ERP SAP S/4HANA,
    01/12/2026 → 31/05/2027, Tanque = Não) + receita realizada R$ 51.000,00 (6604501).
    Pastas Dezembro / 26 (6812129), Janeiro / 27 (6812131), Fevereiro / 27 (6812132), Março / 27 (6812133),
    Abril / 27 (6812134), Maio / 27 (6812135), cada uma com CF 300h e GS 48h (dia 1 → último dia do mês).
- **Duplicados ignorados**: QUALICORP PS_148_2026 (projeto 6759376) e PS_090_2026 (projetos 6759373/6759374/6759375).
- **Pendente**: PS_182_2026 — "Consultoria DERE - (BPO)" (65162328482) e "QUALICORP - Consultoria/BPO - DeRE 1º evento"
  (68559093927) são o mesmo contrato (gerarão 1 projeto no grupo Suporte), mas estão sem Data início/final do contrato.
