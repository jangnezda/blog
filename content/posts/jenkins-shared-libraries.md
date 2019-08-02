---
title: "Using Jenkins Shared Libraries"
date: 2019-06-15T19:57:50+02:00
draft: false
toc: false
images:
tags: 
  - jenkins
  - groovy
---

In recent months I've worked on multiple different projects that had to be setup for continuous integration and delivery with [Jenkins](https://jenkins.io/). The projects share similar broad patterns of what needs to be done in CI, i.e. all of them have build and test phase, as well as deployment to kubernetes cluster. However, the projects also have differences: some of them are in Phoenix/Elixir, others in React/Typescript. As such, the `Jenkinsfile` in each of the projects was similar, but not the same. I wanted to find a way to reduce the size of `Jenkinsfile` and possibly remove related files (like kubernetes pod yaml file), because a lot of it repeated in each of the projects.

In this post I'll look into [Jenkins shared libraries](https://jenkins.io/doc/book/pipeline/shared-libraries/), which are super useful for reducing the repetition and boilerplate in Jenkinsfiles. The examples are based off pipelines to deploy to Openshift managed kubernetes cluster, but I did not go into specifics of either Openshift or kubernetes.

## The problem

The problem is all the repetition and boilerplate needed to integrate a project with Jenkins build server. Consider a [declarative pipeline](https://jenkins.io/doc/book/pipeline/#declarative-versus-scripted-pipeline-syntax) that was used to build and test one of our Elixir projects (simplified for this blog post):

```groovy
pipeline {
  environment {
    NAME = 'project_name'
    // other env variables go here ...
  }

  options {
    disableConcurrentBuilds()
    timeout(time: 1, unit: 'HOURS')
    timestamps()
  }
  agent {
    kubernetes {
      label 'docker'
      defaultContainer 'jnlp'
      yamlFile 'build-pod.yaml'
    }
  }
    
  stages {
    stage('Build Image') {
      steps {
        container('docker') {
          buildImage(NAME)
        }
      }
    }

    stage('Run tests') {
      steps {
        container('openshift') {
          deployEphemeralDb('postgresql-test', 'test')
        }
        container('elixir') {
          sh "mix local.hex --force && mix local.rebar --force && mix deps.get"
          sh "MIX_ENV=test DATABASE_HOSTNAME=postgresql-test.${PROJ}.svc POSTGRESQL_USER=postgres POSTGRESQL_PASSWORD=postgres POSTGRESQL_DATABASE=test mix test"
        }
      }

      post {
        always {
          container('openshift') {
            destroyEphemeralDb('postgresql-test')
          }
          junit "_build/test/lib/project_name/test-junit-report.xml"
        }
      }
    }

    stage('Push image') {
      steps {
        container('docker') {
          tagAndPushImage(NAME, env.CHANGE_ID)
        }
      }
    }

    stage('Deploy') {
      steps {
        container('openshift') {
          deployPersistentDb(NAME, 'postgresql')
          deployApp(NAME, env.CHANGE_ID)
        }
      }
    }

    stage('Seed DB') {
      steps {
        container('openshift') {
          runDistilleryJob(NAME, 'seed')
        }
      }
    }
}

``` 

Looks a lot, but essentially it tells Jenkins:

1. That we will run our stages using kubernetes agent.
2. That we want to build docker image of our project.
3. Run some tests, which need a DB instance (postgres in our case).
4. Push the image to registry if tests suceeded.
5. Deploy the app with a Postgres database.
6. Seed the database with some data.

Most of the steps are implemented in separate functions (`buildImage`, `deployApp`, `deployPersistentDb`, ...), to make reading the pipeline a bit easier. We can immediately see that some steps will be essentially the same for most projects: set up kubernetes agent, build image, push image, deploy app. Not just that, the whole pipeline could be replicated and the custom parts, like environment variables and testing step, provided by specific project. This can be done using Jenkins Shared Libraries.

## Shared library

So, what is a Jenkins Shared Library? It's a project with pre-defined structure that can be used in a Jenkinsfile. The language used is Groovy, which makes sense as Jenkinsfiles are also in Groovy.

You can have common steps and pipelines defined in a shared library, then inside specific Jenkinsfile call those steps or whole pipelines. This sounds great and it is, but with one caveat: it's hardly usable for declarative pipelines. At this time, there are some [limitations](https://jenkins.io/doc/book/pipeline/shared-libraries/#defining-declarative-pipelines) imposed by Jenkins, such as:

- one can't break up pipelines into different parts, then combine those parts in one pipeline,
- the pipeline has to be wholly defined in one function, so it's impossible to wrap it in other functions,
- one can't extract stages into separate functions.

These limitations make it impossible to create generic pipelines. We'll have to use scripted pipelines instead. With those there aren't any limitations, so one can easily build generic stages/pipelines. Scripted pipelines are basically free-form Groovy and while they're powerful, it also means that you lose some safety compared to declarative pipelines. Additionally, Jenkins does some things automatically in case of declarative pipelines, like git check out of the project in the setup phase.

With that in mind, let's see how our shared library approach could look like:

1. The shared library should contain various resource files like the kubernetes pod yaml file (thus removing them from each of the projects).
2. The shared library should have generalized pipelines for different kinds of projects, like Elixir and Typescript.
3. The `Jenkinsfile` in a project should be quite short. In fact, it should only have information to use the shared library and specific environment variables.

## Implementation

Let's dive into shared library project. In essence, a Jenkins shared library has three directories:

```
|
`-- src          # Groovy sources/classes
|
`-- vars         # Groovy scripts exposed as variables
|
`-- resources    # Non-groovy files (xml, json, ...)
```

It may be a bit confusing to have `src` and `vars` both containing Groovy source files, but you can think of `vars` as a way to simplify the implementation - they are automatically exposed in `Jenkinsfile` as variables (essentially auto-created singletons). For our needs, we'll only use `vars` and `resources`.

If you are deploying your projects in a kubernetes cluster, then the `resources` directory could contain yaml files for deploying pods, commands, etc.

```
|
`-- resources
  |
  `-- com
    |
    `-- yourcompany
      |
      `-- jenkins
        |
        `-- build-pod.yaml
        |
        `-- postgresql-ephemeral.yaml
        |
        `-- persistent-volume.yaml
        |
        `-- ...
```

It is recommended to use Java-like package structure to avoid naming conflicts in the classpath. Now we will be able to use these two files in the common pipelines.

Next, we'll put the pipelines and steps into `vars`. I suggest to split the code into multiple files:

```
|
`-- vars
  |
  `-- pipelines.groovy
  |
  `-- stages.groovy
  |
  `-- utils.groovy
  |
  `-- ...
```

It's not necessary to do that, but it makes it easier to read and understand if all the logic isn't stuffed into one file. The scripts in `vars` will be auto-imported in other groovy files (including the Jenkinsfiles), so their functions can be called directly without additional imports, e.g. `utils.myFunction(...)`.

The overarching goal is to make Jenkinsfiles short or in other words, without specific logic. This means that the shared library will have to take care of not just one pipeline (or use case), but more of them:

1. shared library should be able to provide pipelines for different languages/frameworks,
2. shared library should take care of different rules, such as QA vs production environment or similar, because those often mean that a build/deployment is slightly different.

Let's look at `stages.groovy` and `pipelines.groovy`.

## stages.groovy

So stages should contain the common stages needed in pipelines. For example, building a Docker image. Additionally, they should be composable, so that for example it's possible to do something before or after each stage. That way, a pipeline can use a common stage even if it needs to do something additional in that stage.

First, let's define a template function:

```groovy
/**
 * Defines a stage template to reduce the boilerplate in actual stages.
 */
def template(stageName, containerName, body, before = null, after = null) {
  stage(stageName) {
    container(containerName) {
      if (before != null) {
        before()
      }

      body()

      if (after != null) {
        after()
      }
    }
  }
}
```

Essentially it's a stage wrapper that also supports before and after logic. `body`, `before` and `after` are expected to be functions/closures.

Now we can define concrete stages using this template function:

```groovy
/**
 * Builds docker image needed to run the app.
 */
def buildImage(context, before = null, after = null) {
  template('Build images', 'docker', {
    sh "docker build -t ${context.name}:${context.env.CHANGE_ID} ."
  }, before, after)
}

/**
 * Pushes built image to registry.
 */
def pushImage(context, before = null, after = null) {
  template('Push images', 'docker', {
    utils.tagAndPushImage(context.name, env.CHANGE_ID)
  }, before, after)
}

// add other stages as needed ...
```

## pipelines.groovy

The `pipelines.groovy` will contain pipelines, which will be assembled from a number of stages (preferrably defined in `stages.groovy`). It is common to have slightly different pipelines in case it's a `master` branch or `PR` branch. For example, you may deploy Postgres without permanent storage for `PR` deployment and with permanent storage for staging (`develop` branch). Let's start with a template function and basic pipeline/wrapper for kubernetes deployments:

```groovy
import groovy.transform.Field

// Unfortunately, enums can't be exported/used in other modules,
// so we have to use the map.
@Field Type = [
  PullRequest: 'PullRequest',
  DevQA: 'DevQA',
  Production: 'Production'
]

def template(name, env, pipelineStages) {
  def label = UUID.randomUUID().toString()

  if (env.BRANCH_NAME == 'master') {
    // production
    kubernetesPipeline(Type.Production, label, name, env, pipelineStages)
  } else if (env.BRANCH_NAME == 'develop') {
    // staging
    kubernetesPipeline(Type.DevQA, label, name, env, pipelineStages)
  } else if (env.CHANGE_ID) {
    // pull request
    //
    // You may have some additional logic here to decide
    // when to deploy a PR and when not - for example,
    // here we'll only deploy if branch name ends with `-deploy`
    if (env.CHANGE_BRANCH ==~ /^.*?-deploy$/) {
      kubernetesPipeline(Type.PullRequest, label, name, env, pipelineStages)
    } else {
      emptyPipeline('This PR branch is not deployable. Skipping build.')
    }
  } else {
    emptyPipeline('Looks like this is neither PR branch nor develop/master branch. Skipping build.')
  }
}

def kubernetesPipeline(type, nodeId, name, env, stages) {
  podTemplate(label: nodeId, yaml: libraryResource('com/yourcompany/jenkins/build-pod.yaml')) {
    node(nodeId) {
      checkout scm

      def context = [
        type: type,
        env: env,
        name: name,
        project: utils.getProjectName(name, env),
        dockerfileStages: dockerfileStages,
        devClusterDomain: clusterDomain
      ]

      echo "Starting ${type} pipeline"

      stages(context)
    }
  }  
}

def emptyPipeline(message) {
  stage('Skipped build') {
    echo message
  }
}
```

Ok, so we've defined some more building blocks. Let's see how this can be used for a more concrete pipeline:

```groovy
/**
 * Defines a pipeline for Elixir/Phoenix projects.
 *
 * @param name the project name
 * @param env map of environment params
 * @param apps names of the apps in Phoenix umbrella project
 * @param customEnvVars a map of custom env variables
 */
def elixir(name, env, apps, customEnvVars) {
  def pipelineStages = { context ->
    stages.buildImage(context)
    stages.createProject(context)

    runTests(context, apps, customEnvVars)

    stages.pushImage(context)
    
    if (context.type == pipelines.Type.PullRequest) {
      stages.deployAppWithEphemeralDb(context)
      stages.verifyDeployment(context)
      migrateDb(context)
      seedDb(context)
    } else {
      stages.deployAppWithPersistentDb(context)
      stages.verifyDeployment(context)
      migrateDb(context)
    }
  }

  pipelines.template(name, env, pipelineStages)
}

def runTests(context, apps, customVars) {
  stage('Run tests') {
    container('openshift') {
      utils.deployEphemeralDb(context.project, 'postgresql-test', 'test')
      sh "oc rollout status dc postgresql-test -w -n ${context.project}"
      sh "oc adm pod-network join-projects --to=${context.project} openshift-build"
    }

    try {
      container('elixir') {
        sh "mix local.hex --force && mix local.rebar --force && mix deps.get"
        sh "mix credo"

        params = "MIX_ENV=test " +
                "DATABASE_HOSTNAME=postgresql-test.${context.project} " +
                "POSTGRESQL_USER=postgres " +
                "POSTGRESQL_PASSWORD=postgres " +
                "POSTGRESQL_DATABASE=test " +
                "HOSTNAME=localhost "

        if (customVars && customVars.size() > 0) {
          customVars.each {
            params += "${it} "
          }
        }

        sh "${params} mix test"

        apps.each {
          junit "_build/test/lib/${it}/test-junit-report.xml"
        }
      }
    } finally {
      container('openshift') {
        utils.destroyEphemeralDb(context.project, 'postgresql-test')
      }
    }
  }
}

```

Alright, so that's quite a few things going on. We've defined an elixir pipeline as a closure that runs some stages and fed that closure into template pipeline. As an example, I've specified `runTests` stage as a completely custom Groovy code block and used it as part of elixir pipeline.

In similar manner we could define a pipeline for React/Typescript project and so on.

## Projects using the shared library
The `Jenkinsfile` in each of the projects can now be much shorter. Assuming we have our shared library in Github with the name `demo-library`, then the Jenkinsfile could be this short:

```groovy
library(
  identifier: "demo-library@master",
  retriever: modernSCM(
    [
      $class: 'GitSCMSource',
      remote: 'https://github.com/your-company/demo-library.git',
      credentialsId: 'github'
    ]
  )
)

pipelines.elixir(
  'project_name',
  env, // 'env' here is the Jenkins' environment
  ['my_app', 'my_app_web'],
  [
    CUSTOM_VAR1: '...',
    CUSTOM_VAR2: '...',
  ]
)
```

Ok, so this Jenkinsfile does two things:

1. Loads a shared library from a github repo (the `credentialsId` you can find in your Jenkins settings by going to `https://jenkins.your-company.com/credentials/` assuming you've setup Github integration int  Jenkins). Once the shared library is loaded, then its exported functions can be used without further imports or similar mechanisms.
2. Invokes `pipelines.elixir(...)` with the project's name and Jenkins environment.

> Note that you can also import a shared library [globally](https://jenkins.io/doc/book/pipeline/shared-libraries/#using-libraries) in Jenkins configuration, then just alias it in the Jenkinsfile like so: `@Library('demo-library') _`.

Finally, we should remove yaml files or other resources, because they are now part of the shared library.

## Should I use Jenkins Shared Libraries?

I think that if you have multiple projects built, tested and deployed by Jenkins, then shared libraries provide a very flexible way to deal with repetition and boilerplate. But we have to look at cons, too, as is the case with any abstraction. To me, the single biggest con is that looking at a project's Jenkinsfile, one has no idea what is actually happening without looking at the code     in the shared library project. But to me the ability to fix/change/enhance continuous integration and delivery at one place for all projects outweighs the blackbox-iness of this approach.
