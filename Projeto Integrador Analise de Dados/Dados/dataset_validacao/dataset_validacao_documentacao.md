# Dataset de Ações de Validação — Fase 1 e Fase 2

## Sobre este documento

Este documento acompanha dois arquivos CSV — `validacao_fase1.csv` e `validacao_fase2.csv` — extraídos da plataforma da Visior. Ele explica o que os dados representam, como o processo de validação funciona e o significado de cada coluna, para que vocês possam aplicar técnicas de ciência de dados sobre eles com segurança.

Os dados foram **anonimizados antes da extração**: nenhuma coluna identifica validadores, contas de clientes, rodovias ou registros internos (lotes, vídeos, detecções) pelo nome ou ID real. Todos esses identificadores foram substituídos por códigos opacos (explicado na seção "Anonimização" abaixo).

## Contexto: o que a Visior faz

A Visior é uma plataforma de processamento de vídeo geolocalizado voltada para rodovias. Motoristas gravam viagens com smartphones fixados no painel; os vídeos são enviados para a nuvem, onde uma IA (modelos de detecção de objetos) identifica defeitos de pavimento (buracos, trincas) e mapeia ativos fixos (placas de sinalização). O resultado alimenta um "Gêmeo Digital" — um mapa interativo da rodovia — usado pelas concessionárias.

Como toda IA comete erros (falsos positivos, classificação errada, falta de contexto geográfico), existe uma etapa de **validação humana** entre a detecção bruta da IA e a publicação da informação no mapa. É exatamente essa etapa — as ações que os validadores humanos realizam — que este dataset descreve.

## O pipeline de validação (Fase 1 e Fase 2)

Cada vídeo gravado gera uma ou mais **detecções** (recortes de imagem onde a IA acredita ter encontrado algo relevante). Essas detecções são agrupadas em **lotes** (`jobs`) para revisão humana, em duas etapas sequenciais:

### Fase 1 — Mosaico (triagem de classe)

Um mosaico com dezenas de recortes de imagem (crops) é apresentado ao validador, sem o contexto do vídeo completo. O objetivo é rápido: descartar falsos positivos óbvios e corrigir erros grosseiros de classificação, além de marcar se o objeto tem algum defeito físico visível (placa amassada, torta, suja, quebrada, incompleta, caída, girada incorretamente). Alguns modelos de IA (defeitos de pavimento, por exemplo) pulam essa fase inteiramente por já não terem contexto suficiente num recorte isolado.

Nem todo recorte recebe uma ação explícita do validador — se ele não faz nada com um recorte específico (não rejeita, não muda a classe), esse recorte é considerado aprovado tacitamente quando o lote inteiro é fechado. O dataset representa esse caso como uma linha própria (veja `action_type = approved_implicit` abaixo).

### Fase 2 — Contexto (associação ao Gêmeo Digital)

Depois da Fase 1, cada detecção é revisada com o vídeo completo, contexto geográfico e um radar mostrando ativos e ocorrências próximas já cadastrados. Aqui o validador decide se a detecção é válida, e — quando aplicável — a associa a um ativo de inventário existente (uma placa já cadastrada) ou a uma ocorrência já aberta (um buraco já registrado), o que efetivamente atualiza o Gêmeo Digital. É também nesta fase que casos de "ativo ausente" são resolvidos: quando a IA passa por onde uma placa deveria estar e não a encontra, um card especial é gerado para o validador confirmar se o ativo realmente sumiu (furto, queda) ou se foi um erro de leitura.

Cada validador vota de forma independente; o sistema consolida um consenso (maioria de votos, com validadores seniores tendo poder de desempate) para decidir o estado final da detecção — mas o que este dataset registra é o **voto individual de cada validador**, não apenas o resultado consolidado.

## Uma limitação importante: isto é a decisão final, não um clickstream

O sistema registra, para cada (validador, card, fase), a **decisão mais recente** daquele validador — não uma sequência de cliques. Se um validador muda de ideia várias vezes no mesmo card antes de seguir em frente, apenas o estado final fica registrado (com o timestamp da última alteração). Ou seja: dá para calcular perfeitamente coisas como tempo médio até a decisão final, horário do dia com mais atividade, e quais classes são mais corrigidas — mas não dá para reconstruir "quantas vezes a pessoa hesitou" num card específico.

## Anonimização

| Tipo de dado | Como foi anonimizado |
|---|---|
| Validador (`validator_id`) | Código sequencial opaco, sem relação com nome, e-mail ou qualquer identificador real. |
| Conta/cliente (`account_id`) | Código opaco de 6 caracteres. Duas linhas com o mesmo `account_id` pertencem à mesma conta; o código em si não permite descobrir qual conta é. |
| Rodovia (`road_id`) | Mesmo esquema do `account_id` — permite agrupar por rodovia sem revelar qual rodovia é. |
| Lote / vídeo / detecção / ativo (`job_uuid`, `segment_uuid`, `detection_uuid`, `matched_entity_uuid`) | Códigos sequenciais opacos (prefixos `J`, `S`, `D`, `M`). Servem apenas para agrupar linhas que se referem ao mesmo lote, vídeo, card ou ativo/ocorrência — não são identificadores reais nem seguem nenhuma ordem cronológica ou de importância. |

Em todos os casos, **o mesmo registro real sempre recebe o mesmo código** em todo o dataset (é seguro agrupar, contar ou cruzar por essas colunas).

## Dicionário de colunas — `validacao_fase1.csv`

| Coluna | Descrição |
|---|---|
| `action_id` | Identificador único da linha (não carrega significado além de ser único). |
| `action_type` | O que o validador fez. Valores possíveis: `rejected` (descartou o recorte como falso positivo), `class_changed` (corrigiu a classe sugerida pela IA), `defect_flag_only` (só marcou/desmarcou o defeito físico, sem mexer na classe), `approved_implicit` (não fez nenhuma ação — o recorte foi aprovado tacitamente ao fechar o lote). |
| `explicit_vote` | `true` se o validador realmente interagiu com esse card nesta fase; `false` apenas na aprovação tácita (`approved_implicit`). |
| `account_id` | Código anonimizado da conta/cliente dona do vídeo. |
| `validator_id` | Código anonimizado do validador que realizou a ação. |
| `road_id` | Código anonimizado da rodovia (pode vir vazio se o vídeo ainda não tinha rodovia associada). |
| `job_uuid` | Código anonimizado do lote (mosaico) ao qual este card pertence. |
| `segment_uuid` | Código anonimizado do vídeo de origem (cada vídeo tem ~5 minutos de gravação). |
| `detection_uuid` | Código anonimizado do card/detecção específico. |
| `taxonomy_class` | Classe final do card no momento da extração (já refletindo eventuais correções de classe). |
| `original_class` | Classe que a IA originalmente atribuiu ao card, preservada mesmo que ela tenha sido corrigida depois. Comparar `original_class` com `taxonomy_class` é o caminho direto para medir em que classes o modelo mais erra. |
| `class_changed_to` | Preenchida somente quando `action_type = class_changed`; repete a nova classe atribuída (mesmo valor que aparecerá em `taxonomy_class`). |
| `has_defect` | Se o validador marcou o objeto como fisicamente defeituoso (amassado, quebrado, sujo, etc.) nesta ação. |
| `inference_model_name` | Nome do modelo de IA que gerou a detecção original. |
| `validation_flow` | Fluxo de validação do lote: `full` (fluxo completo padrão) ou `simplified_vertical` (fluxo simplificado usado para alguns lotes de placas verticais). |
| `job_cards_count` | Quantidade total de cards (detecções) que existiam nesse lote — o "tamanho" do mosaico que o validador enfrentou. |
| `action_timestamp` | Data/hora da ação. Para `approved_implicit`, todas as linhas do mesmo lote compartilham o mesmo horário (o momento em que o lote foi fechado), já que o sistema não registra um timestamp individual para cards não tocados. |

## Dicionário de colunas — `validacao_fase2.csv`

| Coluna | Descrição |
|---|---|
| `action_id` | Identificador único da linha. |
| `action_type` | Decisão do validador. Valores possíveis: `verified` (confirmou a detecção como válida), `rejected` (marcou como falso positivo), `ignored` (tecnicamente correta, mas irrelevante — ex.: placa temporária de obra, fora da faixa de domínio), `duplicate` (mesmo objeto detectado mais de uma vez por erro de rastreamento), `occurrence_resolved` ("dar baixa" — o card em si é um falso positivo, mas serve de prova de que uma ocorrência aberta anteriormente foi reparada). |
| `account_id` | Código anonimizado da conta/cliente. |
| `validator_id` | Código anonimizado do validador. |
| `road_id` | Código anonimizado da rodovia. |
| `job_uuid` | Código anonimizado do lote. |
| `segment_uuid` | Código anonimizado do vídeo de origem. |
| `detection_uuid` | Código anonimizado do card/detecção. |
| `taxonomy_class` | Classe final do card no momento da extração. |
| `original_class` | Classe originalmente atribuída pela IA (ver explicação na Fase 1). |
| `has_defect` | Confirmação de defeito físico do ativo nesta fase (para placas, por exemplo, é aqui que se confirma se a placa está danificada). |
| `inference_model_name` | Nome do modelo de IA que gerou a detecção. |
| `validation_flow` | `full` ou `simplified_vertical` (ver Fase 1). |
| `job_cards_count` | Quantidade de cards do lote ao qual esta detecção pertence. |
| `card_frames_count` | Só é preenchida em cards de "ativo ausente" (`[MISSING]`): quantidade de frames candidatos que o sistema ofereceu ao validador para escolher a melhor evidência. Vazia nos demais casos. |
| `matched_entity_type` | Tipo de entidade do Gêmeo Digital à qual esta detecção foi associada: `inventory` (um ativo permanente, como uma placa) ou `task` (uma ocorrência, como um buraco). Vazio se não houve associação. |
| `matched_entity_uuid` | Código anonimizado da entidade (ativo ou ocorrência) associada. |
| `inventory_off_roadway_current` | Se o ativo associado está atualmente marcado como "fora da via principal" (fora da faixa de domínio monitorada). **Reflete o estado atual do ativo no momento da extração, não necessariamente o estado exato no momento deste voto específico** — o sistema não guarda esse dado por voto individual. |
| `inventory_visible_opposite_direction_current` | Se o ativo associado está atualmente marcado como visível também para quem trafega no sentido oposto ao cadastrado. Mesma ressalva da coluna anterior: é o estado atual, não o estado no momento do voto. |
| `action_timestamp` | Data/hora do voto. |

## Dúvidas

Qualquer dúvida sobre o significado de uma coluna ou sobre como o processo de validação funciona na prática, fico à disposição.
