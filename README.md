# AuditorIA — Essencial Automação Fiscal

Plataforma de automação e auditoria fiscal que lê arquivos SPED, cruza os dados com uma matriz de regras tributárias e aponta inconsistências antes que elas virem autuação. Também simula o impacto da Reforma Tributária (2027–2033) sobre a carga de cada cliente.

## Arquitetura

```text
 Arquivos SPED (EFD ICMS/IPI, EFD Contribuições)
            │
            ▼
   ┌─────────────────┐     ┌──────────────────────┐
   │  Parser SPED    │────▶│  Supabase (Postgres) │
   │  (Python)       │     │  produtos, auditoria │
   └─────────────────┘     └──────────┬───────────┘
            │                         │
            ▼                         ▼
   ┌─────────────────┐     ┌──────────────────────┐
   │ Matriz de Regras│────▶│  Dashboard (HTML/JS) │
   │ (CST, NCM, CFOP)│     │  KPIs, CSV, PDF      │
   └─────────────────┘     └──────────────────────┘
            ▲
   ┌─────────────────┐
   │ Reforma         │
   │ Tributária      │
   │ 2027–2033       │
   └─────────────────┘
```

## Módulos

| Módulo | Função |
|---|---|
| **Parser SPED** | Lê e valida os registros dos arquivos SPED, normaliza produtos e documentos e grava no banco. |
| **Matriz de Regras** | Confronta CST, NCM, CFOP e alíquotas com as regras vigentes e gera as inconsistências. |
| **Reforma Tributária 2027–2033** | Compara a carga atual (ICMS/ISS/PIS/Cofins) com a transição para CBS e IBS, ano a ano. |
| **Dashboard** | Painel executivo com KPIs, tabela de inconsistências (exportável em CSV) e gráfico do comparador, com impressão/PDF. |

## Stack tecnológica

- **Python** — parser SPED e motor de regras
- **Supabase (PostgreSQL)** — armazenamento e API
- **HTML5 / JavaScript** — dashboard, sem dependências externas

## Estrutura do repositório

```text
.
├── README.md
├── .gitignore
├── docs/
│   └── projeto_executivo.md      # engenharia reversa, DDL e diretrizes
└── src/
    └── dashboard/
        └── index.html            # dashboard executivo
```

## Como visualizar o dashboard

Abra [src/dashboard/index.html](src/dashboard/index.html) em qualquer navegador. Os dados exibidos são de exemplo e devem ser substituídos pela consulta ao Supabase.

## Documentação

Detalhes técnicos, DDL do banco e diretrizes operacionais em [docs/projeto_executivo.md](docs/projeto_executivo.md).
