# 💊 Parametrização e Auditoria de Tabelas Brasíndice e SIMPRO

Metodologia técnica e rotinas de validação para importação, vínculo e auditoria de tabelas referenciais de medicamentos (**Brasíndice**) e materiais médico-hospitalares (**SIMPRO**) em sistemas ERP.

---

## 🎯 O Problema de Negócio
Atualizações quinzenais ou mensais das tabelas Brasíndice e SIMPRO, quando importadas sem validação crítica, causam erros de fracionamento (apresentação vs. unidade de consumo), códigos TUSS desatualizados e cobrança de valores defasados perante as operadoras.

## 💡 A Solução Técnica
Implementação de checklist de governança e consultas SQL de batimento automático entre o cadastro interno do hospital e as tabelas oficiais vigentes.

### Pontos Críticos Auditados:
1. **Fator de Conversão / Fracionamento:** Garantia de que o preço da caixa/frasco está corretamente dividido pela unidade mínima dispensada.
2. **Vínculo Código TISS/TUSS:** Validação de códigos ativos para evitar glosas administrativas imediatas no envio do XML.
3. **Regra de Margem/Deflator Contratual:** Conferência da aplicação correta de PF (Preço Fábrica), PMC (Preço Máximo ao Consumidor) e percentuais acordados por convênio.

---

## 💻 Script Fictício: Detecção de Divergência de Preço e Fracionamento

```sql
/* ============================================================================
   OBJETIVO: Comparar o valor unitário parametrizado no ERP com o valor 
             calculado pela tabela referencial vigente (Brasíndice/SIMPRO).
   ============================================================================ */

SELECT 
    I.CODIGO_INTERNO,
    I.DESCRICAO_ITEM,
    T.CODIGO_REFERENCIA_OFICIAL,
    T.PRECO_EMBALAGEM AS PRECO_TABELA_OFICIAL,
    I.FATOR_FRACIONAMENTO,
    ROUND((T.PRECO_EMBALAGEM / NULLIF(I.FATOR_FRACIONAMENTO, 0)), 4) AS VALOR_UNITARIO_ESPERADO,
    P.VALOR_COBRADO_ERP AS VALOR_ATUAL_SISTEMA,
    (P.VALOR_COBRADO_ERP - ROUND((T.PRECO_EMBALAGEM / NULLIF(I.FATOR_FRACIONAMENTO, 0)), 4)) AS DIFERENCA_UNITARIA
FROM TB_ITENS_HOSPITALARES I
INNER JOIN TB_TABELA_REFERENCIA_VIGENTE T 
    ON I.CODIGO_REFERENCIA = T.CODIGO_REFERENCIA_OFICIAL
INNER JOIN TB_PRECOS_CONVENIO P 
    ON I.CODIGO_INTERNO = P.CODIGO_INTERNO
WHERE 
    ABS(P.VALOR_COBRADO_ERP - ROUND((T.PRECO_EMBALAGEM / NULLIF(I.FATOR_FRACIONAMENTO, 0)), 4)) > 0.05
ORDER BY 
    ABS(DIFERENCA_UNITARIA) DESC;
