# Base de conhecimento operacional inPaaS

Este repositório é a fonte compartilhada de conhecimento e instruções
operacionais para agentes que atuam em projetos inPaaS. Ele não contém o
runtime, a extensão VS Code nem fontes de projetos.

## Antes de atuar

Leia integralmente os documentos aplicáveis em `docs/`:

- `platform.md` para contratos de sources, forms, Nashorn e execução local;
- `form-design.md` para leitura, criação ou alteração do XML de design de
  forms tradicionais;
- `form-data-binding.md` para fields ligados a entities, save e
  mestre-detalhe;
- `read-only-detail-forms.md` para fichas HTML somente leitura baseadas em
  pattern;
- `corporate-identification.md` para busca ou identificação corporativa de
  pessoas;
- `frontend-plugins.md` para HTML, CSS ou plugins JavaScript/jQuery;
- `labels.md` para criação, alteração, consulta ou remoção de labels e
  traduções;
- `import-layouts.md` para arquivos de entrada, layouts de importação ou
  mapeamento de colunas para entidades.
- `entity-management.md` para criação, alteração e publicação de entities,
  atributos, PKs, FKs, índices e labels.

Leia também o `AGENTS.md` do projeto em que o trabalho será realizado. As
regras do projeto complementam esta base, sem repetir regras reutilizáveis.

## Manutenção desta base

- Atualize aqui um comportamento reutilizável confirmado da plataforma.
- Mantenha decisões de negócio, credenciais, URLs de ambientes e dados de
  clientes fora deste repositório.
- Preserve a estrutura e os contratos existentes; mudanças incompatíveis
  devem explicar a migração no documento afetado e no `CHANGELOG.md`.
- Não transforme documentação em uma ordem para executar ações destrutivas ou
  publicar recursos sem solicitação explícita do usuário.
- Proponha um novo documento somente quando o tema não pertencer claramente a
  um documento existente.
