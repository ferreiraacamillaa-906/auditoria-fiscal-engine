# Projeto Executivo — AuditorIA / Essencial Automação Fiscal

## 1. Objetivo

Automatizar a auditoria fiscal de arquivos SPED, reproduzindo a lógica das ferramentas de mercado (referência: e-Auditoria) com regras próprias, e simular o impacto da Reforma Tributária 2027–2033.

## 2. Engenharia reversa do e-Auditoria

> Esta seção descreve o comportamento observável da ferramenta (entradas, saídas e fluxo de uso). Não inclui código, credenciais nem conteúdo protegido do fornecedor. Confirme os termos de uso do serviço antes de reproduzir qualquer funcionalidade.

### 2.1 Fluxo observado

1. **Importação** do SPED (EFD ICMS/IPI e EFD Contribuições) por empresa e período.
2. **Validação estrutural** dos blocos e registros (0, C, D, E, G, H, K).
3. **Cruzamentos** entre registros: itens × cadastro de produtos (0200), CST × CFOP, NCM × alíquota, apuração × documentos.
4. **Classificação** de cada achado por severidade (alta, média, baixa) e por tributo.
5. **Relatório** com a lista de inconsistências, valor potencial de impacto e exportação.

### 2.2 Mapeamento para os módulos do AuditorIA

| Comportamento observado | Módulo | Registros SPED envolvidos |
|---|---|---|
| Leitura e normalização | Parser SPED | 0000, 0150, 0200, C100, C170, C190, E110 |
| Cruzamentos e regras | Matriz de Regras | C170 (CST, CFOP, NCM, alíquota) × 0200 |
| Classificação e relatório | Dashboard | tabela `auditoria_sped` |
| Impacto da reforma | Comparador | base de cálculo anual do cliente |

### 2.3 Regras iniciais da matriz (exemplos)

| Código | Regra | Severidade |
|---|---|---|
| R001 | NCM do item diferente do NCM cadastrado no produto | Média |
| R002 | CST de ICMS incompatível com o CFOP da operação | Alta |
| R003 | Alíquota de ICMS divergente da esperada para UF/NCM | Alta |
| R004 | CST de PIS/Cofins inconsistente com o regime da empresa | Média |
| R005 | Produto sem NCM ou com NCM inexistente na tabela vigente | Baixa |

## 3. Modelo de dados (PostgreSQL / Supabase)

```sql
-- Empresas auditadas
CREATE TABLE empresas (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cnpj        CHAR(14) NOT NULL UNIQUE,
    razao_social TEXT NOT NULL,
    uf          CHAR(2) NOT NULL,
    regime      TEXT NOT NULL CHECK (regime IN ('simples', 'presumido', 'real')),
    criado_em   TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Cadastro de produtos (registro 0200 do SPED)
CREATE TABLE produtos (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    empresa_id   UUID NOT NULL REFERENCES empresas(id) ON DELETE CASCADE,
    cod_item     TEXT NOT NULL,
    descricao    TEXT NOT NULL,
    ncm          CHAR(8),
    cest         CHAR(7),
    unidade      TEXT,
    aliq_icms    NUMERIC(5,2),
    cst_icms     CHAR(3),
    cst_pis      CHAR(2),
    cst_cofins   CHAR(2),
    atualizado_em TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (empresa_id, cod_item)
);

CREATE INDEX idx_produtos_ncm ON produtos (ncm);

-- Arquivos SPED importados
CREATE TABLE sped_arquivos (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    empresa_id  UUID NOT NULL REFERENCES empresas(id) ON DELETE CASCADE,
    tipo        TEXT NOT NULL CHECK (tipo IN ('EFD_ICMS_IPI', 'EFD_CONTRIBUICOES')),
    competencia DATE NOT NULL,
    hash_sha256 CHAR(64) NOT NULL,
    importado_em TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (empresa_id, tipo, competencia, hash_sha256)
);

-- Inconsistências encontradas
CREATE TABLE auditoria_sped (
    id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    arquivo_id     UUID NOT NULL REFERENCES sped_arquivos(id) ON DELETE CASCADE,
    produto_id     UUID REFERENCES produtos(id) ON DELETE SET NULL,
    regra          TEXT NOT NULL,
    severidade     TEXT NOT NULL CHECK (severidade IN ('alta', 'media', 'baixa')),
    tributo        TEXT NOT NULL,
    registro_sped  TEXT NOT NULL,
    linha_arquivo  INTEGER,
    descricao      TEXT NOT NULL,
    valor_impacto  NUMERIC(14,2) NOT NULL DEFAULT 0,
    status         TEXT NOT NULL DEFAULT 'aberta'
                   CHECK (status IN ('aberta', 'em_analise', 'corrigida', 'descartada')),
    criado_em      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_auditoria_arquivo ON auditoria_sped (arquivo_id);
CREATE INDEX idx_auditoria_status  ON auditoria_sped (status, severidade);

-- Row Level Security (Supabase): ajuste a política ao modelo de usuários adotado
ALTER TABLE empresas        ENABLE ROW LEVEL SECURITY;
ALTER TABLE produtos        ENABLE ROW LEVEL SECURITY;
ALTER TABLE sped_arquivos   ENABLE ROW LEVEL SECURITY;
ALTER TABLE auditoria_sped  ENABLE ROW LEVEL SECURITY;
```

## 4. Reforma Tributária 2027–2033

Cronograma considerado no comparador (LC 214/2025):

| Ano | Efeito |
|---|---|
| 2026 | Ano de teste: CBS 0,9% e IBS 0,1%, compensáveis |
| 2027 | CBS entra em vigor e PIS/Cofins são extintos; IBS a 0,1% |
| 2028 | Mesma situação de 2027 |
| 2029–2032 | IBS sobe (10%, 20%, 30%, 40% da alíquota) e ICMS/ISS caem na proporção inversa |
| 2033 | IBS pleno; ICMS e ISS extintos |

As alíquotas de referência (CBS 8,8% e IBS 17,7%) usadas no dashboard são **ilustrativas**. As alíquotas reais serão fixadas por resolução do Senado e variam por ente federado e por produto. Revise o cronograma contra a legislação vigente antes de usar os números com clientes.

## 5. Diretrizes operacionais

1. **Dados do cliente**: arquivos SPED contêm dados fiscais sensíveis. Nunca versionar (ver `.gitignore`) e tratar sob a LGPD.
2. **Segredos**: chaves do Supabase só em `.env`; usar a `anon key` no front e a `service_role` apenas no back-end.
3. **Idempotência**: reimportar o mesmo arquivo não pode duplicar achados (hash SHA-256 em `sped_arquivos`).
4. **Rastreabilidade**: todo achado guarda registro e linha de origem no arquivo.
5. **Revisão humana**: o sistema aponta inconsistências e um profissional contábil valida antes de retificar a escrituração.
6. **Versionamento de regras**: cada regra da matriz tem código e vigência. Mudança de legislação gera nova versão, não sobrescrita.
7. **Testes**: cada regra nova entra com pelo menos um caso positivo e um negativo em arquivo SPED de exemplo anonimizado.

## 6. Roadmap sugerido

1. Parser SPED para C100/C170/0200 e carga no Supabase.
2. Regras R001–R005 e dashboard consumindo a tabela `auditoria_sped`.
3. Comparador da Reforma com dados reais de faturamento do cliente.
4. Ampliação da matriz (substituição tributária, DIFAL, créditos de PIS/Cofins).
