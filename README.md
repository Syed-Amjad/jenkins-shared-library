# Jenkins shared library

Reusable, versioned Jenkins Pipeline steps you can drop into any project. This repo lets teams standardize CI/CD logic (build, test, security scans, deploy) once, then reuse it across many pipelines — reliably and repeatably.

---

## Why this exists

- Centralize pipeline logic in one place.
- Keep application repos clean and focused on code, not CI boilerplate.
- Ship consistent pipelines across teams, clouds, and projects.
- Version-control your pipeline behavior and roll it back like application code.

---

## What’s inside

- vars/ — Global steps (simple, easy-to-call functions usable directly in Jenkinsfiles).
- src/org/example — Namespaced Groovy classes for advanced logic or reusable utilities.
- Jenkinsfile — A sample pipeline (useful for self-testing the library or demos).
- info1.txt — Placeholder/example file (safe to remove or replace).

Tip: The exact step names you can call are the filenames in vars/ without the .groovy extension. Open each file to see parameters and behavior.

---

## Prerequisites

- Jenkins 2.400+ (earlier may work, but aim modern)
- Plugins:
  - Pipeline
  - Git
  - Script Security
  - Credentials
  - (Optional) Job DSL (only if you’ll use a seed job)
  - (Optional) GitHub / GitHub Branch Source (if using GitHub webhooks or multibranch)
- Access to this repo URL (public or via a Git credential if private)

---

## Quick start: Use this library in your Jenkins

1. Configure the shared library (one-time)
   - Manage Jenkins → System → Global Pipeline Libraries → Add
     - Name: jenkins-shared-library (this is the identifier you’ll use in pipelines)
     - Default version: main or master (whichever this repo uses)
     - Retrieval method: Modern SCM → Git
     - Project repository: https://github.com/Syed-Amjad/jenkins-shared-library.git
     - Credentials: select if repo is private
   - Save

2. Reference the library in any Jenkinsfile
   - Add at the top of your pipeline:
     ```
     @Library('jenkins-shared-library') _
     ```
   - Call the global steps defined under vars/. Example template:
     ```groovy
     @Library('jenkins-shared-library') _
     pipeline {
       agent any
       stages {
         stage('Use shared step') {
           steps {
             script {
               // Replace `myStep` with an actual step name from vars/, e.g., `dockerBuild`, `notifySlack`, etc.
               // Example: myStep(param1: 'value', flag: true)
             }
           }
         }
       }
     }
     ```
   - Open vars/ to see what steps are available and what parameters they accept.

---

## Recommended repo layout (for consumers)

Your application repos should include a minimal Jenkinsfile and call shared steps, for example:
```groovy
@Library('jenkins-shared-library') _
pipeline {
  agent any
  options { timestamps() }
  stages {
    stage('Build') {
      steps {
        // e.g., buildApp(), mavenBuild(), gradleBuild(), etc. — replace with steps in vars/
      }
    }
    stage('Test') {
      steps {
        // e.g., runTests()
      }
    }
    stage('Deploy') {
      when { branch 'main' }
      steps {
        // e.g., deployTo('staging')
      }
    }
  }
}
```

---

## Optional: Automate job creation with a seed job (Job DSL)

If you want Jenkins to auto-create a pipeline job that points to an app repo:

1. Create a simple seed job (Freestyle or Pipeline) that runs a Job DSL script.
2. Use a minimal, future-proof DSL (avoid deprecated triggers; rely on webhooks):

```groovy
pipelineJob('pipeline-demo-automated') {
  description('Auto-generated pipeline that consumes the shared library from Jenkins Global Libraries.')
  definition {
    cpsScm {
      scm {
        git {
          remote {
            // Change this to your application repo that contains a Jenkinsfile
            url('https://github.com/your-org/your-app-repo.git')
          }
          branch('*/main')
        }
      }
      scriptPath('Jenkinsfile')
    }
  }
  // Prefer GitHub webhook over SCM polling; configure webhook in GitHub to call:
  // http(s)://<JENKINS_URL>/github-webhook/
}
```

After you run the seed job:
- Jenkins will create pipeline-demo-automated.
- Pushes to your app repo will trigger builds via GitHub webhook (set up below).

Note on approvals: If Script Security flags the DSL, approve it in Manage Jenkins → In-process Script Approval.

---

## Webhook setup (recommended)

- In your GitHub repository:
  - Settings → Webhooks → Add webhook
  - Payload URL: http(s)://<JENKINS_URL>/github-webhook/
  - Content type: application/json
  - Events: Just the push event (or what you need)
  - Save

- In Jenkins:
  - Ensure GitHub plugin(s) are installed.
  - In your pipeline job, you don’t need SCM polling; the webhook is enough.

This avoids duplicate builds and deprecation warnings around old trigger syntax.

---

## Avoid duplicate builds

If you’re seeing both an “older pipeline” and your new auto-generated job building on the same push:

- Disable SCM polling on the old job (uncheck “Poll SCM”).
- Ensure only one job is wired to the repo’s webhook.
- If you must keep both, scope them to different branches or different Jenkinsfile paths.
- For migration, keep the old job disabled but retained for reference.

---

## Using this repo with Terraform (infra-as-code flow)

If your infra repo spins up Jenkins and bootstraps jobs:

- Inputs you’ll typically parameterize:
  - jenkins_url (or VM IP/DNS)
  - jenkins_admin_user / token
  - shared_library_repo_url (this repo)
  - app_repo_url (the repo with the Jenkinsfile that consumes the library)
  - optionally: webhook_secret, credentials IDs

- High-level steps your module might perform:
  1. Provision Jenkins (VM/Container).
  2. Install required plugins.
  3. Configure Global Pipeline Library to point at this repo.
  4. Create a seed job (runs the DSL shown above) or directly create the pipeline job via Jenkins REST/CLI.
  5. Optionally register a GitHub webhook (or output instructions).

With this, “anyone” can deploy the same setup by supplying:
- The Jenkins endpoint (IP/DNS),
- This library’s repo URL,
- The application repo URL,
- And appropriate credentials for private repos.

---

## Development workflow for the library

- Branching/versioning:
  - Use semantic versions via tags (e.g., v1.2.0).
  - In Jenkins Global Library config, pin consumers to a stable version, not main, for predictability.
- Backward compatibility:
  - Prefer additive changes to vars/ steps.
  - When breaking changes are needed, release a new major version.
- Testing changes:
  - Keep the Jenkinsfile in this repo as a quick self-test.
  - Optionally maintain a small demo app repo (see below) to verify end-to-end usage.

---

## Troubleshooting

- “script not yet approved for use”
  - Manage Jenkins → In-process Script Approval → approve pending signatures.
- “triggers is deprecated”
  - Remove old DSL trigger blocks; prefer webhooks.
- Library not found in pipeline
  - Ensure the library name in @Library('name') matches the Global Pipeline Libraries entry.
  - Check the default version/branch and credentials.
- Cannot access private repos
  - Add a Git credential in Jenkins and attach it to the library and/or pipeline SCM.

---

## FAQ

### Do I need a second repo like jenkins-pipeline-demo?
- Purpose: A small “consumer” repo with a Jenkinsfile that imports this library. It’s used to:
  - Demonstrate how to call the library’s steps.
  - Validate changes to the library end-to-end.
  - Provide a working example for teammates or clients.
- Is it necessary? Not strictly. If you already have application repos that will consume the library, you can use those directly. However, having a minimal demo repo makes onboarding, testing, and portfolio demos much easier and safer (no risk to production code).

### Why separate the library from application repos?
- Clean boundaries: app repos focus on product code; library repo focuses on CI/CD behavior.
- Version control: upgrade pipelines by bumping a library version instead of rewriting each Jenkinsfile.
- Reliability: one fix in the library can benefit many pipelines.

---
