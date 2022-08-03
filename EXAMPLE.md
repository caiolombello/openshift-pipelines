# Subindo aplicação de exemplo com Openshift Pipelines

Como aplicação de exemplo iremos utilizar uma aplicação em Quarkus, o projeto [IA Análise de Preços](https://gitlab.vertigo-devops.com/vertigobr/devops/bootcamps/docker/ia-analise-de-precos).

1. Conecte-se à API do OpenShift

```shell
oc login -u kubeadmin https://api.crc.testing:6443
```

2. Crie um projeto para a aplicação de exemplo:

```shell
oc new-project ia-analise-de-precos
```

3. Crie a **Pipeline**

[*pipeline.yaml*](https://github.com/caiolombello/openshift-pipelines/blob/main/.ci/tekton/pipeline.yaml)

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: ia-analise-de-precos-ci
  namespace: ia-analise-de-precos
spec:
  workspaces: # Onde ficarão os artefatos gerados
  - name: shared-workspace  
  params:
  - name: deployment-name
    default: $(context.pipelineRun.namespace)
    description: name of the deployment to be patched
  - name: git-url
    description: url of the git repo for the code of deployment
  - name: git-revision
    default: main
    description: revision to be used from repo of the code for deployment
  - name: DOCKERFILE
    default: ./Dockerfile
    description: path to Dockerfile
  - name: IMAGE # Onde realizará o push da imagem, nesse caso, no registry interno do próprio Openshift
    default: image-registry.openshift-image-registry.svc:5000/$(context.pipelineRun.namespace)/$(context.pipelineRun.namespace):latest  
    description: image to be build from the code
  tasks:
  - name: fetch-repository
    taskRef:  # Referência de uma task pronta
      name: git-clone   # Nome da task pronta
      kind: ClusterTask # Tipo ClusterTask (Tasks localizadas em https://github.com/tektoncd/catalog/tree/main/task)
    workspaces:
    - name: output
      workspace: shared-workspace
    params:
    - name: url
      value: $(params.git-url)
    - name: subdirectory
      value: ""
    - name: deleteExisting
      value: "true"
    - name: revision
      value: $(params.git-revision)
  - name: package
    taskRef:
      name: maven
      kind: ClusterTask
    workspaces: 
      - name: maven-settings
        workspace: shared-workspace
      - name: source
        workspace: shared-workspace
    runAfter: 
      - fetch-repository
  - name: build-image
    taskRef:
      name: buildah
      kind: ClusterTask
    params:
    - name: DOCKERFILE
      value: $(params.DOCKERFILE)
    - name: IMAGE
      value: $(params.IMAGE)
    workspaces:
    - name: source
      workspace: shared-workspace
    runAfter:
    - package
```

4. Aplique a **Pipeline**

```shell
oc apply -f pipeline.yaml
```

5. Para autenticação do Git, crie uma **Secret** com os dados do usuário

[*secret.yaml*](https://github.com/caiolombello/openshift-pipelines/blob/main/.ci/tekton/secret.yaml)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-auth
  namespace: ia-analise-de-precos
  annotations:
    tekton.dev/git-0: https://gitlab.vertigo-devops.com/
type: kubernetes.io/basic-auth
stringData:
  username: 
  password: 
```

6. Aplique a Secret

```shell
oc apply -f secret.yaml
```

7. Crie um **ServiceAccount** e associe a secret

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: git-user 
secrets:
  - name: git-auth
```

8. Aplique a ServiceAccount

```shell
oc apply -f serviceaccount.yaml
```

9. Crie um **pipelineRun**, para rodar a Pipeline e associe a ServiceAccount

[*pipelineRun.yaml*](https://github.com/caiolombello/openshift-pipelines/blob/main/.ci/tekton/pipelineRun.yaml)

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: ia-analise-de-precos-run
  namespace: ia-analise-de-precos
spec:
  serviceAccountName: git-user
  pipelineRef:
    name: ia-analise-de-precos-ci
  params:
  - name: deployment-name
    value: ia-analise-de-precos-app
  - name: git-url
    value: https://gitlab.vertigo-devops.com/vertigobr/devops/bootcamps/docker/ia-analise-de-precos.git
  - name: git-revision
    value: main
  - name: DOCKERFILE
    value: ./src/main/docker/Dockerfile.jvm
  - name: IMAGE
    value: image-registry.openshift-image-registry.svc:5000/ia-analise-de-precos/ia-analise-de-precos-app:latest
  workspaces:
  - name: shared-workspace
    volumeClaimTemplate:
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 500Mi
```

10.  Aplique a PipelineRun

```shell
oc apply -f pipelineRun.yaml
```

