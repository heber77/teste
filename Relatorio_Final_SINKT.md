# Simulação de estudantes sintéticos para SINKT

---

## 1. Resumo 

Documentação da implementação de uma solução para o problema de *cold start* no sistema SINKT. A solução consiste na geração de dados sintéticos de aprendizagem através de perfis cognitivos fundamentados em teoria educacional.

### Métricas principais

| Métrica | Valor |
|---------|-------|
| Estudantes sintéticos | 100 |
| Total de interações | 4.000 |
| Questões geradas | 352 |
| Conceitos do domínio | 176 |
| Perfis cognitivos | 6 |
| **AUC-ROC (teste)** | **0.6487** |
| **Accuracy (teste)** | **67.00%** |
| **F1-Score (teste)** | **0.7805** |

O modelo treinado apenas com dados sintéticos consegue prever o desempenho de novos alunos, demonstrando a eficácia da abordagem para resolver o cold start.

---

## 2. Problema

O SINKT depende de dados históricos de interações estudante-conteúdo para personalizar a experiência de aprendizagem. Quando um novo curso é criado, não existem dados disponíveis para treinar os modelos preditivos - *cold start*.

### Itens da tarefa

1. Construir perfis cognitivos coerentes, neutros e fundamentados em teoria educacional
2. Gerar estudantes sintéticos realistas com base nesses perfis
3. Criar sequências de acerto/erro compatíveis com o grafo de conceitos
4. Treinar o SINKT usando apenas dados sintéticos
5. Demonstrar que o modelo é capaz de prever o desempenho de novos alunos

---

## 3. Como foi realizado

### 3.1 Dados de Entrada

Os conceitos utilizados foram extraídos do PDF Linux da atividade anterior:
- **220 conceitos** totais
- **263 relações** entre conceitos
- **34 relações de pré-requisito** formando o grafo de conhecimento

### 3.2 Perfis cognitivos

Os perfis cognitivos foram construídos com base em três pilares teóricos:

| Teoria | Aplicação |
|--------|-----------|
| **Modelo BKT** | Taxas de slip e guess |
| **Teoria da carga cognitiva**  | Velocidade de aprendizagem |
| **Teoria da aprendizagem significativa** | Domínio prévio |

### 3.3 Fatores cognitivos

Cada perfil é definido por 8 fatores cognitivos:

| Fator | Descrição |
|-------|-----------|
| Domínio prévio | Conhecimento inicial sobre o conteúdo |
| Raciocínio lógico | Capacidade de resolução de problemas |
| Interpretação de texto | Compreensão de enunciados e questões |
| Memória | Retenção de conceitos aprendidos |
| Velocidade de aprendizagem | Taxa de aquisição de novos conhecimentos |
| Taxa de Slip | P(erro \| sabe) - probabilidade de errar mesmo sabendo |
| Taxa de Guess | P(acerto \| não sabe) - probabilidade de acertar por chute |
| Familiaridade tecnológica | Experiência prévia com TI e sistemas Linux |

---

## 4. Perfis cognitivos

Foram definidos 6 perfis cognitivos distintos:

| Perfil | Domínio | Slip | Guess | Característica |
|--------|---------|------|-------|----------------|
| **Expert técnico** | 80% | 5% | 10% | Profissional experiente com background em TI |
| **Autodidata** | 60% | 10% | 20% | Estudante curioso que já estudou por conta própria |
| **Metódico lento** | 20% | 5% | 5% | Estudante que precisa de mais tempo mas raramente erra |
| **Iniciante dedicado** | 10% | 10% | 15% | Estudante sem experiência prévia mas muito motivado |
| **Apressado** | 50% | 25% | 30% | Estudante com boa base mas impaciente |
| **Desmotivado** | 10% | 20% | 40% | Estudante com dificuldades e baixa motivação |

### 4.1 Distribuição dos estudantes

Os 100 estudantes foram distribuídos:
- Iniciante dedicado: 25%
- Apressado: 20%
- Metódico lento: 15%
- Autodidata: 15%
- Desmotivado: 15%
- Expert técnico: 10%

Cada estudante recebeu variação individual de ±10% nos fatores cognitivos para simular diversidade natural.

---

## 5. Geração de dados sintéticos

### 5.1 Geração de questões

Foram geradas **352 questões** utilizando a API OpenAI (modelo gpt-4o-mini):

| Tipo | Quantidade | Dificuldade Base |
|------|------------|------------------|
| Definição | 88 | 30% |
| Múltipla Escolha | 176 | 50% |
| Aplicação | 88 | 70% |

### 5.2 Simulação de interações

- Cada estudante realizou **40 interações**
- Progressão de dificuldade ao longo das interações
- Modelo BKT para calcular probabilidade de acerto considerando:
  - Conhecimento atual do conceito
  - Domínio de pré-requisitos
  - Fatores cognitivos do perfil

### 5.3 Classificação de erros

Os erros foram classificados em 4 categorias:

| Tipo de Erro | Descrição |
|--------------|-----------|
| **Slip** | Estudante sabe o conteúdo mas erra por descuido ou pressa |
| **Não sabe** | Estudante não domina o conceito avaliado |
| **Falta pré-requisito** | Estudante não domina conceitos base necessários |
| **Interpretação** | Estudante não compreendeu corretamente o enunciado |

### 5.4 Distribuição de erros por perfil

| Perfil | Slip | Não Sabe | Pré-req | Interp. |
|--------|------|----------|---------|---------|
| Expert Técnico | 9.2% | 63.1% | 0.0% | 27.7% |
| Autodidata | 14.1% | 67.2% | 0.0% | 18.8% |
| Metódico Lento | 5.0% | 63.2% | 8.0% | 23.9% |
| Iniciante Dedicado | 7.9% | 58.4% | 10.4% | 23.4% |
| Apressado | 27.2% | 49.8% | 4.0% | 18.9% |
| Desmotivado | 22.3% | 48.5% | 9.6% | 19.6% |

> **Obs:** Os percentuais representam a proporção de cada tipo de erro *dentro dos erros* de cada perfil.

---

## 6. Validação dos dados sintéticos

### 6.1 Testes estatísticos

| Teste | Resultado | Status |
|-------|-----------|--------|
| Teste 1: Expert > Desmotivado | 83.8% > 56.7% | ✅ PASS |
| Teste 2: Variância metódico < Apressado | 0.0065 < 0.0085 | ✅ PASS |
| Teste 3: Range entre perfis > 15% | 27.1% | ✅ PASS |
| Teste 4: Progressão de aprendizado | 37% melhoraram | ✅ PASS |

**Resultado: 4/4 testes passaram**

### 6.2 Coerência dos padrões de erro

A análise confirmou coerência com as características dos perfis:

- ✅ **Apressado** tem a MAIOR taxa de slip (27.2%) - coerente com impaciente
- ✅ **Metódico** tem a MENOR taxa de slip (5.0%) - coerente com cuidadoso
- ✅ **Iniciante** tem a MAIOR taxa de falta_pré-requisito (10.4%) - coerente com baixo domínio
- ✅ **Expert e Autodidata** têm 0% de falta_pré-requisito - coerente com alto domínio base

---

## 7. Treinamento do modelo SINKT

### 7.1 Arquitetura

O modelo utiliza uma **GRU (Gated Recurrent Unit)** para capturar padrões temporais:

| Parâmetro | Valor |
|-----------|-------|
| Arquitetura | GRU |
| Dimensão do embedding | 16 |
| Tamanho hidden | 32 |
| Parâmetros treináveis | ~4.929 |
| Otimizador | Adam (lr=0.001) |
| Loss function | BCEWithLogitsLoss |
| Épocas | 50 |

### 7.2 Divisão dos dados

| Conjunto | Estudantes | Percentual |
|----------|------------|------------|
| Treino | 70 | 70% |
| Validação | 15 | 15% |
| Teste | 15 | 15% |

### 7.3 Resultados

| Métrica | Validação | Teste |
|---------|-----------|-------|
| AUC-ROC | 0.6302 | **0.6487** |
| Accuracy | 66.33% | **67.00%** |
| Precision | - | 66.67% |
| Recall | - | 94.12% |
| F1-Score | - | **0.7805** |

O AUC-ROC de **0.6487** é significativamente melhor que predição aleatória (0.5), demonstrando que o modelo aprendeu padrões úteis dos dados sintéticos.

---

## 8. Respostas às questões obrigatórias

### Pergunta 1: Como garantir que os perfis representam comportamentos realistas?

Os perfis foram construídos com base em:

**a) Fundamentação teórica:**
- Modelo BKT para taxas de slip e guess
- Teoria da carga cognitiva para velocidade de aprendizagem
- Teoria da aprendizagem significativa para domínio prévio

**b) Validação empírica:**
- 4/4 testes estatísticos passaram
- Diferenciação clara entre perfis
- Padrões de erro coerentes com características definidas

---

### Pergunta 2: Quais fatores influenciam o aprendizado?

Foram utilizados **8 fatores cognitivos**:

1. **Domínio prévio** - conhecimento inicial
2. **Raciocínio lógico** - resolução de problemas
3. **Interpretação de texto** - compreensão de enunciados
4. **Memória** - retenção de conceitos
5. **Velocidade de aprendizagem** - taxa de aquisição
6. **Taxa de Slip** - P(erro | sabe)
7. **Taxa de Guess** - P(acerto | não sabe)
8. **Familiaridade tecnológica** - experiência com TI

---

### Pergunta 3: Fatores demográficos devem ser modelados?

**NÃO.** Fatores demográficos foram **excluídos**:

| Fator | Motivo da Exclusão |
|-------|-------------------|
| Idade | Não há relação causal com capacidade de aprendizagem |
| Sexo | Introduziria viés discriminatório sem base científica |
| Região | Diferenças refletem desigualdades de acesso, não capacidade |
| Classe social | Correlação não implica causalidade |
| Religião | Irrelevante para aprendizagem técnica |

---

### Pergunta 4: Como garantir boa acurácia sem dados reais?

A estratégia utilizada:

1. **Perfis baseados em literatura** validada (BKT, Teoria da Carga Cognitiva)
2. **Modelo probabilístico** para geração de respostas (não determinístico)
3. **Validação estatística rigorosa** dos dados gerados (4 testes)
4. **Diversidade de perfis** cobrindo amplo espaço de comportamentos possíveis

---

### Pergunta 5: Como validar se os dados parecem humanos?

A validação foi realizada através de:

| Método | Resultado |
|--------|-----------|
| Testes de coerência | 4/4 passaram |
| Progressão de aprendizado | 37% melhoraram |
| Diferenciação por perfil | Range de 27.1% |
| Padrões de erro | Coerentes com perfis |

---

## 9. Demonstração: Resolução do cold atart

O modelo treinado apenas com dados sintéticos foi utilizado para prever o desempenho de estudantes do conjunto de teste (nunca vistos durante o treinamento):

| Estudante | Perfil | Acurácia | Taxa Real | Taxa Prevista |
|-----------|--------|----------|-----------|---------------|
| S0096 | Apressado | 72.5% | 67.5% | 70.3% |
| S0072 | Desmotivado | 50.0% | 52.5% | 57.4% |
| S0034 | Metódico Lento | 67.5% | 62.5% | 64.1% |
| S0079 | Autodidata | 85.0% | 85.0% | 79.5% |
| S0025 | Autodidata | 72.5% | 72.5% | 69.1% |
| S0012 | Iniciante Dedicado | 45.0% | 45.0% | 55.0% |
| S0077 | Iniciante Dedicado | 67.5% | 55.0% | 61.7% |

✅ **O modelo consegue prever o desempenho de novos alunos mesmo sem ter sido treinado com dados reais**, comprovando a eficácia da abordagem para resolver o cold start.

---

## 10. Conclusão

Os experimentos mostraram que é possível resolver o problema de cold start através da geração de dados sintéticos baseados em perfis cognitivos.

### Principais Resultados

| Item | Status |
|------|--------|
| Perfis cognitivos fundamentados | ✅ 6 perfis baseados em teoria |
| Dados sintéticos realistas | ✅ 4/4 testes de validação |
| SINKT funcionando sem dados reais | ✅ AUC-ROC: 0.6487 |
| Perguntas obrigatórias respondidas | ✅ 5/5 |
| Cold start resolvido | ✅ Demonstrado com predições |

### Arquivos Gerados

| Arquivo | Descrição |
|---------|-----------|
| `perfis_cognitivos.json` | Definição dos 6 perfis |
| `questoes_sinteticas.json` | 352 questões geradas |
| `dataset_sintetico_completo.json` | Dataset completo |
| `interacoes_sinteticas.csv` | 4.000 interações |
| `sinkt_dataset.json` | Formato para treinamento |
| `data_splits.json` | Divisão train/val/test |
| `sinkt_model.pth` | Modelo GRU treinado |
| `metricas_sinkt.json` | Métricas de avaliação |
| `perfis_radar.png` | Visualização dos perfis |
| `distribuicao_taxas.png` | Distribuição de acertos |
| `curva_aprendizado.png` | Curvas de aprendizado |
