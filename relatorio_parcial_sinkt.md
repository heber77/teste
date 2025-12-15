# Relatório Parcial: Simulação de Estudantes Sintéticos para SINKT

**Aluno:** Héber Júnior  
**Data:** 15/12/2025  
**Status:** Em andamento

---

## Resumo

Este relatório parcial descreve o progresso na atividade de simulação de estudantes sintéticos para resolver o problema de cold start do SINKT. O trabalho realizado até o momento inclui: reaproveitamento dos conceitos extraídos na atividade anterior (202 conceitos do livro de Certificação Linux), definição de 6 perfis cognitivos fundamentados em teoria educacional, geração de 352 questões sintéticas via LLM, simulação de 100 estudantes com 4.000 interações, e validação estatística dos dados gerados. A etapa de treinamento do modelo SINKT será realizada na próxima fase.

---

## Resultados

### 1. Reaproveitamento dos Conceitos da Atividade Anterior
Foram carregados os dados da extração de conceitos realizada anteriormente, totalizando 220 conceitos (176 principais com definições completas) e 263 relações entre conceitos, incluindo 34 relações de pré-requisito que formam o grafo de conhecimento.

### 2. Definição de 6 Perfis Cognitivos
Foram criados 6 perfis baseados em fatores cognitivos validados pela literatura (Modelo BKT, Teoria da Carga Cognitiva de Sweller, Teoria da Aprendizagem Significativa de Ausubel):

| Perfil | Domínio | Slip | Guess | Característica Principal |
|--------|---------|------|-------|--------------------------|
| Expert Técnico | 80% | 5% | 10% | Alta experiência prévia |
| Iniciante Dedicado | 10% | 10% | 15% | Motivado, progresso lento |
| Apressado | 50% | 25% | 30% | Erra por falta de atenção |
| Metódico Lento | 20% | 5% | 5% | Lento mas preciso |
| Autodidata | 60% | 10% | 20% | Conhecimento com lacunas |
| Desmotivado | 10% | 20% | 40% | Baixo engajamento |

Fatores demográficos (idade, sexo, região, classe social, religião) foram **excluídos** por não terem relação causal com capacidade de aprendizagem e por introduzirem viés discriminatório.

### 3. Geração de 352 Questões Sintéticas
Utilizando a API OpenAI (modelo gpt-4o-mini), foram geradas 2 questões por conceito, totalizando 352 questões distribuídas em 3 tipos: definição (88), múltipla escolha (176) e aplicação (88), com dificuldades variando de 30% a 70%.

### 4. Simulação de 100 Estudantes Sintéticos
Foram gerados 100 estudantes distribuídos entre os perfis (25% Iniciante Dedicado, 20% Apressado, 15% Metódico, 15% Autodidata, 15% Desmotivado, 10% Expert), cada um com variação individual de ±10% nos fatores cognitivos para simular diversidade realista.

### 5. Simulação de 4.000 Interações
Cada estudante realizou 40 interações seguindo progressão de dificuldade. O modelo BKT foi utilizado para calcular probabilidade de acerto considerando: conhecimento do conceito, domínio de pré-requisitos, e fatores cognitivos do perfil.

### 6. Classificação de Tipos de Erro
Os erros foram classificados em 4 categorias: slip (erro por descuido), não_sabe (falta de conhecimento), falta_prerequisito (não domina base), e interpretação (não entendeu o enunciado). A distribuição de erros mostrou coerência com os perfis (ex: Apressado com 26.6% de slip vs Metódico com 5%).

### 7. Validação Estatística - 4/4 Testes Passaram
- **Teste 1:** Expert (83.8%) > Desmotivado (56.7%) ✅
- **Teste 2:** Variância Metódico < Variância Apressado ✅
- **Teste 3:** Range entre perfis > 15% (27.1%) ✅
- **Teste 4:** 62% dos estudantes melhoraram ao longo das interações ✅

### 8. Arquivos Gerados
- `perfis_cognitivos.json`: Definição completa dos 6 perfis
- `questoes_sinteticas.json`: 352 questões geradas
- `dataset_sintetico_completo.json`: Dataset com todos os estudantes
- `interacoes_sinteticas.csv`: 4.000 interações em formato tabular
- `sinkt_dataset.json`: Dados formatados para treinamento
- `data_splits.json`: Divisão train/val/test (70/15/15)

---

## Dificuldades Encontradas

### 1. Criação de Perfis Cognitivos Realistas
O principal desafio foi definir perfis que representassem comportamentos humanos plausíveis sem recorrer a fatores demográficos. A solução foi fundamentar cada perfil em teoria educacional validada (BKT, Teoria da Carga Cognitiva) e validar empiricamente através de testes estatísticos que comparam o comportamento esperado vs observado.

### 2. Calibração do Ganho de Aprendizado
Inicialmente, com 40 interações e ganho de conhecimento de 0.1 por acerto, apenas 27% dos estudantes mostravam progressão. Foi necessário ajustar o ganho para 0.12 (acertos) e 0.06 (erros), incorporando também o fator memória, para que 62% dos estudantes demonstrassem melhoria - um valor mais realista.

### 3. Balanceamento entre Diversidade e Coerência
Garantir que os dados sintéticos fossem diversos (variação de ±10% nos fatores) mas ainda coerentes com os perfis base exigiu múltiplas iterações nos parâmetros de simulação até que todos os 4 testes de validação passassem simultaneamente.

---

## Conclusão a Respeito da Atividade Sendo Validada

Os resultados parciais demonstram que é possível gerar dados sintéticos de aprendizagem que simulam comportamentos humanos plausíveis. Os 4 testes de validação passando indicam que: (1) os perfis cognitivos produzem diferenciação significativa no desempenho, (2) as características de cada perfil se refletem nos padrões de erro, e (3) existe progressão de aprendizado ao longo das interações. O dataset gerado está pronto para ser utilizado no treinamento do modelo SINKT, que será realizado na próxima etapa.

---

## Próximos Passos

- **Treinamento do modelo SINKT:** Utilizar o dataset sintético gerado para treinar uma GRU (Gated Recurrent Unit) que prediz probabilidade de acerto com base no histórico de interações do estudante.

- **Avaliação de métricas:** Calcular AUC-ROC, Accuracy e F1-Score no conjunto de teste para validar a capacidade preditiva do modelo.

- **Demonstração do cold start:** Mostrar que o modelo treinado apenas com dados sintéticos consegue prever o desempenho de "novos alunos" (estudantes do conjunto de teste).

- **Experimentos adicionais:** Realizar variações nos parâmetros (número de estudantes, interações, arquitetura do modelo) para análise de sensibilidade.

- **Relatório final:** Consolidar todos os resultados, incluindo treinamento e métricas finais, com respostas completas às 5 perguntas obrigatórias da atividade.
