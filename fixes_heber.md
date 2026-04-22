### Média Severidade (2)

| #   | Categoria | Arquivo | Linha | Descrição | Regra/Spec Violada | Sugestão |
| --- | --------- | ------- | ----- | --------- | ------------------ | -------- |
| 1   | Spec / schema — nomenclatura | `src/ml_core/src/model/events/training_payloads.py` | 31–34 | O campo `lr_scheduler` tem valor padrão `"linear_warmup"`, enquanto o pipeline sintético (`synthetic_session.py`) combina warmup com **decaimento cosseno** via `build_adamw_with_warmup_cosine_decay`. Quem serializar hiperparâmetros reais pode publicar um rótulo que não reflete o scheduler efetivo. | Alinhamento com uso real / clareza de contrato | Ajustar o default e/ou a descrição do campo para refletir o scheduler pós-warmup usado (por exemplo identificador explícito para warmup+cosseno), ou documentar no módulo qual combinação é suportada. |
| 2   | Rule Violation — CP5 | `src/ml_core/src/model/training/trainer.py` · `src/ml_core/src/model/training/synthetic_session.py` | (várias) | O laço de treino e a sessão sintética não produzem logs estruturados com `correlation_id` / `tenant_id`. Para biblioteca isolada é aceitável; integrada ao job de treino, dificulta diagnóstico distribuído. | CP5 | Ao integrar com o Training Worker, instrumentar pontos-chave (início/fim de época, checkpoint, falha) com logging estruturado e contexto do job. |

### Baixa Severidade (2)

| #   | Categoria | Arquivo | Linha | Descrição | Regra/Spec Violada | Sugestão |
| --- | --------- | ------- | ----- | --------- | ------------------ | -------- |
| 3   | Spec Deviation — documentação | `src/ml_core/src/model/events/training_payloads.py` | 51–78 | [EVENTS.md](../../docs/deployment-units/ml_core/modules/model/EVENTS.md) descreve `final_metrics` genericamente (“loss, accuracy, AUC, etc.”). O código adiciona `train_loss`, `log_loss`, `expected_calibration_error`, `best_epoch`, `stopped_early`. Comportamento extra e útil; a doc de alto nível não lista subcampos. | EVENTS.md (granularidade) | Atualizar [EVENTS.md](../../docs/deployment-units/ml_core/modules/model/EVENTS.md) com a lista de subcampos suportados quando o time consolidar o contrato, ou tratar como extensão consciente. |
| 4   | Otimização / edge case | `src/ml_core/src/model/training/trainer.py` | 115–127 | Se a métrica de checkpoint for sempre não finita (por exemplo `auc_roc` sempre `nan`), o early stopping nunca incrementa `patience` e o laço corre até `max_epochs`. | — | Em cenários reais com AUC sempre indefinido, considerar métrica de fallback (`loss`) ou parada por número de épocas sem métrica válida (follow-up). |

---