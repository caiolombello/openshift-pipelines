# OpenShift Pipelines

## Task

As Tasks são os blocos de construção de uma pipeline e consistem em etapas executadas sequencialmente. É essencialmente uma função de entradas e saídas. Uma tarefa pode ser executada individualmente ou como parte do pipeline. As tasks são reutilizáveis e podem ser usadas em vários pipelines.

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