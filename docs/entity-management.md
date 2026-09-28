# Entities: criação e publicação

Este documento descreve o fluxo de criação e manutenção de entities pelo
editor do kit e pelo endpoint `POST /api/vs-code/entities/publish`.

## Princípios

- Uma entity somente pode ser criada ou publicada quando o usuário solicitar
  explicitamente essa alteração.
- Usar exclusivamente o módulo ativo selecionado no kit para o workspace e
  ambiente atuais. Nunca inferir o módulo pela chave, pela entity relacionada
  ou pelo assunto da demanda.
- Antes de criar, consultar a existência da entity. Depois de criar ou alterar,
  baixar o modelo retornado pelo Studio e validar os metadados persistidos.
- Não atualizar tabelas de metadados ou schema por SQL direto. A publicação usa
  o adaptador de domínio da plataforma.

## Modelo local

O arquivo local segue o nome:

```text
entities/{entity-em-minúsculas}.entity.json
```

O modelo precisa conter, no mínimo, `name`, `module` e `attributes`. A PK é um
atributo obrigatório. No modelo inicial criado pelo kit, o tipo lógico pode ser
`Long`; após o download, a plataforma pode expor o tipo físico equivalente,
como `BIGINT`.

### Ciclo de criação e atualização

O publicador chama `inpaas.core.entity.admin.setEntity(model)`, que transforma
o modelo em um `EntityPackage` e o instala pelo
`PackageInstallerBusinessDelegate`. Essa é a única rota de publicação.

- **Criação:** omitir `id`, informar `name`, `module`, a PK e todos os
  atributos iniciais.
- **Atualização:** baixar a entity primeiro, preservar seu `id` e reenviar a
  lista completa de atributos com a alteração desejada.
- **Verificação:** uma resposta de sucesso não substitui o download de
  conferência; validar o modelo efetivamente reconstruído no cache da
  plataforma.

O campo `attributes` é obrigatório e autoritativo: o `patchEntity()` limpa a
coleção atual e a recompõe a partir do payload. Portanto, nunca publicar uma
lista parcial para “adicionar apenas um campo”; isso pode remover atributos
existentes. O mesmo vale para `queries`, `references`, `indexes` e `triggers`
quando essas coleções forem enviadas: elas são substituídas. Omitir uma coleção
que não faz parte da alteração; enviar `[]` somente quando a intenção for
limpá-la.

### Atributos

O `patchEntity()` aceita os nomes lógicos de tipo usados pelo editor:

| Tipo lógico | Tipo físico publicado | Requisito |
| --- | --- | --- |
| `String` | `VARCHAR` | `size` obrigatório |
| `Unicode` | `NVARCHAR` | `size` obrigatório |
| `Char` | `CHAR` | tamanho 1; domínio quando aplicável |
| `Long` | `BIGINT` | sem tamanho/escala |
| `Integer` | `INT` | sem tamanho/escala |
| `Datetime` | `DATETIME` | sem tamanho/escala |
| `Date` | `DATE` | sem tamanho/escala |
| `Numeric` | `NUMERIC` | `size` e `scale` obrigatórios |
| `Binary` | `VARBINARY` | conteúdo longo (`size=-1`) |

Cada atributo deve informar `name`, `alias`, `required`, `searchable`,
`audit`, `moduleId` e, quando necessário, `defaultValue`, `size`, `scale` e
`domains`. A PK é marcada em seu atributo com `primaryKey: true`; o modelo usa
`primaryKey: "XPK_{ENTITY}"` como nome técnico da constraint.

Para atributos textuais, usar `VARCHAR` (ou o alias visual `String` antes da
normalização) e informar `size`. Não ativar `searchable` sem pedido explícito:
esse flag cria infraestrutura de busca full-text, não apenas capacidade de
filtro SQL.

## Entity 1:1 relacionada a Pessoa

Para dados próprios de uma strategy de Pessoa, o modelo usual é uma entity
filha com:

1. PK técnica `ID_{ENTITY}`;
2. `ID_PERSON`, do tipo compatível com `CRM_PERSON.ID_PERSON` (`BIGINT`), como
   FK para `CRM_PERSON`;
3. os campos funcionais da strategy.

Exemplo mínimo:

```text
CST_PERSON_CORPORATEDATA
├── ID_CST_PERSON_CORPORATEDATA  PK
├── ID_PERSON                    FK → CRM_PERSON.ID_PERSON
└── DS_CORPORATEDATA             VARCHAR(500), opcional
```

A FK descreve o vínculo com a pessoa. Caso a unicidade seja uma regra de banco
obrigatória — uma única linha filha por pessoa, inclusive fora do form — criar
também um índice único em `ID_PERSON`; não presumir que a FK isolada imponha
essa cardinalidade.

## Referências: contrato de publicação

O retorno de download do Studio representa uma FK na forma canônica abaixo:

```json
{
  "name": "FK_CRM_PERSON__CST_PERSON_CORPORATEDATA_ID_PERSON",
  "alias": "person",
  "referenceEntity": "CRM_PERSON",
  "fields": ["ID_PERSON"],
  "referenceFields": ["ID_PERSON"]
}
```

Contudo, o publicador nativo `inpaas.core.entity.admin.patchEntity()` não lê
essa representação. Ele espera o contrato de publicação abaixo:

```json
{
  "name": "FK_CRM_PERSON__CST_PERSON_CORPORATEDATA_ID_PERSON",
  "alias": "person",
  "entity": "CRM_PERSON",
  "field": "ID_PERSON",
  "fieldRef": "ID_PERSON"
}
```

O adapter `inpaas.studio.vscode.utils.publishEntity()` deve converter a forma
canônica para essa forma antes de chamar o serviço de entity management. O
contrato atual de `patchEntity` usa diretamente `reference.entity` para chamar
`EntityCache.get()` e `reference.field` para localizar o atributo local. Se
esses valores não existirem, ele falha com `EntityCache.get(null)`.

Para uma nova entity, construir logo o payload com `entity`, `field` e
`fieldRef`. O adaptador mantém compatibilidade com o modelo canônico salvo pelo
editor; não usar o retorno de download como payload de publicação sem essa
normalização.

## Índices, queries e triggers

Índices usam `name`, `unique`, `moduleId`, `fields` e `includes`. Cada item de
`fields` informa `name`, `direction` e, quando necessário, `nullable`; cada
item de `includes` informa `name`. Para reforçar uma relação 1:1 no banco, o
índice de `ID_PERSON` precisa de `unique: true`.

Queries usam `id` quando já existentes, `key`, `sql`, `type`, `finalVersion` e
`moduleId`. A primeira query com `type: "F"` é tratada como a query de FK pela
plataforma. Triggers são encaminhadas como objetos da coleção `triggers`.

Essas coleções não devem ser incluídas em uma alteração simples de atributos
sem antes baixar e preservar seu conteúdo. A publicação substitui a coleção
enviada.

## Labels

O serviço de entity management usa `title` da entity e `title` de cada atributo
para criar ou atualizar labels. As chaves são derivadas como
`label.{entity-em-minúsculas}` e
`label.{entity-em-minúsculas}.{atributo-em-minúsculas}`. O objeto `labels` do
arquivo local é usado pelo editor para preservar textos e pendências, mas o
publicador de domínio decide a gravação a partir desses títulos.

Definir `title` funcional para cada entity e atributo novo. Não criar labels
por SQL nem pressupor que uma chave sem texto será traduzida automaticamente.

## Publicação e validação

O payload de publicação é:

```json
{ "model": { "name": "...", "module": 0, "attributes": [] } }
```

Após uma resposta de sucesso, baixar a entity e conferir:

- PK e tipos físicos dos atributos;
- FK em `references` e no atributo `ID_PERSON`;
- tamanho e obrigatoriedade dos textos;
- módulo da entity e dos atributos;
- labels criadas ou atualizadas.

Em caso de falha, capturar o stack trace antes de tentar variações do modelo.
Identificar a camada que falhou — adaptador VS Code, service de entity ou
schema — e corrigir o contrato nessa camada. Não tentar publicar repetidamente
com formatos deduzidos.
