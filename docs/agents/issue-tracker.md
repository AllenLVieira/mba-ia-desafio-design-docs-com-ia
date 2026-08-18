# Rastreador de Issues

**Tipo de rastreador**: GitHub Issues

## Visão Geral

As issues deste projeto são rastreadas em GitHub Issues. Os skills de engenharia (`to-tickets`, `triage`, etc.) usam a CLI do GitHub (`gh`) para ler e escrever no rastreador de issues.

## Lendo do rastreador

- Listar issues: `gh issue list`
- Ver detalhes de uma issue: `gh issue view <número>`
- Buscar issues por label: `gh issue list --label <label>`

## Escrevendo no rastreador

- Criar uma issue: `gh issue create --title "..." --body "..."`
- Fechar uma issue: `gh issue close <número>`
- Adicionar uma label: `gh issue edit <número> --add-label <label>`

## Pull Requests Externos

Pull requests externos **não** são incluídos na fila de triagem — apenas issues criadas diretamente no rastreador são consideradas.

## Veja também

- `docs/agents/domain.md` — layout de documentação de domínio e regras de consumo
