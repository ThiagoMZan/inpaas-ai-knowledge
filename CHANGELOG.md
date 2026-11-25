# Changelog

## Unreleased

- Base de conhecimento separada do `inpaas-dev-kit` para manutenção
  colaborativa.
- Documentado o padrão de modal declarativo em Form Design e o binding de
  detalhe 1:1 com carregamento em `beforeRender` e save padrão do form.
- Reforçada a separação entre estrutura declarada no XML, DataTables dinâmicos
  e modais em forms independentes abertos por `dynaform.modal`.
- Documentado que cada bloco funcional de uma form strategy deve usar form
  próprio, com chave `.strategy.{nomefuncional}` e nome
  `{Função Pai} | Strategy | {Nome da Função}`.
