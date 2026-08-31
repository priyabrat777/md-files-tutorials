# Jenkins Multibranch Pipeline and Declarative `Jenkinsfile` Tutorial

This tutorial explains how to create a Jenkins Multibranch Pipeline, how branch and pull-request discovery works, and how to write a production-ready Declarative Pipeline. It also serves as a syntax reference for the core Declarative Pipeline grammar.

> **Scope:** Declarative Pipeline directives are finite and are covered below. Pipeline **steps** are extensible: Jenkins and every installed plugin can add steps, so no static document can list every possible step. Use **Dashboard → your job → Pipeline Syntax** (Snippet Generator) or `${JENKINS_URL}/pipeline-syntax/` for the exact step catalog on your controller.

## Table of contents

1. [What a Multibranch Pipeline does](#1-what-a-multibranch-pipeline-does)
2. [Prerequisites](#2-prerequisites)
3. [Repository layout](#3-repository-layout)
4. [Create the Multibranch Pipeline](#4-create-the-multibranch-pipeline)
5. [A complete practical Jenkinsfile](#5-a-complete-practical-jenkinsfile)
6. [Declarative Pipeline grammar](#6-declarative-pipeline-grammar)
7. [Agent syntax](#7-agent-syntax)
8. [Environment variables and credentials](#8-environment-variables-and-credentials)
9. [Options](#9-options)
10. [Parameters](#10-parameters)
11. [Triggers and Jenkins cron syntax](#11-triggers-and-jenkins-cron-syntax)
12. [Stages, steps, and script blocks](#12-stages-steps-and-script-blocks)
13. [The when directive](#13-the-when-directive)
14. [Input and approvals](#14-input-and-approvals)
15. [Post actions](#15-post-actions)
16. [Sequential, parallel, and matrix stages](#16-sequential-parallel-and-matrix-stages)
17. [Tools](#17-tools)
18. [Common Pipeline steps](#18-common-pipeline-steps)
19. [Multibranch environment variables](#19-multibranch-environment-variables)
20. [Pull requests, tags, and deployment policy](#20-pull-requests-tags-and-deployment-policy)
21. [Shared Libraries](#21-shared-libraries)
22. [Testing, validation, and troubleshooting](#22-testing-validation-and-troubleshooting)
23. [Security and production practices](#23-security-and-production-practices)
24. [Quick syntax cheat sheet](#24-quick-syntax-cheat-sheet)
25. [Official references](#25-official-references)

---

## 1. What a Multibranch Pipeline does

A normal Pipeline job points at one Pipeline definition. A **Multibranch Pipeline** scans a source-code repository and creates a child job for every eligible branch, pull/change request, or tag that contains a `Jenkinsfile`.

For example:

```text
Repository
├── main                 → Jenkins job: project/main
├── feature/login        → Jenkins job: project/feature%2Flogin
├── PR-42                → Jenkins job: project/PR-42
└── v2.1.0               → Jenkins job: project/v2.1.0
```

The important properties are:

- Pipeline code is versioned beside application code.
- Different branches may safely evolve their own `Jenkinsfile`.
- Branch-source plugins can discover branches, pull requests, and tags.
- Webhooks can trigger indexing/builds quickly; periodic scans provide a fallback.
- Deleted branches can be removed according to the configured orphaned-item strategy.
- An **Organization Folder** applies the same idea to many repositories.

## 2. Prerequisites

You need:

- A supported Jenkins LTS installation and at least one build agent.
- **Pipeline** and **Pipeline: Declarative** plugins.
- **Git** plus the branch-source plugin for the SCM provider, such as GitHub Branch Source, GitLab Branch Source, Bitbucket Branch Source, or Gitea.
- Repository read credentials in Jenkins when the repository is private.
- A `Jenkinsfile` committed to the configured script path, normally the repository root.
- Any runtime used by the example (`docker`, Maven, JDK, Node.js, and so on) installed on the relevant agent or provided through a container.

Common optional plugins include JUnit, HTML Publisher, Pipeline Utility Steps, Credentials Binding, Docker Pipeline, Kubernetes, and a notification integration. A step supplied by one of these plugins fails with “No such DSL method” if that plugin is absent.

## 3. Repository layout

A typical repository looks like this:

```text
my-service/
├── Jenkinsfile
├── pom.xml
├── src/
│   ├── main/
│   └── test/
└── scripts/
    └── deploy.sh
```

Jenkins uses the exact case-sensitive script path configured in the branch source. `Jenkinsfile`, `ci/Jenkinsfile`, and `jenkinsfile` are different paths on a case-sensitive filesystem.

Start with the smallest valid Declarative Pipeline:

```groovy
pipeline {
    agent any

    stages {
        stage('Hello') {
            steps {
                echo 'Hello from Jenkins'
            }
        }
    }
}
```

Declarative syntax rules:

- The top-level block must be `pipeline {}`.
- A Pipeline has one top-level `agent` directive (`agent any`, `agent none`, or an agent block).
- A Pipeline must have a `stages {}` section containing at least one `stage`.
- Each ordinary stage has one of `steps`, nested `stages`, `parallel`, or `matrix`.
- Directives have defined locations. For example, top-level `parameters` cannot be placed inside `steps`.
- Steps are Groovy method calls, but Declarative structure is not arbitrary Groovy. Put imperative Groovy in `script {}` only when necessary.

## 4. Create the Multibranch Pipeline

### 4.1 Add Jenkins credentials

For a private repository:

1. Open **Manage Jenkins → Credentials**.
2. Select the appropriate domain/store.
3. Add an SSH private key, username/token, or provider-specific credential.
4. Give it a stable ID, such as `github-app-ci` or `scm-read-key`.

Do not put tokens or private keys in the repository URL or `Jenkinsfile`.

### 4.2 Create the job

1. Select **New Item**.
2. Enter a job name without spaces when practical.
3. Select **Multibranch Pipeline**, then **OK**.
4. Under **Branch Sources**, add the SCM provider.
5. Enter the repository owner/project and repository URL.
6. Select the repository credential.
7. Configure **Behaviors**: branch discovery, pull-request discovery, tag discovery, include/exclude filters, and clone options as required.
8. Set **Build Configuration → Mode** to `by Jenkinsfile` and the script path to `Jenkinsfile` (or the repository-relative custom path).
9. Configure **Scan Multibranch Pipeline Triggers** as a fallback to webhooks.
10. Configure **Orphaned Item Strategy** so old branch jobs and build history do not grow forever.
11. Save and select **Scan Multibranch Pipeline Now**.

After indexing, Jenkins creates jobs only for heads that match the discovery rules and contain the configured script path.

### 4.3 Configure webhooks

Prefer the SCM provider’s Jenkins integration or webhook endpoint. The precise endpoint and events depend on the branch-source plugin. Generally enable push, pull/merge-request, and tag events that the job must build.

Use periodic scanning as a safety net, not as a substitute for a correctly configured webhook. In multibranch jobs, branch indexing discovers new/deleted heads; a Pipeline-level `pollSCM` checks for changes to an already known head.

### 4.4 First-run parameter behavior

Declarative `parameters {}` are job properties. On a newly discovered branch, Jenkins may need one run to create/update those properties before **Build with Parameters** appears. Design defaults so the first automatic run is safe.

## 5. A complete practical Jenkinsfile

The following example illustrates a realistic Java service. It intentionally demonstrates many directives in one place; remove parts your project does not need.

```groovy
pipeline {
    // Allocate agents per stage so Linux, Docker, or deployment nodes can differ.
    agent none

    options {
        buildDiscarder(logRotator(numToKeepStr: '30', artifactNumToKeepStr: '10'))
        disableConcurrentBuilds(abortPrevious: true)
        timeout(time: 60, unit: 'MINUTES')
        timestamps()
        ansiColor('xterm')                 // Plugin-provided option; remove if unavailable.
        skipDefaultCheckout(true)
        preserveStashes(buildCount: 5)
        parallelsAlwaysFailFast()
    }

    parameters {
        choice(
            name: 'DEPLOY_ENV',
            choices: ['none', 'dev', 'staging', 'production'],
            description: 'Optional deployment target'
        )
        booleanParam(
            name: 'RUN_INTEGRATION_TESTS',
            defaultValue: true,
            description: 'Run integration tests'
        )
        string(
            name: 'MAVEN_ARGS',
            defaultValue: '-B -ntp',
            description: 'Non-secret Maven arguments',
            trim: true
        )
    }

    environment {
        APP_NAME = 'orders-service'
        REGISTRY = 'registry.example.com'
    }

    stages {
        stage('Checkout') {
            agent { label 'linux && git' }
            options {
                timeout(time: 10, unit: 'MINUTES')
                retry(2)
            }
            steps {
                checkout scm
                sh 'git --no-pager log -1 --oneline'
                stash name: 'source', includes: '**/*', excludes: '.git/**'
            }
        }

        stage('Build and test') {
            agent {
                docker {
                    image 'maven:3.9.9-eclipse-temurin-21'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                unstash 'source'
                sh 'mvn $MAVEN_ARGS clean verify'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
                success {
                    archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
                    stash name: 'application', includes: '**/target/*.jar'
                }
            }
        }

        stage('Integration tests') {
            when {
                beforeAgent true
                expression { params.RUN_INTEGRATION_TESTS }
            }
            agent { label 'linux && docker' }
            steps {
                unstash 'source'
                sh './scripts/integration-test.sh'
            }
        }

        stage('Quality checks') {
            parallel {
                stage('Static analysis') {
                    agent { label 'linux' }
                    steps {
                        unstash 'source'
                        sh './mvnw -B -ntp verify -DskipTests'
                    }
                }
                stage('Dependency audit') {
                    agent { label 'linux' }
                    steps {
                        unstash 'source'
                        sh './scripts/dependency-audit.sh'
                    }
                }
            }
        }

        stage('Build image') {
            when {
                anyOf {
                    branch 'main'
                    buildingTag()
                }
            }
            agent { label 'linux && docker' }
            environment {
                IMAGE_TAG = "${env.BUILD_NUMBER}"
                // Creates REGISTRY_CREDS, REGISTRY_CREDS_USR, and REGISTRY_CREDS_PSW.
                // Scoping it here keeps this credential out of earlier stages.
                REGISTRY_CREDS = credentials('container-registry-credentials')
            }
            steps {
                unstash 'source'
                unstash 'application'
                sh '''
                    set -eu
                    printf '%s' "$REGISTRY_CREDS_PSW" | docker login "$REGISTRY" \
                      --username "$REGISTRY_CREDS_USR" --password-stdin
                    docker build -t "$REGISTRY/$APP_NAME:$IMAGE_TAG" .
                    docker push "$REGISTRY/$APP_NAME:$IMAGE_TAG"
                '''
            }
        }

        stage('Deploy') {
            when {
                beforeInput true
                allOf {
                    branch 'main'
                    expression { params.DEPLOY_ENV != 'none' }
                }
            }
            input {
                message "Deploy build ${env.BUILD_NUMBER} to ${params.DEPLOY_ENV}?"
                ok 'Deploy'
                submitter 'release-engineers'
                submitterParameter 'APPROVED_BY'
            }
            agent { label 'linux && deploy' }
            steps {
                withCredentials([
                    string(credentialsId: 'deployment-api-token', variable: 'DEPLOY_TOKEN')
                ]) {
                    sh '''
                        set +x
                        ./scripts/deploy.sh "$DEPLOY_ENV" "$BUILD_NUMBER"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Build succeeded: ${env.BUILD_URL}"
        }
        unstable {
            echo 'Build is unstable; inspect test or quality reports.'
        }
        failure {
            echo "Build failed: ${env.BUILD_URL}"
        }
    }
}
```

Important adjustments before using it:

- Create credentials with IDs `container-registry-credentials` and `deployment-api-token`.
- Ensure all labels exist, or replace them with your agent labels.
- Ensure the selected agent can run Docker, and review the security implications of mounting the Docker socket or cache directories.
- Replace the build, audit, and deployment commands with repository scripts.
- Remove `ansiColor` if the AnsiColor plugin is not installed.
- Avoid using untrusted pull-request code on agents that can reach production credentials or privileged Docker services.

## 6. Declarative Pipeline grammar

This skeleton shows where core sections and directives belong:

```groovy
pipeline {
    agent any                         // Required at Pipeline level.

    environment { /* variables */ }  // Optional, once at this level.
    options { /* options */ }         // Optional, once at this level.
    parameters { /* parameters */ }   // Optional, once at this level.
    triggers { /* triggers */ }       // Optional, once at this level.
    tools { /* tools */ }             // Optional, once at this level.

    stages {                          // Required.
        stage('Name') {
            agent any                 // Optional stage override.
            environment { /* ... */ }
            options { /* ... */ }
            tools { /* ... */ }
            input { /* ... */ }
            when { /* ... */ }

            steps {                   // Use one of steps/stages/parallel/matrix.
                echo 'work'
            }

            post { /* stage result actions */ }
        }
    }

    post { /* Pipeline result actions */ }
}
```

At a stage’s execution-content position, these are mutually exclusive:

```groovy
stage('Example') {
    steps { /* ordinary steps */ }
    // OR stages { /* sequential child stages */ }
    // OR parallel { /* parallel child stages */ }
    // OR matrix { /* generated parallel cells */ }
}
```

A stage nested under `parallel` or `matrix` cannot itself contain another `parallel` or `matrix` block. Use a surrounding sequential stage to reshape complex flows.

## 7. Agent syntax

### 7.1 Any agent

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps { echo "Running on ${env.NODE_NAME}" }
        }
    }
}
```

### 7.2 No global agent

This is efficient when stages need different environments:

```groovy
pipeline {
    agent none
    stages {
        stage('Linux') {
            agent { label 'linux' }
            steps { sh 'uname -a' }
        }
        stage('Windows') {
            agent { label 'windows' }
            steps { bat 'ver' }
        }
    }
}
```

### 7.3 Label expression

```groovy
agent { label 'linux && docker' }
```

Label expressions support operators such as `&&`, `||`, `!`, `->`, and `<->`, with parentheses for grouping. Quote the entire expression.

### 7.4 Node and custom workspace

```groovy
agent {
    node {
        label 'linux'
        customWorkspace '/opt/jenkins-workspaces/orders-service'
    }
}
```

Use custom workspaces cautiously: concurrent jobs can collide unless the path is unique and managed deliberately.

### 7.5 Docker image

Requires Docker Pipeline and a compatible agent:

```groovy
agent {
    docker {
        image 'node:22-alpine'
        label 'linux && docker'
        args '-v /tmp/npm-cache:/root/.npm'
        alwaysPull true
        reuseNode true
        customWorkspace 'workspace/frontend'
        registryUrl 'https://registry.example.com'
        registryCredentialsId 'registry-pull-credential'
    }
}
```

Not every plugin/version accepts every optional field at every location. The controller’s Declarative Directive Generator is authoritative for the installed version.

### 7.6 Dockerfile

Build an image from the repository’s `Dockerfile`:

```groovy
agent {
    dockerfile {
        filename 'ci.Dockerfile'
        dir 'ci'
        label 'linux && docker'
        additionalBuildArgs '--build-arg VERSION=21'
        args '--network host'
        reuseNode true
        registryUrl 'https://registry.example.com'
        registryCredentialsId 'registry-pull-credential'
    }
}
```

`docker` and `dockerfile` are plugin-backed agent implementations. Kubernetes and cloud plugins may add other agent forms; consult that plugin’s documentation.

### 7.7 Kubernetes pod

The Kubernetes plugin adds a `kubernetes` agent. A compact example is:

```groovy
agent {
    kubernetes {
        defaultContainer 'builder'
        yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: builder
      image: eclipse-temurin:21-jdk
      command: ['sleep']
      args: ['99d']
'''
    }
}
```

Pod-template fields and additional options are provided by the Kubernetes plugin and can vary by version. Use its documentation and the controller’s Directive Generator. Keep Kubernetes `Secret` values out of inline YAML; reference cluster-managed secrets or bind Jenkins credentials at runtime.

### 7.8 Top-level versus stage-level timeout

- With a top-level agent, Jenkins allocates the agent and then starts a top-level `timeout`.
- With a stage-level agent, stage `options` are evaluated before allocating the stage agent, so allocation time counts toward a stage timeout.

This distinction matters when agents are created on demand.

## 8. Environment variables and credentials

### 8.1 Static and computed variables

```groovy
pipeline {
    agent any
    environment {
        SERVICE = 'catalog'
        RELEASE_ID = "${env.BUILD_NUMBER}"
        CC = 'clang'
    }
    stages {
        stage('Example') {
            environment {
                MODE = 'test' // Scoped to this stage.
            }
            steps {
                sh 'echo "$SERVICE $RELEASE_ID $MODE"'
                echo "Groovy reads it as ${env.SERVICE}"
            }
        }
    }
}
```

Use `env.NAME` in Groovy and the shell’s native syntax inside shell steps: `$NAME` in `sh`, `%NAME%` in `bat`, and `$Env:NAME` in PowerShell.

Dynamic values returned by steps can be assigned when an agent is available:

```groovy
environment {
    SHORT_SHA = "${sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()}"
}
```

This form executes a step and therefore needs an agent/context. Often it is clearer to set the value in a stage:

```groovy
steps {
    script {
        env.SHORT_SHA = sh(
            script: 'git rev-parse --short HEAD',
            returnStdout: true
        ).trim()
    }
}
```

### 8.2 The credentials helper

For supported credential types:

```groovy
environment {
    API_TOKEN = credentials('api-secret-text')
    SCM_LOGIN = credentials('scm-username-password')
    SIGNING_KEY_FILE = credentials('signing-secret-file')
    SSH_KEY_FILE = credentials('ssh-private-key')
}
```

For username/password credentials, `SCM_LOGIN` contains `username:password`, while Jenkins also creates `SCM_LOGIN_USR` and `SCM_LOGIN_PSW`. For a secret file, the variable contains a temporary file path. For an SSH private-key credential, the main variable contains the temporary key-file path and Jenkins also creates `_USR` and `_PSW` variables for the username and passphrase.

Scope secrets as narrowly as practical. `withCredentials` supports more types and tighter scoping:

```groovy
steps {
    withCredentials([
        string(credentialsId: 'api-token', variable: 'TOKEN'),
        usernamePassword(
            credentialsId: 'service-login',
            usernameVariable: 'SERVICE_USER',
            passwordVariable: 'SERVICE_PASSWORD'
        ),
        file(credentialsId: 'kubeconfig-file', variable: 'KUBECONFIG')
    ]) {
        sh '''
            set +x
            ./publish --user "$SERVICE_USER" --token "$TOKEN"
        '''
    }
}
```

Prefer single-quoted Groovy strings for commands containing secrets. This lets the shell expand environment variables and prevents Groovy from interpolating a secret into the process argument before masking is applied. Masking reduces accidental disclosure; it is not a security boundary against malicious Pipeline code.

### 8.3 Temporary environment overrides

```groovy
steps {
    withEnv(['MODE=integration', 'LOG_LEVEL=debug']) {
        sh './run-tests.sh'
    }
}
```

## 9. Options

The `options` directive configures Pipeline-wide or stage-specific behavior. Availability depends on Jenkins and plugin versions.

### 9.1 Common top-level options

```groovy
options {
    buildDiscarder(logRotator(
        daysToKeepStr: '30',
        numToKeepStr: '50',
        artifactDaysToKeepStr: '14',
        artifactNumToKeepStr: '10'
    ))
    checkoutToSubdirectory('source')
    disableConcurrentBuilds(abortPrevious: true)
    disableRestartFromStage()
    disableResume()
    newContainerPerStage()
    overrideIndexTriggers(true)
    parallelsAlwaysFailFast()
    preserveStashes(buildCount: 5)
    quietPeriod(10)
    retry(2)
    skipDefaultCheckout(true)
    skipStagesAfterUnstable()
    timeout(time: 1, unit: 'HOURS')
    timestamps()
}
```

Meaning of the core options:

| Option | Purpose |
|---|---|
| `buildDiscarder(...)` | Retains a bounded number/age of builds and artifacts. |
| `checkoutToSubdirectory('dir')` | Places the automatic checkout in a subdirectory. |
| `disableConcurrentBuilds()` | Queues a run while an earlier run is active. |
| `disableConcurrentBuilds(abortPrevious: true)` | Aborts the older run in favor of the new run. |
| `disableRestartFromStage()` | Removes the Declarative **Restart from Stage** action. |
| `disableResume()` | Prevents resuming the Pipeline after a controller restart. |
| `newContainerPerStage()` | With a top-level Docker agent, starts a fresh container for each stage. |
| `overrideIndexTriggers(true\|false)` | Overrides whether branch indexing triggers builds for this job. |
| `parallelsAlwaysFailFast()` | Applies fail-fast behavior to subsequent parallel stages. |
| `preserveStashes(buildCount: 5)` | Retains stashes so a Declarative run can restart from a later stage. |
| `quietPeriod(seconds)` | Delays a scheduled build. |
| `retry(count)` | Retries the entire Pipeline on failure. |
| `skipDefaultCheckout(true)` | Disables automatic `checkout scm`; check out explicitly. |
| `skipStagesAfterUnstable()` | Skips later stages after the build becomes unstable. |
| `timeout(...)` | Aborts work after a duration. |
| `timestamps()` | Prepends timestamps to log lines. |

Common timeout units include `NANOSECONDS`, `MICROSECONDS`, `MILLISECONDS`, `SECONDS`, `MINUTES`, `HOURS`, and `DAYS`.

### 9.2 Stage-level options

Stage `options` are invoked before the stage’s `agent` and `when` evaluation:

```groovy
stage('Flaky external test') {
    options {
        timeout(time: 15, unit: 'MINUTES')
        retry(2)
        timestamps()
        skipDefaultCheckout(true)
    }
    agent { label 'ephemeral' }
    steps {
        sh './external-test.sh'
    }
}
```

Some plugins expose additional job properties or options, such as `ansiColor`, `lock`, or rate limiting. Generate their syntax on the target controller.

Some installations also expose `durabilityHint('MAX_SURVIVABILITY')` (or another configured durability mode) as a Declarative option. Because availability and accepted values depend on the installed Pipeline plugins and controller policy, generate this option on the target controller before using it.

## 10. Parameters

Core Declarative parameter types are:

```groovy
parameters {
    string(
        name: 'VERSION',
        defaultValue: '1.0.0',
        description: 'Release version',
        trim: true
    )
    text(
        name: 'RELEASE_NOTES',
        defaultValue: 'Summary\n',
        description: 'Multiline release notes'
    )
    booleanParam(
        name: 'DRY_RUN',
        defaultValue: true,
        description: 'Do not change external systems'
    )
    choice(
        name: 'TARGET',
        choices: ['dev', 'staging', 'production'],
        description: 'Deployment target'
    )
    password(
        name: 'LEGACY_PASSWORD',
        defaultValue: '',
        description: 'Avoid when a Jenkins credential can be used'
    )
}
```

Read values through `params`:

```groovy
steps {
    echo "Version: ${params.VERSION}"
    script {
        if (params.DRY_RUN) {
            echo 'Dry-run mode is active'
        }
    }
}
```

Parameters are also exported as environment variables at build start, but `params.NAME` preserves types—for example, a Boolean stays a Boolean—and is clearer in Groovy.

The first `choice` entry is the default. Treat `password` parameters as legacy convenience, not durable secret storage; use Jenkins Credentials for managed secrets. Plugins may add parameter types such as credentials, file, run, active-choice, or Git parameters.

## 11. Triggers and Jenkins cron syntax

Declarative supports these core trigger forms:

```groovy
triggers {
    cron('H 2 * * 1-5')
    pollSCM('H/15 * * * *')
    upstream(
        upstreamProjects: 'library-main,security-scan',
        threshold: hudson.model.Result.SUCCESS
    )
}
```

Use only the triggers the job actually requires:

- `cron` schedules a build even when source did not change.
- `pollSCM` polls SCM and builds only when Jenkins detects a change.
- `upstream` starts after one of the named jobs reaches the configured result threshold.
- Provider plugins/webhooks commonly schedule multibranch builds without a `triggers` block.

Jenkins cron has five fields:

```text
MINUTE HOUR DAY-OF-MONTH MONTH DAY-OF-WEEK
```

Examples:

```groovy
triggers {
    cron('H/15 * * * *')        // About every 15 minutes, hash-balanced.
    // cron('H H(1-3) * * 1-5') // Once in the 01:00–03:59 window on weekdays.
    // cron('@daily')            // Once daily at a hash-selected time.
}
```

`H` is a stable hash chosen from the job name. It spreads load across Jenkins and is usually better than scheduling every job at minute `0`. Supported aliases include `@yearly`, `@annually`, `@monthly`, `@weekly`, `@daily`, `@midnight`, and `@hourly`.

## 12. Stages, steps, and script blocks

### 12.1 Ordinary stages and steps

```groovy
stages {
    stage('Build') {
        steps {
            echo 'Building'
            sh './gradlew assemble'
        }
    }
}
```

### 12.2 Declarative `script`

Use `script {}` for imperative Groovy that cannot be expressed cleanly with directives or Pipeline steps:

```groovy
steps {
    script {
        def modules = ['api', 'worker', 'web']
        for (String module : modules) {
            sh "./build-module.sh '${module}'"
        }
        currentBuild.description = "${env.BRANCH_NAME} #${env.BUILD_NUMBER}"
    }
}
```

Keep large logic in tested repository scripts or a Shared Library. Groovy code in a sandbox may require administrator approval, and Pipeline serialization can reject non-serializable objects retained across suspension points.

### 12.3 `steps` versus `script`

This is valid because `sh`, `echo`, and `junit` are Pipeline steps:

```groovy
steps {
    sh './test.sh'
    junit '**/test-results/*.xml'
}
```

This requires `script` because it contains Groovy control flow:

```groovy
steps {
    script {
        if (env.BRANCH_NAME == 'main') {
            echo 'Main branch'
        }
    }
}
```

For gating whole stages, prefer `when` over a Groovy `if`.

## 13. The `when` directive

`when` controls whether a stage runs.

### 13.1 Built-in conditions

```groovy
stage('Conditional examples') {
    when {
        allOf {
            branch pattern: 'release-*', comparator: 'GLOB'
            not { buildingTag() }
            environment name: 'DEPLOY_ALLOWED', value: 'true'
            equals expected: 'production', actual: params.TARGET
            expression { currentBuild.number > 1 }
        }
    }
    steps {
        echo 'All conditions passed'
    }
}
```

Core conditions:

| Condition | Example and purpose |
|---|---|
| `branch` | `branch 'main'` or `branch pattern: 'release-*', comparator: 'GLOB'`; multibranch only. |
| `buildingTag` | `buildingTag()`; true when building a tag. |
| `tag` | `tag 'release-*'`; tag name match. An empty pattern checks whether `TAG_NAME` exists. |
| `changelog` | `changelog '.*JIRA-[0-9]+.*'`; regular-expression match against changelog text. |
| `changeset` | `changeset '**/*.js'`; match changed file paths. |
| `changeRequest` | `changeRequest target: 'main'`; match pull/change-request metadata. |
| `environment` | `environment name: 'MODE', value: 'production'`. |
| `equals` | `equals expected: 2, actual: currentBuild.number`. |
| `expression` | `expression { params.DEPLOY }`; custom Boolean logic. |
| `triggeredBy` | `triggeredBy 'TimerTrigger'` or a cause with fields. |
| `not` | `not { branch 'main' }`. |
| `allOf` | True only when all nested conditions are true. |
| `anyOf` | True when at least one nested condition is true; short-circuits. |

`branch` and several name/path matchers support comparators:

- `GLOB` — Ant-style glob matching; commonly the default.
- `REGEXP` — regular expression matching.
- `EQUALS` — literal equality.

Examples:

```groovy
when {
    branch pattern: 'release-\\d+', comparator: 'REGEXP'
}
```

```groovy
when {
    changeRequest target: 'main', fork: 'trusted-org', comparator: 'EQUALS'
}
```

Common `changeRequest` attributes include `id`, `target`, `branch`, `fork`, `url`, `title`, `author`, `authorDisplayName`, and `authorEmail`. Available metadata depends on the SCM branch-source plugin.

```groovy
when {
    changeset pattern: '**/*.TEST.PY', caseSensitive: true, comparator: 'GLOB'
}
```

```groovy
when {
    triggeredBy cause: 'UserIdCause', detail: 'alice'
}
```

### 13.2 Evaluation timing

By default, Jenkins applies stage `options`, enters `input`, and allocates the stage agent before evaluating `when`. Use one of these flags to evaluate sooner and avoid waiting unnecessarily:

```groovy
when {
    beforeAgent true
    branch 'main'
}
```

```groovy
when {
    beforeInput true
    branch 'main'
}
```

```groovy
when {
    beforeOptions true
    branch 'main'
}
```

Precedence is `beforeOptions` over `beforeInput`, and `beforeInput` over `beforeAgent`. Use only the relevant timing flag in a stage.

### 13.3 Expression truthiness trap

Returning the string `'false'` is truthy in Groovy. Return a Boolean or `null`:

```groovy
when {
    expression {
        return params.DEPLOY && env.BRANCH_NAME == 'main'
    }
}
```

## 14. Input and approvals

The stage-level Declarative `input` directive pauses before the stage agent is allocated and makes submitted parameter values available to that stage’s environment:

```groovy
stage('Production approval') {
    input {
        message 'Deploy to production?'
        id 'production-deploy'
        ok 'Deploy now'
        submitter 'alice,bob,release-managers'
        submitterParameter 'APPROVER'
        parameters {
            string(
                name: 'CHANGE_TICKET',
                defaultValue: '',
                description: 'Approved change-ticket number'
            )
            choice(
                name: 'REGION',
                choices: ['us-east-1', 'eu-west-1'],
                description: 'Deployment region'
            )
        }
    }
    agent { label 'deploy' }
    steps {
        echo "Approved by ${env.APPROVER} for ${env.CHANGE_TICKET}"
        sh './deploy.sh "$REGION"'
    }
}
```

The lower-level `input` **step** can be used inside `steps` when a pause must occur in the middle of execution:

```groovy
steps {
    script {
        def answer = input(
            message: 'Continue?',
            parameters: [booleanParam(name: 'CONFIRM', defaultValue: false)]
        )
        echo "Answer: ${answer}"
    }
}
```

Wrap approvals in an appropriate `timeout` so abandoned builds do not wait forever.

## 15. Post actions

`post` can appear at Pipeline or stage level. Conditions run in this order as applicable, with `cleanup` last:

```groovy
post {
    always {
        echo 'Runs regardless of result'
    }
    changed {
        echo 'Result differs from the previous completed run'
    }
    fixed {
        echo 'Current run succeeded after a failure/unstable run'
    }
    regression {
        echo 'Result is worse than the previous successful run'
    }
    aborted {
        echo 'Run was aborted'
    }
    failure {
        echo 'Run failed'
    }
    success {
        echo 'Run succeeded'
    }
    unstable {
        echo 'Run is unstable'
    }
    unsuccessful {
        echo 'Run did not succeed'
    }
    cleanup {
        deleteDir()
    }
}
```

Typical actions include publishing test reports, archiving artifacts, sending notifications, recording deployment metadata, and cleaning a workspace.

Use defensive publication when a previous step may not have produced output:

```groovy
post {
    always {
        junit allowEmptyResults: true, testResults: '**/test-results/*.xml'
        archiveArtifacts allowEmptyArchive: true, artifacts: '**/reports/**'
    }
}
```

## 16. Sequential, parallel, and matrix stages

### 16.1 Sequential stages

Nested sequential stages let a parent own one agent, `when`, and `post` block:

```groovy
stage('Deploy and verify') {
    when { branch 'main' }
    agent { label 'deploy' }
    stages {
        stage('Deploy') {
            steps { sh './deploy.sh staging' }
        }
        stage('Smoke test') {
            steps { sh './smoke-test.sh staging' }
        }
    }
    post {
        failure { echo 'Deployment sequence failed' }
    }
}
```

### 16.2 Parallel stages

```groovy
stage('Cross-platform tests') {
    failFast true
    parallel {
        stage('Linux') {
            agent { label 'linux' }
            steps { sh './test.sh' }
            post { always { junit '**/results/*.xml' } }
        }
        stage('Windows') {
            agent { label 'windows' }
            steps { bat 'test.cmd' }
            post { always { junit '**/results/*.xml' } }
        }
    }
}
```

Instead of `failFast true` on each parallel stage, use top-level `options { parallelsAlwaysFailFast() }`.

### 16.3 Matrix

`matrix` statically generates a cell for every axis combination, subtracts excluded combinations, and runs the remaining cells in parallel.

```groovy
stage('Test matrix') {
    matrix {
        axes {
            axis {
                name 'PLATFORM'
                values 'linux', 'windows', 'mac'
            }
            axis {
                name 'JDK'
                values '17', '21'
            }
            axis {
                name 'BROWSER'
                values 'chrome', 'firefox'
            }
        }

        excludes {
            exclude {
                axis {
                    name 'PLATFORM'
                    values 'mac'
                }
                axis {
                    name 'BROWSER'
                    values 'firefox'
                }
            }
            exclude {
                axis {
                    name 'PLATFORM'
                    notValues 'windows'
                }
                axis {
                    name 'JDK'
                    values '17'
                }
            }
        }

        agent { label "${PLATFORM}" }

        when {
            expression {
                params.PLATFORM_FILTER == 'all' ||
                    params.PLATFORM_FILTER == PLATFORM
            }
        }

        environment {
            TEST_PROFILE = "${PLATFORM}-${JDK}-${BROWSER}"
        }

        options {
            timeout(time: 30, unit: 'MINUTES')
        }

        stages {
            stage('Build') {
                steps {
                    echo "Building ${TEST_PROFILE}"
                }
            }
            stage('Test') {
                steps {
                    echo "Testing ${TEST_PROFILE}"
                }
            }
        }

        post {
            always {
                echo "Finished ${TEST_PROFILE}"
            }
        }
    }
}
```

Matrix syntax components:

- `axes` contains one or more `axis` blocks.
- Each `axis` has a `name` and one or more static `values`.
- Axis names become environment variables inside each cell.
- `excludes` contains `exclude` blocks; values in one `exclude` form an AND match.
- `notValues` excludes every value except those listed for that axis.
- `stages` defines work repeated for every cell.
- Cell-level `agent`, `environment`, `input`, `options`, `post`, `tools`, and `when` are optional.
- Axis combinations are calculated before execution; runtime code cannot dynamically add matrix cells.

## 17. Tools

`tools` automatically installs or selects globally configured tool installations and adds them to `PATH`:

```groovy
pipeline {
    agent any
    tools {
        jdk 'temurin-21'
        maven 'maven-3.9'
        gradle 'gradle-8'
    }
    stages {
        stage('Versions') {
            tools {
                // A stage may override a tool selection.
                jdk 'temurin-17'
            }
            steps {
                sh 'java -version'
                sh 'mvn -version'
                sh 'gradle --version'
            }
        }
    }
}
```

The names must match **Manage Jenkins → Tools**. Supported tool types depend on installed plugins. `tools` is ignored when the relevant stage uses `agent none`, because no node exists on which to install the tool.

The lower-level `tool` step returns an installation path:

```groovy
steps {
    script {
        def mvnHome = tool name: 'maven-3.9', type: 'maven'
        withEnv(["PATH+MAVEN=${mvnHome}/bin"]) {
            sh 'mvn -version'
        }
    }
}
```

## 18. Common Pipeline steps

The exact step list is controller-specific. These examples cover frequently used core and popular plugin steps.

### 18.1 Console and shell execution

```groovy
steps {
    echo 'A log message'
    sh 'make test'                       // POSIX agents
    bat 'gradlew.bat test'               // Windows cmd.exe
    powershell 'Get-ChildItem Env:'       // Windows PowerShell
    pwsh 'Get-ChildItem Env:'             // PowerShell Core
}
```

Capture output or status:

```groovy
steps {
    script {
        String sha = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
        int status = sh(script: './optional-check.sh', returnStatus: true)
        echo "SHA=${sha}, optional check status=${status}"
    }
}
```

### 18.2 SCM checkout

```groovy
steps {
    checkout scm // Best default in a Multibranch Pipeline.
}
```

`scm` represents the exact branch/change request selected by the branch-source plugin. A hard-coded `git` step can accidentally check out a different ref.

### 18.3 Files, directories, and workspaces

```groovy
steps {
    dir('frontend') {
        sh 'npm ci && npm test'
    }
    writeFile file: 'version.txt', text: "${env.BUILD_NUMBER}\n"
    script {
        String version = readFile('version.txt').trim()
        echo "Version ${version}"
    }
    fileExists('pom.xml')
    deleteDir()
}
```

`readJSON`, `writeJSON`, `readYaml`, and related steps normally come from Pipeline Utility Steps.

### 18.4 Stash versus artifacts

```groovy
stage('Compile') {
    steps {
        sh 'make package'
        stash name: 'package', includes: 'dist/**'
        archiveArtifacts artifacts: 'dist/**', fingerprint: true
    }
}
stage('Consume') {
    steps {
        unstash 'package'
    }
}
```

- `stash`/`unstash` transfers files between stages/agents in the same run. Stashes are normally discarded when the run ends.
- `archiveArtifacts` keeps downloadable build artifacts according to retention policy.
- For large artifacts, use an external artifact repository rather than sending them repeatedly through the Jenkins controller.

### 18.5 Tests and reports

```groovy
post {
    always {
        junit testResults: '**/build/test-results/**/*.xml',
              allowEmptyResults: false,
              skipPublishingChecks: false
    }
}
```

Report publishers such as HTML Publisher, JaCoCo, warnings/quality gates, and coverage tools are plugin steps.

### 18.6 Flow control and error handling

```groovy
steps {
    timeout(time: 5, unit: 'MINUTES') {
        retry(3) {
            sh './occasionally-flaky-command.sh'
        }
    }

    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
        sh './quality-gate.sh'
    }

    warnError('Optional scan failed') {
        sh './optional-scan.sh'
    }
}
```

Other useful steps include:

```groovy
steps {
    sleep time: 10, unit: 'SECONDS'
    waitUntil(initialRecurrencePeriod: 1000) {
        return fileExists('ready.flag')
    }
    error 'Stop with failure'
    unstable 'Mark this run unstable'
}
```

### 18.7 Build another job

```groovy
steps {
    build job: 'downstream-job',
          wait: true,
          propagate: true,
          parameters: [
              string(name: 'VERSION', value: env.BUILD_NUMBER),
              booleanParam(name: 'DRY_RUN', value: false)
          ]
}
```

### 18.8 Workspace allocation and locks

```groovy
steps {
    ws("${env.WORKSPACE}@isolated") {
        sh './build.sh'
    }
}
```

The Lockable Resources plugin provides `lock`:

```groovy
steps {
    lock(resource: 'shared-test-environment', inversePrecedence: true) {
        sh './deploy-and-test.sh'
    }
}
```

### 18.9 Docker steps

Docker Pipeline also supports Scripted-style steps inside Declarative `script`:

```groovy
steps {
    script {
        def image = docker.build("example/app:${env.BUILD_NUMBER}")
        docker.withRegistry('https://registry.example.com', 'registry-credential') {
            image.push()
        }
    }
}
```

Use the Snippet Generator for exact argument names and credential bindings on your Jenkins instance.

## 19. Multibranch environment variables

Common variables exposed by the Branch API include:

| Variable | Meaning |
|---|---|
| `BRANCH_NAME` | Name of the branch/head Jenkins is building. For a PR job this is often something like `PR-42`, not the source branch. |
| `BRANCH_IS_PRIMARY` | String flag when the SCM marks this head as a primary branch; provider support varies. |
| `CHANGE_ID` | Pull/change-request identifier. Present only for change requests. |
| `CHANGE_URL` | Change-request URL. |
| `CHANGE_TITLE` | Change-request title. |
| `CHANGE_AUTHOR` | Author login/identifier. |
| `CHANGE_AUTHOR_DISPLAY_NAME` | Author display name. |
| `CHANGE_AUTHOR_EMAIL` | Author email when available. |
| `CHANGE_TARGET` | Target/base branch, such as `main`. |
| `CHANGE_BRANCH` | Source/head branch when the provider supplies it. |
| `CHANGE_FORK` | Source fork identifier when applicable. |
| `TAG_NAME` | Tag being built. |
| `TAG_TIMESTAMP` | Tag timestamp in milliseconds since Unix epoch, when available. |
| `TAG_UNIXTIME` | Tag timestamp in seconds since Unix epoch. |
| `TAG_DATE` | Tag timestamp formatted as a date string. |

Common Jenkins variables, whether multibranch or not, include `BUILD_NUMBER`, `BUILD_ID`, `BUILD_TAG`, `BUILD_URL`, `JOB_NAME`, `JOB_BASE_NAME`, `NODE_NAME`, `WORKSPACE`, and `JENKINS_URL`.

Inspect what the current controller/provider supplies:

```groovy
stage('Context') {
    steps {
        echo "BRANCH_NAME=${env.BRANCH_NAME}"
        echo "CHANGE_ID=${env.CHANGE_ID ?: 'not a change request'}"
        echo "CHANGE_TARGET=${env.CHANGE_TARGET ?: 'n/a'}"
        echo "TAG_NAME=${env.TAG_NAME ?: 'not a tag'}"
    }
}
```

Avoid dumping the entire environment with `printenv` when credentials are bound; even with masking, logs may reveal sensitive metadata.

## 20. Pull requests, tags, and deployment policy

Use explicit stage policy:

```groovy
stage('PR validation') {
    when { changeRequest() }
    steps {
        sh './fast-pr-checks.sh'
    }
}

stage('Main deployment') {
    when {
        allOf {
            branch 'main'
            not { changeRequest() }
        }
    }
    steps {
        sh './deploy.sh staging'
    }
}

stage('Release') {
    when {
        tag pattern: 'v\\d+\\.\\d+\\.\\d+', comparator: 'REGEXP'
    }
    steps {
        sh './publish-release.sh "$TAG_NAME"'
    }
}
```

Security boundary for untrusted pull requests:

- Do not bind production credentials to builds whose `Jenkinsfile` can be changed by an untrusted contributor.
- Do not run untrusted code on controller nodes or privileged agents.
- Separate trusted deployment from untrusted validation, potentially into a different job/repository or a trusted Shared Library.
- Understand whether the branch-source plugin builds the PR head, a synthetic merge commit, or both.
- Review “build origin PRs” versus “build fork PRs” discovery strategies and trust settings.

## 21. Shared Libraries

Shared Libraries remove duplicated Pipeline code while keeping the `Jenkinsfile` concise.

Load a globally configured library:

```groovy
@Library('company-ci@v3') _

pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                companyBuild language: 'java', jdk: 21
            }
        }
    }
}
```

Typical library layout:

```text
shared-library/
├── src/org/example/Release.groovy
├── vars/companyBuild.groovy
├── vars/companyBuild.txt
└── resources/org/example/defaults.yaml
```

- `vars/name.groovy` exposes a global variable/step named `name`.
- `src/` contains class-like Groovy sources.
- `resources/` contains library resources.
- Pin deployment-critical libraries to a reviewed version rather than an untrusted branch.

A `vars/companyBuild.groovy` custom step might be:

```groovy
def call(Map config = [:]) {
    sh "./mvnw -B -ntp -Djava.version=${config.jdk ?: 21} verify"
}
```

Declarative Pipelines can also be defined in library code under documented restrictions; a simpler and more transparent pattern is usually to keep the top-level `pipeline {}` in the repository and share reusable steps.

## 22. Testing, validation, and troubleshooting

### 22.1 Generate and validate syntax

On the target Jenkins controller:

- **Pipeline Syntax → Snippet Generator** generates installed Pipeline step syntax.
- **Pipeline Syntax → Declarative Directive Generator** generates Declarative directives.
- The Declarative linter endpoint can validate a file before commit when enabled and authenticated:

```bash
curl --user 'USER:API_TOKEN' \
  --form 'jenkinsfile=<Jenkinsfile' \
  'https://jenkins.example.com/pipeline-model-converter/validate'
```

The linter validates Declarative structure but cannot prove shell scripts, external systems, credentials, agent labels, or plugin-dependent runtime behavior will work.

### 22.2 Recommended development loop

1. Create a feature branch.
2. Commit the `Jenkinsfile` with a small valid stage.
3. Run branch indexing if the webhook has not discovered it.
4. Add stages incrementally.
5. Use `when` to keep deployment disabled on the feature branch.
6. Validate reports, artifacts, timeouts, and cleanup on success and failure.
7. Test aborts and controller/agent restarts according to your durability requirements.

### 22.3 Frequent errors

| Symptom | Likely cause and fix |
|---|---|
| Branch is not listed | Run **Scan Multibranch Pipeline Now**; verify discovery behavior, credentials, filters, and script path. |
| “Jenkinsfile not found” | Check case and configured script path in that exact branch/ref. |
| “No such DSL method” | Step/plugin is missing or the name is wrong; use the target controller’s Snippet Generator. |
| “Invalid agent type” | Required agent plugin is absent or syntax differs for that installed version. |
| Stage waits forever for an agent | Label expression matches no online executor, or cloud provisioning is failing. |
| `params.X` is missing on first run | Job properties have not yet been created, or parameter name/case differs. |
| Files disappear between stages | Stages ran in different workspaces/containers; use checkout, stash/unstash, or an artifact store. |
| Secret appears in logs | Groovy interpolation or shell tracing exposed it; rotate the secret, use `withCredentials`, single-quoted Groovy command strings, and `set +x`. |
| Docker command cannot connect | Agent lacks Docker access or correct daemon/socket configuration. |
| `when { branch ... }` never matches a PR | `BRANCH_NAME` may be `PR-42`; use `changeRequest target: 'main'` for PR metadata. |
| Matrix syntax fails | Pipeline: Declarative version is too old, an axis is malformed, or unsupported nesting is used. |
| Pipeline cannot resume | `disableResume()` was selected, state was not persisted, or a non-serializable object crossed a suspension point. |

### 22.4 Jenkinsfile Runner and unit tests

For deeper automation, teams often use one or more of:

- Jenkinsfile Runner for a controlled Jenkins/plugin environment.
- Jenkins Pipeline Unit for unit-testing Shared Library functions with mocked steps.
- A disposable Jenkins controller created from configuration as code.
- Repository scripts tested independently of Jenkins.

These tools complement, rather than replace, a real multibranch integration test with the production plugin set.

## 23. Security and production practices

- Run builds on agents, not the built-in/controller node.
- Use least-privilege, short-lived credentials and separate read, publish, and deploy identities.
- Never commit secrets to the `Jenkinsfile` or pass secrets through build parameters when Credentials can be used.
- Scope `withCredentials` tightly and never assume masking defeats malicious code.
- Keep untrusted pull-request builds away from secrets, host mounts, production networks, and privileged containers.
- Pin container images by immutable digest where reproducibility and supply-chain integrity matter.
- Bound every external call with timeouts and use retries only for genuinely transient/idempotent operations.
- Disable uncontrolled concurrency for deployments or use a lockable resource.
- Set build and artifact retention.
- Archive only required files; broad patterns can capture secrets or huge workspaces.
- Prefer `checkout scm` in multibranch jobs so Jenkins checks out the discovered revision.
- Put complex commands in versioned scripts with strict shell settings and their own tests.
- Use `set -eu` or an appropriate stricter mode, but understand pipeline/command exceptions before enabling `pipefail` globally.
- Do not interpolate untrusted parameters directly into shell commands; validate them or pass them through environment variables and quote them.
- Approvals are governance controls, not a substitute for access control on agents and credentials.
- Keep Jenkins core and plugins current and test plugin upgrades in a staging controller.

Safer parameter handling example:

```groovy
stage('Deploy') {
    when {
        expression { params.TARGET in ['dev', 'staging', 'production'] }
    }
    environment {
        SAFE_TARGET = "${params.TARGET}"
    }
    steps {
        sh './deploy.sh "$SAFE_TARGET"'
    }
}
```

## 24. Quick syntax cheat sheet

```groovy
pipeline {
    agent any | none | { label '...' } | { node { ... } } |
          { docker { ... } } | { dockerfile { ... } } | { kubernetes { ... } }

    environment { KEY = 'value'; SECRET = credentials('id') }
    options { timeout(...); retry(...); timestamps(); /* ... */ }
    parameters { string(...); text(...); booleanParam(...); choice(...); password(...) }
    triggers { cron('...'); pollSCM('...'); upstream(...) }
    tools { jdk 'name'; maven 'name'; gradle 'name' }

    stages {
        stage('Name') {
            agent { /* optional override */ }
            environment { /* stage scope */ }
            options { /* stage scope */ }
            tools { /* stage scope */ }
            input { /* approval before agent */ }
            when { /* branch/change/tag/expression/... */ }

            steps { /* Pipeline steps */ }
            // OR stages { stage('Child') { ... } }
            // OR parallel { stage('A') { ... }; stage('B') { ... } }
            // OR matrix { axes { ... }; excludes { ... }; stages { ... } }

            post { /* stage post conditions */ }
        }
    }

    post {
        always { /* ... */ }
        changed { /* ... */ }
        fixed { /* ... */ }
        regression { /* ... */ }
        aborted { /* ... */ }
        failure { /* ... */ }
        success { /* ... */ }
        unstable { /* ... */ }
        unsuccessful { /* ... */ }
        cleanup { /* runs last */ }
    }
}
```

> The semicolons above are only compact cheat-sheet separators. In real Declarative Pipeline code, put statements on separate lines and omit semicolons.

## 25. Official references

- [Pipeline Syntax reference](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [Branches and Pull Requests (Multibranch Pipeline)](https://www.jenkins.io/doc/book/pipeline/multibranch/)
- [Using a Jenkinsfile](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/)
- [Pipeline as Code](https://www.jenkins.io/doc/book/pipeline/pipeline-as-code/)
- [Pipeline Steps reference](https://www.jenkins.io/doc/pipeline/steps/)
- [Pipeline Examples](https://www.jenkins.io/doc/pipeline/examples/)
- [Extending with Shared Libraries](https://www.jenkins.io/doc/book/pipeline/shared-libraries/)
- [Jenkins security documentation](https://www.jenkins.io/doc/book/security/)

---

### Final checklist

- [ ] `Jenkinsfile` is committed at the configured path.
- [ ] Branch-source plugin and repository credentials are configured.
- [ ] Branch, pull-request, and tag discovery policies are intentional.
- [ ] Webhook delivery succeeds; periodic indexing is configured as a fallback.
- [ ] Every agent label and tool name exists.
- [ ] Credential IDs exist and are scoped to the correct folders/jobs.
- [ ] Pull requests from forks cannot reach trusted secrets or privileged agents.
- [ ] Build timeout, concurrency, retention, and orphaned-item policies are set.
- [ ] Test reports and artifacts are published even when relevant stages fail.
- [ ] Deployment conditions distinguish branches, change requests, and tags.
- [ ] The file passes the target controller’s Declarative linter.
- [ ] Success, failure, abort, retry, and restart behavior have been tested.
