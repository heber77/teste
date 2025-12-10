# Relatório Técnico: Validação dos Conceitos SINKT (Extração + Estratégias)

**Atividade:** SINKT - Sistema de Extração e Validação de Conceitos para Grafos de Conhecimento

---

## 1. Descrição das Técnicas Testadas

### 1.1 Notebook Baseline (SINKT_Baseline.ipynb)

Nesse notebook implementei um baseline, uma abordagem completa conforme especificado na atividade, porém aplicada apenas aos capítulos 1 e 2 do material PDF devido a limitações de escalabilidade.

**Técnicas implementadas:**

**a) Prompting Estruturado:**
- Utilização de prompts JSON com formato obrigatório definido
- Instruções claras para extração de conceitos técnicos (8-12 por capítulo)
- Definição explícita dos 5 tipos de relação: `definition`, `prerequisite`, `property`, `part-of`, `including`
- Limite de 3-5 relações por extração

**b) Validação Cruzada entre Múltiplas LLMs:**
- Implementação com 4 modelos:
  - GPT-3.5 (gpt-3.5-turbo)
  - GPT-4.1 (gpt-4.1 - usei o 4.1 porque a openai está descontinuando o 4.5 e na própria docs recomendam usar o 4.1 como substituto)
  - GPT-5.1 (gpt-5.1)
  - Claude-4.5-Opus (claude-opus-4-5-20251101)
- Fluxo bidirecional: Modelo A gera → Modelo B valida → Modelo B gera → Modelo A valida
- Matriz completa de 6 pares de validação cruzada

**c) Sistema Multi-Agente com Debate Estruturado (9 agentes):**
- 3 Agentes Proponentes (GPT-4.1, GPT-5.1, Claude-4.5-Opus)
- 3 Agentes Críticos (GPT-4.1, GPT-5.1, Claude-4.5-Opus)
- 1 Agente Moderador (GPT-5.1)
- 1 Agente de Consenso (GPT-5.1)
- 1 Agente Auditor (Claude-4.5-Opus)

**d) Extração Iterativa:**
- Cada proponente gera independentemente
- Críticos avaliam cada proposta de cada proponente (matriz 3x3)
- Moderador consolida todas as propostas e críticas
- Consenso gera versão final
- Auditor valida o processo completo

---

### 1.2 Notebook Otimizado (SINKT_pipeline_agents.ipynb)

Gerei outra versão do notebook para processar todos os 26 capítulos do material de forma escalável.

**Técnicas implementadas:**

**a) Prompting hierárquico com regras obrigatórias:**
- Cada conceito DEVE ter uma relação `definition` (regra mandatória)
- Normalização automática de tipos de relação não padrão
- Mapeamento de sinônimos para os 5 tipos obrigatórios

**b) Pipeline de 8 agentes especializados:**
1. **Agente Extrator:** Extração inicial de até 12 conceitos
2. **Agente Validador:** Verificação de conformidade e taxa de aprovação
3. **Agente Crítico:** Identificação de problemas específicos
4. **Agente Defensor:** Defesa de conceitos válidos criticados indevidamente
5. **Agente Moderador:** Decisão final sobre cada conceito (manter/corrigir/remover)
6. **Agente de Consenso:** Consolidação da versão final
7. **Agente Auditor:** Verificação do checklist de qualidade
8. **Agente de Grafo:** Análise estrutural do grafo resultante

**c) Processamento em lotes:**
- 5 lotes definidos: (1-5), (6-10), (11-15), (16-20), (21-26)
- Salvamento automático após cada lote
- Pausas entre capítulos para evitar rate limiting

**d) Extração direta com validação iterativa:**
- Modelo único (GPT-4.1) para todos os agentes
- Foco em eficiência e consistência
- Normalização automática de relações

---

## 2. Metodologia do experimento

### 2.1 Comparação entre LLMs (Baseline)

**Procedimento:**
1. Extração de texto dos capítulos via pdfplumber
2. Para cada par de modelos (6 combinações):
   - Modelo A gera extração com prompt estruturado
   - Modelo B valida a extração de A (pontuação 0-100)
   - Modelo B gera sua própria extração
   - Modelo A valida a extração de B
3. Concordância definida quando ambas validações ≥ 70/100
4. Conceitos validados em 2+ pares são considerados de alta confiança

**Modelos utilizados:**
| Modelo | Provider | Temperatura |
|--------|----------|-------------|
| GPT-3.5 | OpenAI | 0 |
| GPT-4.1 | OpenAI | 0 |
| GPT-5.1 | OpenAI | 0 |
| Claude-4.5-Opus | Anthropic | 0 |

### 2.2 Atuação dos agentes

**Baseline (9 agentes):**
- Proponentes trabalham em paralelo (cada um gera independentemente)
- Críticos avaliam TODAS as propostas (9 críticas por capítulo)
- Moderador recebe matriz completa de propostas + críticas
- Consenso gera versão unificada
- Auditor verifica checklist de etapas

**Otimizado (8 agentes):**
- Pipeline sequencial (cada agente recebe saída do anterior)
- Crítico encontra problemas específicos com categorização
- Defensor pode rejeitar críticas inválidas
- Moderador decide ação por conceito
- Auditor foca em regras obrigatórias (definition em todos conceitos)

### 2.3 Critérios de validação

**Conceito válido deve:**
- Aparecer explicitamente no texto do capítulo
- Ter definição coerente com o conteúdo
- Possuir pelo menos uma relação `definition`
- Utilizar apenas os 5 tipos de relação permitidos

**Relação válida deve:**
- Conectar conceitos existentes
- Ter justificativa baseada no texto
- Seguir a direção correta (ex: prerequisite aponta para o pré-requisito)

### 2.4 Regras de consenso

**Baseline:**
- Conceito aceito se aparece em 2+ propostas OU não tem críticas graves
- Meta: 8-12 conceitos por capítulo
- Limite: máximo 5 relações inter-conceito

**Otimizado:**
- Conceitos sem `definition` devem ser corrigidos (não removidos)
- Taxa de aprovação mínima: 70%
- Score de qualidade calculado pelo consenso (0.0-1.0)

### 2.5 Limitações e observações

**Baseline:**
- Alto custo computacional: muitas chamadas API por capítulo
- Tempo médio: ~5 minutos por capítulo
- Total para 2 capítulos: 166.970 tokens, 602 segundos
- Projeção para 26 capítulos: ~2.1M tokens, ~130 minutos

**Otimizado:**
- Menos chamadas API por capítulo
- Total para 26 capítulos: 306.543 tokens
- Modelo único reduz variabilidade mas pode introduzir viés

---

## 3. Evidências e resultados

### 3.1 Resultados da validação cruzada (Baseline)

**Capítulo 1 - "Introdução ao Linux":**
| Par de Modelos | Pontuação Média | Concordância |
|----------------|-----------------|--------------|
| GPT-3.5 ↔ GPT-4.1 | 95/100 | ✅ |
| GPT-3.5 ↔ GPT-5.1 | 95/100 | ✅ |
| GPT-3.5 ↔ Claude-4.5-Opus | 92/100 | ✅ |
| GPT-4.1 ↔ GPT-5.1 | 95/100 | ✅ |
| GPT-4.1 ↔ Claude-4.5-Opus | 95/100 | ✅ |
| GPT-5.1 ↔ Claude-4.5-Opus | 100/100 | ✅ |

**Resultado:** 6/6 pares concordantes (100%)

**Capítulo 2 - "Certificações Linux":**
| Par de Modelos | Pontuação Média | Concordância |
|----------------|-----------------|--------------|
| GPT-3.5 ↔ GPT-4.1 | 94/100 | ✅ |
| GPT-3.5 ↔ GPT-5.1 | 100/100 | ✅ |
| GPT-3.5 ↔ Claude-4.5-Opus | 92/100 | ✅ |
| GPT-4.1 ↔ GPT-5.1 | 95/100 | ✅ |
| GPT-4.1 ↔ Claude-4.5-Opus | 94/100 | ✅ |
| GPT-5.1 ↔ Claude-4.5-Opus | 100/100 | ✅ |

**Resultado:** 6/6 pares concordantes (100%)

**Conceitos mais validados (frequência ≥2x):**
- Linux (12x)
- kernel (8x)
- sistema operacional (7x)
- GNU (5x)
- kernel Linux (5x)

### 3.2 Resultados do Pipeline de agentes (Baseline)

**Capítulo 1:**
| Fase | Agente | Resultado |
|------|--------|-----------|
| 1 | Proponente_1 (GPT-4.1) | 12 conceitos, 5 relações |
| 1 | Proponente_2 (GPT-5.1) | 12 conceitos, 5 relações |
| 1 | Proponente_3 (Claude-4.5-Opus) | 12 conceitos, 5 relações |
| 2 | Crítico_1 | ✅ prop_1, ❌ prop_2, ❌ prop_3 |
| 2 | Crítico_2 | ✅ prop_1, ❌ prop_2, ❌ prop_3 |
| 2 | Crítico_3 | ✅ prop_1, ✅ prop_2, ✅ prop_3 |
| 3 | Moderador | 10 conceitos candidatos, 5 relações |
| 4 | Consenso | ✅ Aprovado - 10 conceitos finais |
| 5 | Auditor | ✅ Processo OK, ✅ Aprovado final |

**Capítulo 2:**
- Proponentes: 12, 10 e 11 conceitos respectivamente
- Moderador: 11 conceitos candidatos
- Consenso: ✅ Aprovado - 11 conceitos finais
- Auditor: ✅ Processo OK

### 3.3 Resultados do Pipeline Otimizado (26 capítulos)

**Estatísticas gerais:**
- Capítulos processados: 26
- Conceitos extraídos: 202
- Relações identificadas: 505
- Tokens consumidos: 306.543

**Distribuição de relações:**
| Tipo | Quantidade | Percentual |
|------|------------|------------|
| definition | 202 | 40.0% |
| includes | 125 | 24.8% |
| part-of | 107 | 21.2% |
| property | 36 | 7.1% |
| prerequisite | 35 | 6.9% |

**Eficácia dos agentes:**
- Taxa média de validação: ~85-90%
- Taxa de remoção de conceitos: ~32% (controle de qualidade)
- Problemas encontrados pelo crítico: média de 4.0 por capítulo

**Exemplos de conceitos aceitos:**
- Capítulo 1: Linux, kernel Linux, sistema operacional, GNU, Linus Torvalds
- Capítulo 5: Red Hat, Debian, Ubuntu, Fedora, distribuição Linux
- Capítulo 23: FHS, /etc, /bin, /var, hierarquia de diretórios

**Exemplos de conceitos rejeitados/modificados:**
- Conceitos genéricos sem evidência textual direta
- Conceitos sem relação `definition` (corrigidos)
- Relações com tipos não padronizados (normalizadas)

### 3.4 Divergências entre modelos

**Observadas no baseline:**
- GPT-3.5 extraiu menos conceitos (5 vs 12 dos outros)
- Claude-4.5-Opus foi mais tolerante como validador
- GPT-4.1 e GPT-5.1 apresentaram alta concordância entre si
- Modelos mais recentes (GPT-5.1, Claude-4.5-Opus) tiveram pontuação perfeita (100/100)

**Correções realizadas:**
- Normalização de tipos de relação (ex: "is-a" → "definition")
- Adição de relação `definition` quando ausente
- Remoção de conceitos sem evidência textual

### 3.5 Métricas de Custo

**Baseline (2 capítulos):**
- Tokens entrada: 122.109
- Tokens saída: 44.861

**Otimizado (26 capítulos):**
- Tokens totais: 306.543

---

## 4. Tabelas Comparativas

### 4.1 Comparação entre Abordagens

| Critério | Baseline | Otimizado |
|----------|----------|-----------|
| Capítulos processados | 2 | 26 |
| LLMs utilizadas | 4 | 1 |
| Agentes | 9 | 8 |
| Chamadas API/capítulo | ~39 | ~8 |
| Tokens/capítulo | ~83.485 | ~11.790 |
| Tempo/capítulo | ~300s | ~60s |
| Escalabilidade | Baixa | Alta |
| Validação cruzada LLM | Sim | Não |
| Debate estruturado | Sim | Simplificado |

### 4.2 Qualidade dos resultados

| Métrica | Baseline (2 caps) | Otimizado (26 caps) |
|---------|-------------------|---------------------|
| Conceitos/capítulo | 10-11 | 7.8 (média) |
| Taxa concordância LLMs | 100% | N/A |
| Taxa aprovação | ~95% | ~85-90% |
| Tipos relação cobertos | 5/5 | 5/5 |

### 4.3 Distribuição por Capítulo (Otimizado)

| Cap | Conceitos | Relações | definition | prerequisite | property | part-of | includes |
|-----|-----------|----------|------------|--------------|----------|---------|----------|
| 1 | 7 | 15 | 7 | 1 | 0 | 2 | 5 |
| 2 | 12 | 31 | 12 | 0 | 1 | 7 | 11 |
| 5 | 6 | 13 | 6 | 3 | 0 | 2 | 2 |
| 23 | 7 | 16 | 7 | 0 | 1 | 5 | 3 |
| 24 | 7 | 19 | 7 | 1 | 0 | 1 | 10 |
| **TOTAL** | **202** | **505** | **202** | **35** | **36** | **107** | **125** |

---

## 5. Conclusão objetiva: Qual técnica apresentou melhor resultado?

### 5.1 Análise Comparativa

**Baseline (SINKT_Baseline.ipynb):**
- ✅ Implementação completa conforme especificação
- ✅ Alta taxa de concordância entre LLMs (100%)
- ✅ Debate estruturado rigoroso
- ✅ Validação cruzada multi-modelo
- ❌ Não escalável (muitos tokens para processar o PDF completo)
- ❌ Alto custo computacional
- ❌ Cobertura limitada 

**Otimizado (SINKT_pipeline_agents.ipynb):**
- ✅ Processou 100% dos capítulos
- ✅ Eficiência de tokens 7x maior
- ✅ Pipeline reproduzível e escalável
- ✅ Todos os 5 tipos de relação obrigatórios presentes
- ✅ 202 conceitos extraídos
- ⚠️ Usa apenas 1 modelo (potencial viés), pode ser ajustado para ter uma validação extra com outro modelo

### 5.2 Possível abordagem híbrida

Proposta de **combinação das duas abordagens**:

1. **Para produção em larga escala:** Utilizar o pipeline otimizado (SINKT_pipeline_agents.ipynb) que demonstrou capacidade de processar todos os 26 capítulos com eficiência e qualidade adequada.

2. **Para validação de qualidade:** Aplicar a validação cruzada multi-LLM do baseline em uma amostra representativa (ex: 3-5 capítulos) para garantir que não há viés sistemático.

### 5.3 Justificativa

O **pipeline otimizado é a escolha pragmática** pelos seguintes motivos:

1. **Completude:** Processou 100% do material em um tempo muito menor do que seria na primeira abordagem
2. **Eficiência:** 7x menos tokens consumidos por capítulo
3. **Escalabilidade:** Processamento em lotes com salvamento incremental
4. **Qualidade comprovada:** 
   - Taxa de aprovação ~85-90%
   - Todos os tipos de relação obrigatórios presentes
   - Sistema de defesa evitou remoção de conceitos válidos
   - 32% de conceitos filtrados (controle de qualidade ativo)

5. **Reprodutibilidade:** Pipeline pode ser executado novamente com parâmetros ajustados

### 5.4 Melhorias para próximos passos

1. **Validação híbrida:** Executar validação cruzada multi-LLM em amostra de 10-20% dos capítulos
2. **Ensemble de modelos:** Usar GPT-4.1 + Claude em paralelo nos agentes críticos
3. **Métricas de grafo:** Implementar validação automática de ciclos e nós órfãos
4. **Human-in-the-loop:** Adicionar revisão humana para conceitos com score < 0.7

---

## 6. Arquivos Gerados

### Notebook Baseline:
- `matriz_validacao_cap1.json` - Resultados da validação cruzada cap 1
- `matriz_validacao_cap2.json` - Resultados da validação cruzada cap 2
- `agentes_cap1_completo.json` - Logs completos dos agentes cap 1
- `agentes_cap2_completo.json` - Logs completos dos agentes cap 2
- `extracao_final_mesclada.json` - Conceitos consolidados caps 1-2

### Notebook Otimizado:
- `sumario_estruturado.json` - Estrutura do documento
- `todos_capitulos.json` - Texto extraído de todos os capítulos
- `lote_01_05.json` até `lote_21_26.json` - Resultados por lote
- `grafo_final_completo.json` - Grafo consolidado
- `grafo_final_completo.graphml` - Formato para Gephi
- `grafo_final_completo.gexf` - Formato alternativo
- `grafo_final_kamada.png` - Visualização Kamada-Kawai
- `grafo_final_hierarquico.png` - Visualização hierárquica
- `relatorio_evidencias.txt` - Relatório detalhado de evidências

---

## 7. Referências aos Notebooks

- **SINKT_Baseline.ipynb:** Implementação completa da especificação com validação cruzada multi-LLM e debate estruturado com 9 agentes, aplicada aos capítulos 1 e 2.

- **SINKT_pipeline_agents.ipynb:** Implementação escalável com pipeline de 8 agentes e processamento em lotes, aplicada a todos os 26 capítulos do material.

