# Documentação de Domínio

## Layout

Este projeto usa um layout de documentação de domínio em **contexto único**:

- Um `CONTEXT.md` na raiz do repositório — visão geral do código, módulos-chave, padrões e convenções
- `docs/adrs/` — Architecture Decision Records (ADRs) individuais em formato MADR

## Regras de Consumo

### CONTEXT.md

O arquivo `CONTEXT.md` deve conter:
- Visão geral de alto nível do código
- Módulos-chave e suas responsabilidades
- Padrões arquiteturais e convenções utilizadas
- Stack de tecnologia e dependências
- Como executar e testar o projeto

Skills que leem `CONTEXT.md` varrerão o arquivo para entender a estrutura do projeto antes de mergulhar em arquivos específicos.

### Architecture Decision Records (ADRs)

Cada ADR fica em `docs/adrs/` como um arquivo Markdown independente, nomeado no formato:
```
ADR-NNN-titulo-em-kebab-case.md
```

ADRs seguem o formato MADR (Markdown Architecture Decision Records) com as seções:
- **Status** — Proposed, Accepted, Deprecated, Superseded
- **Contexto** — O problema ou questão que esta decisão endereça
- **Decisão** — O que foi decidido
- **Alternativas Consideradas** — Outras opções que foram avaliadas
- **Consequências** — Consequências positivas e negativas desta decisão, com trade-off explícito

Seções adicionais em uso nos ADRs deste projeto:
- **Pontos em aberto** — questões que a decisão deixou sem resolver, quando houver
- **Fontes** — origem rastreável de cada afirmação do ADR (itens da transcrição e caminhos de arquivo reais)

Os títulos de seção são escritos em português, conforme `docs/adrs/README.md`.

Skills leem ADRs para entender decisões passadas e sua fundamentação antes de propor mudanças arquiteturais novas.

## Veja também

- `docs/agents/issue-tracker.md` — onde issues são rastreadas
