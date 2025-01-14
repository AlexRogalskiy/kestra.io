---
order: 4
---
# Outputs

Kestra's flow can produce outputs when processing tasks. Output data is stored in execution flow context and can be handled and used by tasks next to the output producer task.

You can use outputs everywhere [variables](/docs/developer-guide/variables/) are allowed, so they can be used as next task values for iteration or conditional processing or even as extra output content.

## Using outputs

You can declare as many outputs as desired for any flow. Outputs context variables are stored following each task declaration.

Here how to use an output between tasks into a flow:

```yaml
tasks:
- id: produceOutput
  type: io.kestra.core.tasks.debugs.Return
  format: my output {{ execution.id }}
- id: use-output
  type: io.kestra.core.tasks.debugs.Echo
  format: This task display previous task output {{ outputs.produceOutput.value }}
```

In the example above the first task produces an output with the format of a yaml property. The ouput content is then used in the second task output formatting. Indeed, the `use-output` task uses the templating system <code v-pre>{{ outputs.produce-output.value }}</code> to reference the previous task output.

Using this template context variable interpolates the bracket reference with the entire output generated by the first task.

::: tip note
The `.value` in the template bracket that reaches another task's output content is a value that depends on what data is produced per value. In our case, for the **Return** task, the `value` content is filled with the output. It could be `bq_table` for another task implemented for BigQuery management. Have a look at each task documentation for specific information about what context variables are filled with ouput contents.
:::

## Dynamic variables

#### Current value
You can access the current value with <code v-pre>{{ taskrun.value }}</code> like this:

```yaml
tasks:
  - id: each
    type: io.kestra.core.tasks.flows.EachSequential
    value: '["value 1", "value 2", "value 3"]'
    tasks:
      - id: inner
        type: io.kestra.core.tasks.debugs.Return
        format: "{{task.id}} > {{taskrun.value}} > {{taskrun.startDate}}"
```

###  Specific outputs for dynamic tasks

Another more specific case for output management is the runtime generated tasks output variable. This is the case for the **EachSequential** or **EachParallel** task, which produces other tasks dynamically, depending on it's `value` property. In this case it is possible to reach each iteration output individually using the following syntax:

```yaml
id: output-sample
namespace: io.kestra.tests

tasks:
  - id: each
    type: io.kestra.core.tasks.flows.EachSequential
    tasks:
      - id: sub
        type: io.kestra.core.tasks.debugs.Return
        format: "{{ task.id }} > {{ taskrun.value }} > {{ taskrun.startDate }}"
    value: "[\"s1\", \"s2\", \"s3\"]"
  - id: use
    type: io.kestra.core.tasks.debugs.Return
    format: "Previous task produced output : {{ outputs.sub.s1.value }}"
```

Here the `outputs.1_1-produce_output.s1.a.value` reach the first `1-output` task element.

#### Previous task lookup
It is also possible to locate a special task by its `value`:
```yaml
tasks:
  - id: each
    type: io.kestra.core.tasks.flows.EachSequential
    tasks:
      - id: inner
        type: io.kestra.core.tasks.debugs.Return
        format: "{{ task.id }}"
    value: "[\"value 1\", \"value 2\", \"value 3\"]"
  - id: end
    type: io.kestra.core.tasks.debugs.Return
    format: "{{ task.id }} > {{ outputs.inner['value 1'].value }}"
```
with the format `outputs.TASKID.[VALUE].PROPERTY`. The special bracket `[]` in  `.[VALUE].` enable special chars like space (and can be remove without any special characters)

#### Lookup in current childs tasks tree

Sometimes, it can be useful to access previous outputs on the current task tree. For example, on
[EachSequential](/plugins/core/tasks/flows/io.kestra.core.tasks.flows.EachSequential.md)
you iterate for a list of value, performing the first tasks (Download a file for example) and
loading the previous files to a database.

For this, you can pass the `taskrun.value` to outputs object:
```yaml
tasks:
  - id: each
    type: io.kestra.core.tasks.flows.EachSequential
    tasks:
      - id: first
        type: io.kestra.core.tasks.debugs.Return
        format: "{{task.id}}"
      - id: second
        type: io.kestra.core.tasks.debugs.Return
        format: "{{ outputs.first[taskrun.value].value }}"
    value: "[\"value 1\", \"value 2\", \"value 3\"]"
  - id: end
    type: io.kestra.core.tasks.debugs.Return
    format: "{{task.id}} > {{outputs.second['value 1'].value}}"
```
