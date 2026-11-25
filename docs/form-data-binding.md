# Binding e persistência de forms tradicionais

Este documento descreve como fields de um Form Design tradicional são ligados
às entities da plataforma e como o save padrão persiste mestre e detalhes. Não
se aplica a Forms V2/Vue.

## Princípio do binding

Um form pode estar associado a uma entity principal. Cada `FormField` pode
estar ligado a um atributo dessa entity ou de outra entity:

```xml
<field name="customer-name"
       type="TEXT"
       entity="CRM_CUSTOMER"
       entity-field="DS_NAME"
       visible="true">
  <properties>
    <property name="id">customer-name</property>
  </properties>
</field>
```

O XML usa nomes físicos. Durante a importação,
`FormFieldBusinessDelegate` resolve `entity` e `entity-field` no cache de
entities e persiste seus IDs internos. IDs de entity e atributo não devem ser
escritos manualmente no XML.

O atributo HTML `name` gerado para inputs vem de `FormField.name`, não
necessariamente do nome da coluna. No POST, o backend encontra novamente o
`FormField` pelo nome recebido e só então determina a entity e a coluna de
destino.

Fields sem `entity`/`entity-field`, ou tipos puramente visuais, podem participar
da interface sem entrar no save automático.

## Save no frontend

O runtime tradicional define `window.dynaform.save(form, opts)` em
`web-inpaas/src/main/webapp/web/js/inpaas-default.js`.

O fluxo:

1. lê `form-key` e o input marcado com `record-id`;
2. verifica `data-permission-insert` ou `data-permission-update`;
3. desabilita inputs de template de datatable para não validá-los;
4. executa validação jQuery quando disponível;
5. marca os inputs de template com `dynaform-template-field`;
6. descarrega editores HTML para sincronizar seus valores;
7. usa o atributo `action` do próprio `<form>` como URL;
8. envia `POST` com `serialize()` ou `FormData` quando há upload;
9. dispara o evento `aftersave` com a resposta.

O `action` é o `postpath` produzido pelo template, normalmente a mesma rota que
abriu o form:

```text
POST /forms/{form-key}
POST /forms/{form-key}/{record-id}
```

Sem record ID, o controller exige `INSERT`; com record ID, exige `UPDATE`.

## Entrada no backend

`FormController.doPost()` obtém o delegate pela
`DynaFormBusinessDelegateFactory` e chama:

```java
delegate.set(requestData, recordId)
```

Sem source Nashorn vinculado, é usado `DynaFormBusinessDelegate`. Com pattern
vinculado, a factory executa o source e materializa uma especialização desse
delegate.

O método `set()` primeiro chama `parseFormData()`. Para cada parâmetro postado:

1. localiza o field com `form.getFormFieldPost(name)`;
2. ignora parâmetros sem field ou sem binding completo;
3. converte o valor conforme tipo do componente e field format;
4. separa dados da entity principal e de entities filhas.

Valores vazios viram `null`. Upload, comentário e fields formatados possuem
conversores específicos.

## Dados mestre

Se o `entityId` do field é igual ao da entity principal do form, o valor entra
diretamente no `TO` mestre usando `entity-field` como chave:

```text
campo HTML customer-name
    -> FormField customer-name
    -> entity CRM_CUSTOMER
    -> entity-field DS_NAME
    -> maindata.DS_NAME
```

Depois de `beforeSet`, a PK da entity principal recebe o record ID, quando
existente, e `save(parentEntity, maindata)` decide entre insert e update. Em um
insert, a PK gerada é lida do próprio `maindata`.

O retorno recebe `record-id` e passa por `afterSet` antes de voltar ao
frontend.

## Dados mestre-detalhe

Quando um field aponta para outra entity, `parseFormData()` agrupa os valores
em:

```text
children: Map<entityId, List<TO>>
```

Inputs repetidos, como linhas de datatable, são alinhados pelo índice de suas
listas. Cada item representa um registro detalhe. O mapa usa o ID interno da
entity, embora o XML continue usando seu nome físico.

Após salvar o mestre, para cada entity filha o delegate:

1. confirma que existe referência entre detalhe e mestre;
2. carrega os detalhes atualmente persistidos;
3. remove a primeira linha quando ela representa apenas o template da tabela;
4. identifica updates pela PK do detalhe;
5. adiciona a PK do mestre ao item filho;
6. executa `save(childEntity, childdata)` para cada item recebido;
7. remove registros antigos que não permaneceram na coleção enviada.

Se a única linha contém apenas `dynaform-template-field`, os detalhes atuais
são removidos. Se a entity filha nem aparece em `children`, ela não entra nesse
ciclo.

O relacionamento é obtido do modelo de entities, por `EntityReference`. O
fluxo reconhece referência do detalhe para o mestre e possui tratamento para
referência inversa. A modelagem da FK precisa estar correta na plataforma;
proximidade de nomes de tabelas não é suficiente.

Fields de uma entity secundária não viram detalhe automaticamente apenas por
estarem dentro de um container visual. O que determina o agrupamento é o
binding `entity`/`entity-field` e a referência declarada entre as entities.

### Detalhe 1:1 exibido junto ao mestre

Para dados complementares de um único registro filho por mestre — por exemplo,
uma ficha de funcionário de uma pessoa ou um cabeçalho de pedido de uma
ocorrência — usar o mecanismo mestre-detalhe padrão. Não criar uma chamada Ajax
no `blur` para carregar ou gravar esses campos: eles devem ser enviados no POST
normal do form juntamente com o mestre.

Pré-requisitos:

- a entity filha tem uma FK para a entity mestre;
- a cardinalidade 1:1 é garantida por índice único na FK da filha;
- o form contém fields com `entity` e `entity-field` da filha; e
- a PK técnica da filha está presente como `HIDDEN` ligado à sua coluna PK.

O hidden permite que o parser identifique um registro já existente como update,
em vez de tentar inserir outro detalhe. O nome lógico e o `id` HTML são livres,
mas devem ser estáveis e únicos; os nomes físicos da entity e das colunas são o
contrato do binding. Exemplo para `CRM_CASE` e a filha `CST_CASE_PEDIDO`:

```xml
<field entity="CST_CASE_PEDIDO" entity-field="ID_CST_CASE_PEDIDO"
       type="HIDDEN" fixed="false"
       name="hdn-cst_case_pedido-id" visible="false">
  <dependency/><hardcode/>
  <properties>
    <property name="id">hdn-cst_case_pedido-id</property>
  </properties>
</field>

<field entity="CST_CASE_PEDIDO" entity-field="DS_NUMERO_PEDIDO"
       type="TEXT" fixed="false" name="case-order-number" visible="true">
  <dependency/><hardcode/>
  <properties>
    <property name="id">case-order-number</property>
    <property name="class">form-control input-sm</property>
    <property name="readonly">Y</property>
  </properties>
</field>

<field entity="CST_CASE_PEDIDO" entity-field="DT_DATA_ENTREGA"
       type="TEXT" fixed="false" name="case-delivery-date"
       format="fieldformat.date" visible="true">
  <dependency/><hardcode/>
  <properties>
    <property name="id">case-delivery-date</property>
    <property name="class">form-control input-sm</property>
    <property name="readonly">Y</property>
  </properties>
</field>
```

Use `readonly`, e não `disabled`, quando a interface só deve liberar a edição
após uma ação como **Editar**. Inputs desabilitados não participam de
`serialize()` e, portanto, não seguem no save padrão. A strategy pode trocar o
estado visual com `.prop('readonly', false)`; ela não deve salvar o detalhe.

O registro filho não é preenchido automaticamente pelo binding na abertura. No
`beforeRender` do Business Delegate ou da business strategy, consultar a filha
pela FK do mestre e inserir no `data` a PK e todos os atributos usados pelos
fields. Em strategies despachadas por `plusoftcrm.libs.pattern.strategy`, o
hook recebe um único objeto com `renderContext` e `data`:

```javascript
function beforeRender(args) {
  var data = args.data;
  var order;

  if (args.renderContext.getFormField() || !data) return;

  order = daoFactory.getDao('CST_CASE_PEDIDO')
    .filter('id_case')
    .equalsTo(data.get('id_case'))
    .findFirst();

  if (!order) return;

  data.put('id_cst_case_pedido', order.id_cst_case_pedido);
  data.put('ds_numero_pedido', order.ds_numero_pedido);
  data.put('dt_data_entrega', order.dt_data_entrega);
}

scope.beforeRender = beforeRender;
```

Ignorar renderização de field isolado evita consultas repetidas. Se não houver
filho, não preencher a PK: no primeiro save o delegate criará o detalhe e
injetará a FK do mestre conforme a referência modelada. Nunca usar nomes de
colunas copiados de outro ambiente; consultar a entity baixada e usar seus
nomes físicos reais.

## Hooks do DynaFormBusinessDelegate

Os hooks mais usados são:

- `beforeSet(data, id)`: recebe os dados já separados em mestre e `children`, antes da persistência;
- `afterSet(data)`: recebe o mestre salvo com `record-id`;
- `beforeRender(context, data)`: executa antes de renderizar fields em modos diferentes de `DESIGN`;
- `afterRender(context, formData, designerHtml)`: pode alterar o HTML gerado;
- `beforeList(formField, fixedFilter, criteria)`: ajusta uma listagem antes da query;
- `afterList(formField, data)`: transforma a coleção retornada.

`beforeSet` é o ponto para completar, transformar ou reorganizar `children`
quando o binding automático não expressa a regra necessária. Ele não deve ser
usado para repetir manualmente um mapeamento que fields e referências já
resolvem.

Na versão `2.7.67-dev`, `beforeSet` deve mutar o `TO` recebido. O método Java
atribui o retorno do hook a uma variável, mas continua persistindo o objeto
`maindata` original. Retornar um novo objeto sem modificar o argumento não
substitui efetivamente os dados usados pelo save nessa versão.

### Validações gerais antes de gravar uma Ocorrência

Em uma strategy que estende `plusoftcrm.case.forms.crm_ticket.crm_case.edit`,
as validações gerais que podem bloquear a gravação da Ocorrência devem ser
declaradas na função `beforeSaveCase(event)`. Ela é acionada diretamente pelo
botão de salvar, antes do handler delegado do form-base; quando a regra falhar,
deve chamar `event.preventDefault()` e `event.stopImmediatePropagation()`,
apresentar uma mensagem acionável e devolver `false`.

Não associar essa regra a `change` ou `blur` de um campo específico, como a data
de conclusão: isso mistura a validação de fechamento com a edição do campo e
pode alterar valores antes da tentativa efetiva de salvar. A função deve
permitir o save quando a condição não se aplica e focar o primeiro campo inválido
quando bloquear a operação.

```javascript
function beforeSaveCase(event) {
  if (!mustBlockSave()) return true;

  event.preventDefault();
  event.stopImmediatePropagation();
  $.notify('Corrija os dados obrigatórios antes de salvar.', {
    autoHide: false,
    className: 'error'
  });
  scope.find(firstInvalidSelector()).focus();
  return false;
}

scope.find('button[role=case-save]')
  .off('click.strategyBeforeSaveCase')
  .on('click.strategyBeforeSaveCase', beforeSaveCase);
```

Em Nashorn, é comum o source retornar uma extensão de
`DynaFormBusinessDelegate`, recebendo `scriptContext` e o binding global
`form`. Ao modificar esses hooks, preservar o contrato `TO`/coleções Java e os
IDs internos usados como chaves de `children`.

## Telas de configuração com registro único

Uma tela de configuração única abre diretamente o form de edição a partir do
menu; ela não usa listagem para escolher registros. O padrão se aplica somente
quando o produto define que a tabela possui uma única configuração lógica.

O form deve estar vinculado a um source de Business Delegate já cadastrado no
Studio. A implementação carrega o registro existente em `beforeRender` e
sobrescreve `set(data, id)` para encaminhar a gravação com a PK encontrada.
`beforeSet` sozinho não é suficiente para escolher insert ou update, pois a
decisão de persistência usa o `id` recebido por `set`.

```javascript
(function () {
  'use strict';

  var DynaFormBusinessDelegate = Java.extend(
    Java.type('br.com.inpaas.forms.businessdelegate.DynaFormBusinessDelegate')
  );
  var daoFactory = require('inpaas.core.entity.dao');
  var ENTITY = 'CST_CONFIG_EXAMPLE';
  var PRIMARY_KEY = 'id_cst_config_example';
  var instance;
  var parent;

  function getConfiguration() {
    return daoFactory.getDao(ENTITY).findFirst();
  }

  function beforeRender(renderContext, data) {
    var record = getConfiguration();
    if (record) {
      data.put(PRIMARY_KEY, record[PRIMARY_KEY]);
      data.put('ds_example', record.ds_example);
    }
    return data;
  }

  function set(data, id) {
    var record = getConfiguration();
    if (record) {
      id = record[PRIMARY_KEY];
    }
    return parent.set(data, id);
  }

  instance = new DynaFormBusinessDelegate(scriptContext, form) {
    beforeRender: beforeRender,
    set: set
  };
  parent = Java.super(instance);
  return instance;
})();
```

Regras obrigatórias:

- Criar `instance` antes de chamar `Java.super(instance)`; obter o super antes
  da instância existir deixa o delegate inválido.
- Usar a PK efetivamente retornada da tabela. Nunca substituir o `id` por uma
  constante como `1`: sequences, migrações ou registros já existentes não
  garantem esse valor.
- Quando não houver registro, manter o `id` recebido vazio para que o primeiro
  salvamento faça o insert; nas gravações seguintes, usar a PK existente para
  fazer update.
- Copiar para `data` todos os campos que o form precisa renderizar, incluindo a
  PK, usando as chaves expostas pelo DAO.
- No design, manter somente Salvar, com retorno ao próprio form. Remover ações
  de excluir, duplicar, salvar como novo e navegação para listagem.
- Restringir a permissão de exclusão do form. A remoção do botão não protege
  uma chamada HTTP `DELETE` feita diretamente.

Esse delegate protege o fluxo do form, mas não substitui uma restrição de banco
quando a unicidade precisa valer também para integrações, SQL administrativo ou
outros pontos de gravação concorrentes. Nesses casos, modelar uma garantia de
unicidade na entity antes de publicar a solução.

## Metadados necessários para criar um field

Para atender com segurança a um pedido como “crie um campo do tipo X que grava
em TABELA.COLUNA”, é preciso conhecer:

- chave do form e sua entity principal;
- nome físico da entity alvo;
- nome físico do atributo;
- tipo, tamanho e escala;
- obrigatoriedade e valor default;
- PK da entity;
- domínio do atributo, quando houver;
- field format aplicável;
- referência FK, query FK e entity referenciada;
- se o valor pertence ao mestre ou a um detalhe;
- cardinalidade esperada para detalhe;
- componente visual e propriedades desejadas;
- posição na árvore/layout do XML.

Para um field simples da entity principal, entity, atributo e tipo geralmente
bastam para uma primeira proposta. `required`, domínio, referência e formato
podem mudar o componente correto: por exemplo, texto, checkbox, combobox,
combobox FK, data ou numeral.

Para mestre-detalhe, é obrigatório confirmar PKs e referências entre as duas
entities. Também é necessário examinar o componente que produzirá arrays no
POST, normalmente uma datatable ou lógica frontend específica.

## Fontes de metadados

Há três fontes com finalidades diferentes:

- autocomplete SQL do plugin: lista tabelas e colunas, mas atualmente descarta metadados ricos;
- XML de entity baixado pela extensão: contém atributos, tipos, PK, domínios e referências e é a melhor evidência local;
- cache de entities da plataforma: `InstanceContext.getInstance().getCache()` expõe o `DatabaseEntity` efetivamente usado pelo runtime.

O comando `inPaaS: Baixar entity` deve ser usado para materializar localmente a
entity alvo antes de editar bindings complexos. Para mestre-detalhe, baixar
tanto a entity mestre quanto a detalhe.

Se esse fluxo ficar frequente, `inpaas.studio.vscode.utils` pode expor um método
de leitura de metadados por nome, retornando apenas dados necessários ao design:

```json
{
  "name": "CRM_CUSTOMER",
  "primaryKey": "ID_CUSTOMER",
  "fields": [
    {
      "name": "DS_NAME",
      "type": "VARCHAR",
      "size": 200,
      "required": true,
      "defaultValue": null,
      "domains": []
    }
  ],
  "references": []
}
```

Esse método seria uma conveniência para o plugin e não substituiria o cadastro
de entities. Antes de criá-lo, validar se o XML de entity já cobre o fluxo real
de trabalho sem custo excessivo.

## Checklist para alteração assistida

Antes de adicionar ou mudar um field com persistência:

1. carregar o XML do form e identificar sua entity principal;
2. carregar a definição da entity alvo;
3. confirmar que o atributo existe e é gravável;
4. escolher `FormFieldType` compatível com tipo, domínio e referência;
5. definir `name` único e binding físico correto;
6. verificar propriedades frontend necessárias;
7. se for detalhe, confirmar FK, PK, cardinalidade e formato do POST;
8. examinar o source Nashorn do form para hooks que alterem os dados;
9. preservar fields, filtros e fragmentos não relacionados;
10. testar insert, update e, para detalhes, remoção de linhas.
