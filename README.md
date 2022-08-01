# OpenShift Pipelines

## Pré-Requisitos

- Openshift 4 cluster
- [OpenShift Pipelines Operator](https://github.com/openshift/pipelines-tutorial#install-openshift-pipelines)
- [OpenShift command-line interface (oc)](https://docs.openshift.com/container-platform/4.10/cli_reference/openshift_cli/getting-started-cli.html)
- [Tekton CLI](https://github.com/tektoncd/cli#installing-tkn)

## Task

As Tasks são os blocos de construção de uma pipeline e consistem em etapas executadas sequencialmente. É essencialmente uma função de entradas e saídas. Uma tarefa pode ser executada individualmente ou como parte do pipeline. 

As tasks são reutilizáveis e podem ser usadas em várias pipelines.

![Tasks](/images/task.png)

### Exemplo de uma task:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
    name: deploy-to-my-awesome-cloud
spec:
    params:
        - name: api-url
          default: cloud.com
    steps:
        - name: deploy-app
          image: foo/base-image:2.7
          script: |
            cloud login -a $(params.api-url)
```

#### Expressão When

A expressão When protege a execução de tarefas definindo critérios para a execução de tarefas em uma pipeline. Elas contêm uma lista de componentes que permite que uma tarefa seja executada somente quando determinados critérios forem atendidos. Expressões When também têm suporte no conjunto final de tarefas especificadas usando o campo finally no arquivo YAML do pipeline.

Os componentes chaves da expressão When são:

- input: Especifica entradas ou variáveis estáticas, como um parâmetro, resultado da tarefa e status de execução. Você deve inserir uma entrada válida. Se você não inserir uma entrada válida, seu valor padrão será uma string vazia.
- operator: Especifica o relacionamento de uma entrada com um conjunto de valores. Digite in ou notin como seus valores de operador.
- values: Especifica uma matriz de valores de string. Insira uma matriz não vazia de valores estáticos ou variáveis, como parâmetros, resultados e um estado vinculado de um espaço de trabalho.

As expressões When declaradas são avaliadas antes que a tarefa seja executada. Se o valor de uma expressão When for True, a tarefa será executada. Se o valor de uma expressão When for False, a tarefa será ignorada.

Você pode usar as expressões When em vários casos de uso. Por exemplo, se:

- O resultado de uma tarefa anterior é o esperado.
- Um arquivo em um repositório Git foi alterado nos commits anteriores.
- Existe uma imagem no registro.
- Um workspace opcional está disponível.

O exemplo a seguir mostra as expressões When de uma pipeline que são executadas. A execução da pipeline executará a tarefa create-file somente se os seguintes critérios forem atendidos: o parâmetro de caminho for README.md e a tarefa echo-file-exists será executada somente se o resultado existente da tarefa de check-file for yes.

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun 
metadata:
  generateName: guarded-pr-
spec:
  serviceAccountName: 'pipeline'
  pipelineSpec:
    params:
      - name: path
        type: string
        description: The path of the file to be created
    workspaces:
      - name: source
        description: |
          This workspace is shared among all the pipeline tasks to read/write common resources
    tasks:
      - name: create-file 
        when:
          - input: "$(params.path)"
            operator: in
            values: ["README.md"]
        workspaces:
          - name: source
            workspace: source
        taskSpec:
          workspaces:
            - name: source
              description: The workspace to create the readme file in
          steps:
            - name: write-new-stuff
              image: ubuntu
              script: 'touch $(workspaces.source.path)/README.md'
      - name: check-file
        params:
          - name: path
            value: "$(params.path)"
        workspaces:
          - name: source
            workspace: source
        runAfter:
          - create-file
        taskSpec:
          params:
            - name: path
          workspaces:
            - name: source
              description: The workspace to check for the file
          results:
            - name: exists
              description: indicates whether the file exists or is missing
          steps:
            - name: check-file
              image: alpine
              script: |
                if test -f $(workspaces.source.path)/$(params.path); then
                  printf yes | tee /tekton/results/exists
                else
                  printf no | tee /tekton/results/exists
                fi
      - name: echo-file-exists
        when: 
          - input: "$(tasks.check-file.results.exists)"
            operator: in
            values: ["yes"]
        taskSpec:
          steps:
            - name: echo
              image: ubuntu
              script: 'echo file exists'
...
      - name: task-should-be-skipped-1
        when: 
          - input: "$(params.path)"
            operator: notin
            values: ["README.md"]
        taskSpec:
          steps:
            - name: echo
              image: ubuntu
              script: exit 1
...
    finally:
      - name: finally-task-should-be-executed
        when: 
          - input: "$(tasks.echo-file-exists.status)"
            operator: in
            values: ["Succeeded"]
          - input: "$(tasks.status)"
            operator: in
            values: ["Succeeded"]
          - input: "$(tasks.check-file.results.exists)"
            operator: in
            values: ["yes"]
          - input: "$(params.path)"
            operator: in
            values: ["README.md"]
        taskSpec:
          steps:
            - name: echo
              image: ubuntu
              script: 'echo finally done'
  params:
    - name: path
      value: README.md
  workspaces:
    - name: source
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 16Mi
```

#### Finally tasks

As finally tasks são o conjunto final de tarefas especificadas usando o campo finally no arquivo YAML da pipeline. Uma tarefa finally sempre executa as tarefas na pipeline, independentemente das execuções da pipeline serem executadas com êxito. As finally tasks são executadas em paralelo depois que todas as tarefas da pipeline são executadas, antes que a pipeline correspondente seja encerrada.

Pode se configurar uma finally task para consumir os resultados de qualquer tarefa na mesma pipeline. Essa abordagem não altera a ordem na qual essa finnaly task é executada. Ele é executado em paralelo com outras finnaly tasks após todas as tasks que não são finally serem executadas.

O exemplo a seguir mostra um trecho de código da pipeline clone-cleanup-workspace. Esse código clona o repositório em um workspace compartilhado e limpa o workspace. Depois de executar as tasks da pipeline, a task de limpeza especificada na seção finally do arquivo YAML da pipeline limpa o workspace.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: clone-cleanup-workspace 
spec:
  workspaces:
    - name: git-source 
  tasks:
    - name: clone-app-repo 
      taskRef:
        name: git-clone-from-catalog
      params:
        - name: url
          value: https://github.com/tektoncd/community.git
        - name: subdirectory
          value: application
      workspaces:
        - name: output
          workspace: git-source
  finally:
    - name: cleanup 
      taskRef: 
        name: cleanup-workspace
      workspaces: 
        - name: source
          workspace: git-source
    - name: check-git-commit
      params: 
        - name: commit
          value: $(tasks.clone-app-repo.results.commit)
      taskSpec: 
        params:
          - name: commit
        steps:
          - name: check-commit-initialized
            image: alpine
            script: |
              if [[ ! $(params.commit) ]]; then
                exit 1
              fi
```

### TaskRun

A task é executada usando o TaskRun

```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
    generateName: deploy-to-my-awesome-cloud-
spec:
    params:
        - name: api-url
          value: gcp.com
    taskRef:
        - name: deploy-to-my-awesome-cloud
```

## Step

Steps são uma série de comandos que são executados sequencialmente pela task e atingem um objetivo específico, como construir uma imagem. Cada task é executada como um pod e cada step é executado como um contêiner dentro desse pod. Como as steps são executadas no mesmo pod, elas podem acessar os mesmos volumes para armazenar em cache, mapas de configuração e segredos.

![Steps](/images/steps.png)

### Exemplo de um step:

```yaml
steps:
    - name: deploy-app
      image: foo/base-image:2.7
      env:
        - name: API_KEY
          valueFrom:
            secretKeyRef:
                name: secure-properties
                key: apikey
      script: |
        cloud login -a $(params.api-url)
```

## Pipeline

Em uma visão geral, a pipeline seria dessa forma, passando por tasks e tasks passando por steps.

![Steps](/images/pipeline.png)

Tasks em uma pipeline compartilham dados através de resultados e workspaces. 

Se o dado a ser compartilhado dentre as tasks for pequeno, ele pode ser escrito como resultado, caso contrário, ele pode ser escrito através de um workspace, que é um Persistent Volume Claim (PVC) compartilhado. 

O Tekton também provisiona isolamento de tasks e steps.

### Exemplo de uma pipeline:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
    name: project-pipeline
spec:
    params:
        - name: api-url
        - name: cloud-region
    tasks:
        - name: clone
          taskRef:
            name: git-clone
        - name: build
          taskRef:
            name: build
          runAfter:
            - clone
        - name: deploy
          taskRef:
            name: deploy
        runAfter:
            - build
```

### PipelineRun

A pipeline é executada usando o PipelineRun

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
    generateName: project-pipeline-run-
spec:
    params:
        - name: api-url
          value: gcp.com
        - name: cloud-region
          value: us-east
    pipelineRef:
        - name: project-pipeline
```

## Tekton Hub

Catálogo de recursos compartilhados pela comunidade que podem ser reutilizados

https://hub.tekton.dev/

## Tasks Customizadas

Uma task que roda varios tipos de testes baseados em um parametro:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
    name: testtask
spec:
    params:
        - name: test-type
          type: string
    steps:
        - name: run-test
          image: docker.hub/...
          args: ["$(params.test-type)"]
```

Para rodar essa task, pode se usar uma task customizada de loop, que recebe um parâmetro de iteração com todos os tipos de testes.

```yaml
apiVersion: custom.tekton.dev/v1alpha1
kind: TaskLoop
metadata:
    name: testloop
spec:
    taskRef:
        name: testtask
    iterateParam: test-type
```

Então, você pode especificar um Run para executar a análise do código de unidade e os testes end-to-end.

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Run
metadata:
    generateName: testloop-run-
spec:
    params:
        - name: test-type
          value: 
          - codeanalysis
          - unittests
          - e2etests
    ref:
        apiVersion: custom.tekton.dev/v1alpha1
        kind: TaskLoop
        name: testloop
```

Executando esse run, criará três tasks runs, um task run para cada tipo de task.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
    name: pipeline-with-tests
spec:
    tasks:
        - name: test-selector
          taskSpec:
            results:
                - name: listoftests
                  description: the list of tests to execute
            steps:
                - name: producer
                  image: busybox
                  script: |
                   # Some logic to select tests
                   echo "codeanalysis" > $(result.listoftests.path)
                   echo "unittests" >> $(result.listoftests.path)
        - name: loop
          taskRef:
            apiVersion: custom.tekton.dev/v1alpha1
            kind: TaskLoop
            name: testloop
          params:
            - name: test-type
              value: $(tasks.test-selector.results.listoftests)
```

## Triggers

Pode se executar a pipeline manualmente, mas para automatizar a chamada da pipeline, como quando se da o push em um commit de código ou faz um pull request, pode-se usar um trigger.

![Steps](/images/trigger.png)

O trigger tem três componentes: Trigger Binding, Trigger Template e Event Listener.

### Trigger Binding

É um recurso customizado que extrai a informação de um evento do payload do evento.

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
    name: trigger-binding
spec:
    params:
        - name: git-repo-url
          value: $(event.repository.git_http_url)
```

### Trigger Templates

É um recurso customizado que fornece um plano para criação de um outro recurso como uma linha de uma task ou uma pipeline run.

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
    name: trigger-template
spec:
    resourcetemplates:
        - apiVersion: tekton.dev/v1beta1
          kind: PipelineRun
          spec:
            pipelineRef:
                name: project-pipeline
```

### Event Listener

O Event Listenerconecta o trigger binding ao trigger template.

É um objeto do Kubernetes que ouve eventos e especifica triggers que tem trigger bindings e trigger templates.

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
    name: event-listener
spec:
    triggers:
      - name: event-trigger
        bindings:
            - name: trigger-binding
        template:
            name: trigger-template
```

## Autenticação Git

Uma execução de pipeline ou task pode exigir várias autenticações para acessar diferentes repositórios Git. Anote cada secret com os domínios em que as Pipelines possam usar suas credenciais.

Uma chave de anotação de credencial para secret do Git deve começar com tekton.dev/git-, e seu valor é a URL do host para o qual você deseja que as Pipelines usem essa credencial.

1. No exemplo a seguir, as Pipelines usam um segredo de autenticação básica, que depende de um nome de usuário e senha, para acessar repositórios em github.com e gitlab.com.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: basic-user-pass
  annotations:
    tekton.dev/git-0: github.com
    tekton.dev/git-1: gitlab.com
type: kubernetes.io/basic-auth
stringData:
  username: 
  password: 
```

Você também pode usar uma secret ssh-auth (chave privada) para acessar um repositório Git.

```yaml
apiVersion: v1
kind: Secret
metadata:
  annotations:
    tekton.dev/git-0: https://github.com
type: kubernetes.io/ssh-auth
stringData:
  ssh-privatekey: 
```

2. No arquivo serviceaccount.yaml, associe o segredo à conta de serviço apropriada.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-bot 
secrets:
  - name: basic-user-pass 
```

3. No arquivo run.yaml, associe a conta de serviço a uma execução de tarefa ou de pipeline.

   - Associe a conta de serviço a uma execução de tarefa:

    ```yaml
    apiVersion: tekton.dev/v1beta1
    kind: TaskRun
    metadata:
    name: build-push-task-run-2 
    spec:
    serviceAccountName: build-bot 
    taskRef:
        name: build-push 
    ```

    - Associe a conta de serviço a um recurso PipelineRun:

    ```yaml
    apiVersion: tekton.dev/v1beta1
    kind: PipelineRun
    metadata:
    name: demo-pipeline 
    namespace: default
    spec:
    serviceAccountName: build-bot 
    pipelineRef:
        name: demo-pipeline 
    ```

4. Apply the changes.

```shell
oc apply --filename secret.yaml,serviceaccount.yaml,run.yaml
```

## [Deploy de uma aplicação de exemplo](https://github.com/caiolombello/openshift-pipelines/blob/main/EXAMPLE.md) 

## Referências

https://tekton.dev/docs/

https://docs.openshift.com/container-platform/4.10/cicd/

https://github.com/openshift/pipelines-tutorial

https://youtu.be/adFl-4A4fX4