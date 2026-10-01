# Tarefas do Scheduler

Uma tarefa do Scheduler é um cadastro no Omni que define uma execução
recorrente. Ela deve apontar para um source do tipo **Scheduler Task**, cuja
implementação expõe o método `run()`.

## Consulta

Consultar o cadastro vinculado à key do source:

```http
GET /api/vs-code/scheduler/{key}
```

`key` é a key do source Scheduler Task. A consulta deve anteceder a criação
para evitar cadastros duplicados.

## Criação

Quando não houver tarefa cadastrada, criar por:

```http
POST /api/vs-code/scheduler/create
Content-Type: application/json
```

```json
{
  "task": {
    "id_module": 50,
    "ds_task": "Descrição amigável da tarefa",
    "ds_key": "modulo.exemplo.task",
    "ds_sourcekey": "modulo.exemplo.task"
  }
}
```

- `id_module`: ID do módulo ativo confirmado para o projeto.
- `ds_task`: descrição amigável exibida no Omni.
- `ds_key`: key técnica da tarefa; usar a key do source.
- `ds_sourcekey`: key do source Scheduler Task; usar a mesma key.

O cadastro da tarefa não define sua periodicidade nem a ativa. Configurar a
frequência somente após validar o source, suas configurações e o comportamento
em homologação.

## Verificação e falhas

Após criar, consultar novamente o mesmo endpoint e conferir módulo, descrição e
as duas keys. Se o endpoint responder que `getSchedulerTask` ou
`createSchedulerTask` não está exposto por `inpaas.studio.vscode.utils`, a API
está publicada sem os métodos necessários; não repetir a criação nem cadastrar
por SQL. Corrigir/publicar o source que expõe a API e repetir a consulta.
