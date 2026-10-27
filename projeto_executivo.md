# 📋 PROJETO EXECUTIVO DE ENGENHARIA REVERSA E ARQUITETURA
## Sistema AuditorIA (Baseado no ecossistema e-Auditoria)

---

### 1. INTRODUÇÃO E OBJETIVOS

O objetivo deste projeto executivo é documentar a engenharia reversa das funcionalidades core da plataforma **e-Auditoria**, estabelecendo as diretrizes técnicas para o desenvolvimento do **AuditorIA**, um módulo complementar de inteligência fiscal integrado ao ecossistema **Essencial Gestão**.

#### Objetivos Estratégicos:
1. **Auditoria Preventiva:** Sanar inconsistências fiscais em arquivos SPED e cadastros de produtos antes do envio à Receita Federal e SEFAZ.
2. **Recuperação Tributária:** Identificar automaticamente valores pagos a maior em produtos monofásicos (PIS/COFINS) e ST do ICMS no Simples Nacional.
3. **Simulação da Reforma Tributária (2027-2033):** Oferecer aos clientes análises comparativas de cenários (Simples Unificado vs. Híbrido vs. Presumido vs. Real) sob a égide da LC 214/2025.

---

### 2. ENGENHARIA REVERSA DOS MOTORES CORE

#### Motor 1: Parser e Corretor de SPED
* **Entrada:** Arquivos `.txt` delimitados por pipe (`|`) correspondentes ao EFD ICMS/IPI e EFD Contribuições.
* **Processamento:**
  - Validação da estrutura hierárquica (Bloco 0 -> Bloco C -> Bloco E -> Bloco 1).
  - Verificação de integridade referencial dos participantes (0150) e itens (0200).
  - Cruzamento de somatórios do Registro C100 com itens do C170 e apuração do E110/M200.
* **Saída:** Arquivo SPED sanitizado e relatório de divergências estruturado.

#### Motor 2: Matriz Tributária e Classificação de Itens
* **Entrada:** Cadastro de itens da empresa (NCM, CEST, Descrição, CST, CFOP).
* **Processamento:**
  - Consulta à Matriz de Regras para mapear se a NCM pertence à lista de Produtos Monofásicos de PIS/COFINS (Autopeças, Medicamentos, Bebidas, Cosméticos).
  - Cruzamento do CST de saída do DAS com a tabela de tributação concentrada.
* **Saída:** Extrato de Segregação do DAS com segregação de receitas e cálculo de economia fiscal.

#### Motor 3: Simulação de Cenários da Reforma Tributária
* **Entrada:** Faturamento Anual Bruto, Valor de Compras/Insumos (com direito a crédito), Folha de Pagamento.
* **Processamento:**
  - Projeção da alíquota gradual do IBS e CBS de 2027 a 2033 conforme cronograma oficial.
  - Cálculo comparativo da carga tributária efetiva nos 4 regimes simultaneamente.
* **Saída:** Laudo Executivo em PDF e dashboard comparativo interativo.

---

### 3. ESPECIFICAÇÃO DE DADOS (MODELAGEM DDL POSTGRESQL)

```sql
-- Tabela de Produtos / Cadastros sanitizados
CREATE TABLE IF NOT EXISTS public.produtos_auditados (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    empresa_id UUID NOT NULL,
    codigo_item VARCHAR(60) NOT NULL,
    descricao TEXT NOT NULL,
    ncm VARCHAR(8) NOT NULL,
    cest VARCHAR(7),
    cst_pis_cofins VARCHAR(3),
    cst_icms VARCHAR(3),
    is_monofasico BOOLEAN DEFAULT FALSE,
    aliquota_pis NUMERIC(5,2) DEFAULT 0.00,
    aliquota_cofins NUMERIC(5,2) DEFAULT 0.00,
    criado_em TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Tabela de Auditoria de Arquivos SPED
CREATE TABLE IF NOT EXISTS public.auditorias_sped (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    empresa_id UUID NOT NULL,
    tipo_sped VARCHAR(20) CHECK (tipo_sped IN ('EFD_ICMS_IPI', 'EFD_CONTRIBUICOES')),
    periodo_referencia VARCHAR(7) NOT NULL, -- Formato MM/YYYY
    qtd_erros_identificados INT DEFAULT 0,
    status VARCHAR(20) DEFAULT 'PROCESSANDO', -- PROCESSANDO, CONCLUIDO, ERRO
    log_divergencias JSONB,
    criado_em TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

### 4. BIZUS DE IMPLEMENTAÇÃO E PERFORMANCE

* **Estratégia de Performance no Parser:** Processar arquivos texto em chunks via stream. Não carregar arquivos de 500MB inteiros na memória de uma vez.
* **Privacidade e Conformidade:** Garantir que laudos gerados não exponham dados desnecessários e apresentem a assinatura do escritório de forma limpa e corporativa.
* **Padrão de Interface:** Todos os relatórios gerados em HTML devem seguir o padrão corporativo com suporte nativo a impressão (`window.print()`), exportação CSV e esquema de cores profissional.
