# Red Hat OpenShift Pipelines

This is a sample tutorial to shows the main capabilities of
Red Hat OpenShift Pipelines (based in [Tekton](https://tekton.dev/)).

This tutorial walks through from the main components of this new way to
define CICD pipelines.

To follow it, the next requirements must be resolved:

* Red Hat OpenShift
* Red Hat OpenShift Pipelines operator
* [Tekton CLI](https://github.com/tektoncd/cli) (`tkn`)

If you don't have available a Red Hat OpenShift cluster, you could use
[Red Hat CodeReady Containers](https://github.com/code-ready/crc) to have
OpenShift 4 on your laptop.

**TIP**: Create a new project (e.g.) `pipelines-demo` to follow this repo:

```shell
oc new-project pipelines-demo
```

## Tasks

A `Task` is a collection of `Steps` that you define and arrange in a specific
order of execution as part of your continuous integration flow. A `Task` executes
as a Pod on your OpenShift cluster. A `Task` is available within a specific
namespace, while a `ClusterTask` is available across the entire cluster.

References:

* [TektonCD - Tasks](https://tekton.dev/docs/pipelines/tasks/)

### Sample Task

A sample task with a single step looks like similar to:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: hello-task
spec:
  steps:
    - name: say-hello
      image: registry.redhat.io/ubi7/ubi-minimal
      command:
        - /bin/bash
      args: ['-c', 'echo Hello World']
```

Create this task in OpenShift:

```shell
oc apply -f 01-hello-task.yaml
```

Start the task and show the log output:

```shell
tkn task start hello-task --showlog
```

### Parametrized Task

`Tasks` can also take parameters. This way, you can pass various flags to be
used in the `Task`. These parameters can be instrumental in making your `Tasks`
more generic and reusable across `Pipelines`.

A extended version of the previous task to ask for the name of a person looks
like similar to:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: hello-params-task
spec:
  params:
    - name: person
      description: Name of person to greet
      default: World
      type: string
  steps:
    - name: say-hello
      image: registry.redhat.io/ubi7/ubi-minimal
      command:
        - /bin/bash
      args: ['-c', 'echo Hello $(params.person)']
```

This parameter could define a default value, in case it is not defined when the
task is started.

Create the task:

```shell
oc apply -f 02-hello-params-task.yaml
```

Start the task without parameters it will force to declare them:

```shell
❯ tkn task start hello-params-task --showlog
? Value for param `person` of type `string`? (Default is `World`) World
TaskRun started: hello-params-task-run-96bw7
Waiting for logs to be available...
[say-hello] Hello World
```

`tkn` includes the `--use-param-defaults` argument to use the default values
of the parameters. The argument `--param` or `-p` sets up a different value for
the parameter:

```shell
tkn task start hello-params-task hello-params \
  -p person=Roman \
  --showlog
```

### Multiple Stepped Task

`Tasks` can have more than one `step`, allowing to specialize the task with more
detailed steps. The steps will run in the order in which they are defined in the
steps array.

The [03-hello-multisteps-task.yaml](./03-hello-multisteps-task.yaml) includes a
task with two steps to combine the execution of the task.

Create the task:

```shell
oc apply -f 03-hello-multisteps-task.yaml
```

Start the task.

```shell
tkn task start hello-multisteps-task \
  -p person=Roman \
  --showlog
```

## Pipelines

A `Pipeline` is a collection of `Tasks` that you define and arrange in a
specific order of execution as part of your continuous integration flow.
Each `Task` in a `Pipeline` executes as a `Pod` on your OpenShift cluster.
You can configure various execution conditions to fit your business needs.

In fact, tasks should do one single thing so you can reuse them across
pipelines or even within a single pipeline.

References:

* [TektonCD - Pipelines](https://tekton.dev/docs/pipelines/pipelines/)

### Sample Pipeline

A simple pipeline looks like similar to:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: say-things-pipeline
spec:
  tasks:
    - name: first-task
      params:
        - name: pause-duration
          value: "2"
        - name: say-what
          value: "Hello, this is the first task"
      taskRef:
        name: say-something-task
    - name: second-task
      params:
        - name: say-what
          value: "And this is the second task"
      taskRef:
        name: say-something-task
```

Create the task and pipeline:

```shell
oc apply -f 04-say-something-task.yaml
oc apply -f 04-say-things-pipeline.yaml
```

To start the pipeline:

```shell
tkn pipeline start say-things-pipeline --showlog
```

Or create a `PipelineRun` definition to start the pipeline:

```yaml
---
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: say-things-pipelinerun
spec:
  pipelineRef:
    name: say-things-pipeline
```

```shell
oc apply -f 04-say-something-pipelinerun.yaml
```

### Parallel Pipeline

`Tasks` will be executed in the order defined in the pipeline, or create a
sequence using `runAfter` definition.

[05-say-things-in-order-pipeline.yaml](./05-say-things-in-order-pipeline.yaml) file
has a sample of order and parallel tasks in a pipeline.

```shell
oc apply -f 05-say-things-in-order-pipeline.yaml
```

To start the pipeline:

```shell
tkn pipeline start say-things-in-order-pipeline --showlog
```

### Resources

A `Pipeline` requires `PipelineResources` to provide inputs and store outputs
for the `Tasks` that comprise it. You can declare those in the `resources`
field in the `spec` section of the `Pipeline` definition. Each entry requires
a unique name and a type.

This is a mechanism to define the pipeline as generic as we can. Using resources
the pipeline can be reused across different projects.

**NOTE**: This resource is is defined as `alpha`, so it could change in the
future. It is not recommended to used, as there are other resources available
to cover this feature (such as `Workspaces`)

This is a sample task using an input resource to count files:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: count-files-task
spec:
  resources:
    inputs:
      - name: repo
        type: git
        targetPath: code
  steps:
    - name: count
      image: registry.redhat.io/ubi7/ubi-minimal
      command:
        - /bin/bash
      args: ['-c', 'echo $(find /workspace/code -type f | wc -l) files in repo']
```

This sample pipeline declares an input resource to be used for a task:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: count-pipeline
spec:
  resources:
    - name: git-repo
      type: git
  tasks:
    - name: count-task-repo
      taskRef:
        name: count-files-task
      resources:
        inputs:
          - name: repo
            resource: git-repo
```

Create the task and the pipeline:

```shell
oc apply -f 06-count-files-task.yaml
oc apply -f 06-count-pipeline.yaml
```

A sample of `PipelineResource` to identify the resource used by a pipeline
is similar to:

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: git-repo-one
spec:
  type: git  
  params:
    - name: url
      value: https://github.com/mashrafrh/spring-trial.git
```

Create the resources for the pipeline

```shell
oc apply -f 06-git-repo-one-pipelineresource.yaml
oc apply -f 06-git-repo-two-pipelineresource.yaml
```

Start the pipeline to use the first resource:

```shell
tkn pipeline start count-pipeline \
    --resource git-repo=git-repo-one \
    --showlog
```

And start the same pipeline using the second resource:

```shell
tkn pipeline start count-pipeline \
    --resource git-repo=git-repo-two \
    --showlog
```

### Workspaces

`PipelineResources` are a great to define resources and reuse them in
different pipelines, but they could not be enough for more complex patterns.

`Workspaces` allow `Tasks` to declare parts of the filesystem that need
to be provided at runtime by `TaskRuns`. A `TaskRun` can make these parts
of the filesystem available in many ways:

* using a read-only `ConfigMap` or `Secret`,
* an existing `PersistentVolumeClaim` shared with other `Tasks`, create
a `PersistentVolumeClaim` from a provided `VolumeClaimTemplate`,
* or simply an `emptyDir` that is discarded when the `TaskRun` completes.

`Workspaces` are similar to `Volumes` except that they allow a `Task` author
to defer to users and their `TaskRuns` when deciding which class of
storage to use.

The main use cases for `Workspaces` are:

* Storage of inputs and/or outputs
* Sharing data among `Tasks`
* Mount points for configurations held in `Secrets` or `ConfigMaps`
* A cache of build artifacts that speed up jobs

References:

* [TektonCD - Workspaces](https://tekton.dev/docs/pipelines/workspaces/)

This is a sample task with a workspace:

```yaml
---
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: count-files-workspace-task
spec:
  workspaces:
    - name: source
      description: The workspace consisting of repository.
  steps:
    - name: count
      image: registry.redhat.io/ubi7/ubi-minimal
      command:
        - /bin/bash
      args: ['-c', 'echo $(find /workspace/source -type f | wc -l) files in repo']
```

[07-count-workspace-pipeline.yaml](./07-count-workspace-pipeline.yaml)
file describes a sample pipeline using a workspace across
different tasks. This example uses a `ClusterTask` to
demonstrate how to reuse these objects in a pipeline.

Create the task and the pipeline:

```shell
oc apply -f 07-count-files-workspace-task.yaml
oc apply -f 07-count-workspace-pipeline.yaml
```

As this workspace requires a storage, we need to create the `PersistentVolumeClaim`.
The [07-workspace-pvc.yaml](./07-workspace-pvc.yaml) defines it.

Create PVC to store the data:

```shell
oc apply -f 07-workspace-pvc.yaml
```

Start the pipeline requires to declare the workspace to use, among other
parameters declared in the pipeline:

```shell
tkn pipeline start count-workspace-pipeline \
    --param GIT_URL=https://github.com/mashrafrh/spring-trial.git \
    --param GIT_REVISION=master \
    --workspace name=workspace,claimName=workspace-pvc \
    --showlog
```

Start again the pipeline with other parameters:

```shell
tkn pipeline start count-workspace-pipeline \
    --param GIT_URL=https://github.com/elsony/devfile-sample-code-with-quarkus.git \
    --use-param-defaults \
    --workspace name=workspace,claimName=workspace-pvc \
    --showlog
```

## Triggers

Tekton Triggers is a Tekton component that allows you to detect and extract
information from events from a variety of sources and deterministically instantiate
and execute `TaskRuns` and `PipelineRuns` based on that information.
Tekton Triggers can also pass information extracted from events directly to
`TaskRuns` and `PipelineRuns`.

References:

* [TektonCD - Triggers](https://tekton.dev/docs/triggers/)

To show how triggers works, we will extend our previous
pipeline to be executed with a trigger when a new
change is pushed into the GitHub repository.

The main objects related with triggers are:

* `EventListener`: listens for events at a specified port on your OpenShift
cluster. Specifies one or more `Triggers` or `TriggerTemplates`.

Create our `EventListener` that uses a `TriggerTemplate`:

```shell
oc apply -f 08-count-workspace-pipeline-eventlistener.yaml
```

The `EventListener` will create a service to be used to access to. If we want
to use this service externally, we need to expose as a route:

```shell
❯ oc get svc
NAME                                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
el-count-workspace-pipeline-eventlistener   ClusterIP   172.30.134.93   <none>        8080/TCP,9000/TCP   13m
❯ oc expose svc el-count-workspace-pipeline-eventlistener
```

The new should be similar to:

```
❯ oc get route
NAME                                        HOST/PORT                                                                                                   PATH   SERVICES                                    PORT            TERMINATION   WILDCARD
el-count-workspace-pipeline-eventlistener   el-count-workspace-pipeline-eventlistener-pipelines-demo.apps.cluster-76lkr.76lkr.sandbox1545.opentlc.com          el-count-workspace-pipeline-eventlistener   http-listener                 None
```

We will use this route later to integrate in our GitHub repository as a WebHook.

* `Trigger`: specifies what happens when the `EventListener` detects an event.
A `Trigger` specifies a `TriggerTemplate`, a `TriggerBinding`, and
optionally, an Interceptor.

```shell
oc apply -f 08-count-workspace-pipeline-trigger.yaml
```

The trigger includes an [Interceptor](https://tekton.dev/docs/triggers/interceptors/),
that it is a "catch-all" event processor to perform payload filtering, to get
the details from GitHub repo.

```yaml
  interceptors:
    - ref:
        name: github
      params:
        - name: secretRef
          value:
            secretName: github-interceptor-webhook
            secretKey: secret
```

This secret will be used to add a security check to confirm that GitHub is
invoking the `EventListener` under a security context.

We need to create a `Secret` with the value to use from GitHub WebHook
to create a secured call.

```shell
oc apply -f 08-github-interceptor-webhook-secret.yaml
```

* `TriggerTemplate`: specifies a blueprint for the resource, such as a `TaskRun`
or `PipelineRun`, that you want to instantiate and/or execute when your
`EventListener` detects an event. It exposes parameters that you can use
anywhere within your resource’s template.

```shell
oc apply -f 08-count-workspace-pipeline-triggertemplate.yaml
```

* `TriggerBinding`: specifies the fields in the event payload from which you
want to extract data and the fields in your corresponding `TriggerTemplate`
to populate with the extracted values. You can then use the populated fields
in the `TriggerTemplate` to populate fields in the associated `TaskRun` or `PipelineRun`.

```shell
oc apply -f 08-count-workspace-pipeline-triggerbinding.yaml
```

The latest step is create a new GitHub WebHook in our repository using the
route exposed above and adding the `/hooks` path. 

In your GitHub repo go to `Settings -> Webhooks` and click `Add Webhook`. The
fields we need to set are:

* **Payload URL**: Your external IP Address from the route with `/hooks` path
* **Content type**: application/json
* **Secret**: Value defined in `github-interceptor-webhook` secret.

From now every time you push a new change in your repository, a new pipeline
execution will happen.

```shell
git push origin main
```

For more details about how to create a webhook, please, review this
[doc](https://docs.github.com/en/developers/webhooks-and-events/webhooks/creating-webhooks).

## ... and beyond

This repo includes a short summary of many of the main objects and capabilities of
Tekton, but it is only a surface of all the them. Please, go to the resources of
this amazing project to learn and improve your CICD pipelines in a cloud native way.

* [TektonCD - Docs](https://tekton.dev/docs/)
* [Tekton Playground](https://katacoda.com/tektoncd/scenarios/playground)
