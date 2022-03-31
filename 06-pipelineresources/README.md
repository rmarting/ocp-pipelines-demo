# Pipeline Resources

:warning: `PipelineResources` are deprecated. :warning:

**NOTE**: This resource is is defined as `alpha`, so it could change in the
future. It is not recommended to used, as there are other resources available
to cover this feature (such as `Workspaces`)

:dart: This section is here only for old references but it will not longer maintained :dart:

A `Pipeline` requires `PipelineResources` to provide inputs and store outputs
for the `Tasks` that comprise it. You can declare those in the `resources`
field in the `spec` section of the `Pipeline` definition. Each entry requires
a unique name and a type.

This is a mechanism to define the pipeline as generic as we can. Using resources
the pipeline can be reused across different projects.

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
      value: https://github.com/rmarting/kafka-clients-quarkus-sample.git
```

Create the resources for the pipeline:

```shell
oc apply -f 06-git-repo-one-pipelineresource.yaml
oc apply -f 06-git-repo-two-pipelineresource.yaml
```

Start the pipeline to use the first resource:

```shell
❯ tkn pipeline start count-pipeline \
    --resource git-repo=git-repo-one \
    --showlog
PipelineRun started: count-pipeline-run-n8w4q
Waiting for logs to be available...
[count-task-repo : git-source-repo-6gqc6] {"level":"info","ts":1648708933.673673,"caller":"git/git.go:169","msg":"Successfully cloned https://github.com/rmarting/kafka-clients-quarkus-sample.git @ dad5aecdb1903c0f975f684e6bdedc2b1880a3e6 (grafted, HEAD) in path /workspace/code"}
[count-task-repo : git-source-repo-6gqc6] {"level":"info","ts":1648708933.7013226,"caller":"git/git.go:207","msg":"Successfully initialized and updated submodules in path /workspace/code"}

[count-task-repo : count] 117 files in repo
```

And start the same pipeline using the second resource:

```shell
❯ tkn pipeline start count-pipeline \
    --resource git-repo=git-repo-two \
    --showlog
PipelineRun started: count-pipeline-run-mvl79
Waiting for logs to be available...
[count-task-repo : git-source-repo-fhcnl] {"level":"info","ts":1648708954.7296708,"caller":"git/git.go:169","msg":"Successfully cloned https://github.com/rmarting/strimzi-migration-demo.git @ 4c877ac4f944907fe9585580ee48328e7f7758ee (grafted, HEAD) in path /workspace/code"}
[count-task-repo : git-source-repo-fhcnl] {"level":"info","ts":1648708954.771556,"caller":"git/git.go:207","msg":"Successfully initialized and updated submodules in path /workspace/code"}

[count-task-repo : count] 124 files in repo
```
