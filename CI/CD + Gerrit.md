# CI/CD + Gerrit Crash Course
### Interview Prep — Automation Engineer Role

---

## 1. What Is CI/CD (Big Picture)

```
Developer pushes code
       ↓
  Code Review (Gerrit)
       ↓
  CI Pipeline triggers (Jenkins / GitLab CI / Buildbot)
       ↓
  Build → Test → Package → Deploy
       ↓
  Metrics/Monitoring (Prometheus, Grafana, ELK)
```

**CI (Continuous Integration):** Every commit is automatically built and tested.  
**CD (Continuous Delivery/Deployment):** Validated builds are automatically packaged or deployed.

---

## 2. Jenkins — Core Concepts

### Key Terminology

| Term | Meaning |
|---|---|
| **Job / Project** | A named CI task (build, test, deploy) |
| **Pipeline** | A scripted sequence of stages defined in a Jenkinsfile |
| **Node / Agent** | A machine that runs the job |
| **Stage** | A logical phase (Build, Test, Deploy) |
| **Step** | A single command within a stage |
| **Executor** | A thread slot on an agent; determines parallelism |
| **Workspace** | The directory on the agent where the job runs |
| **Artifact** | A file produced by the job (binary, .zip, .deb) |
| **Upstream/Downstream** | Jobs that trigger / are triggered by other jobs |

### Declarative Jenkinsfile (memorise this pattern)

```groovy
pipeline {
    agent any                          // Run on any available agent
    environment {
        REPO_URL = "https://android.googlesource.com/platform/manifest"
    }
    stages {
        stage('Sync Source') {
            steps {
                sh 'repo init -u $REPO_URL -b main'
                sh 'repo sync -c -j8'          // -c: current branch only, -j8: 8 threads
            }
        }
        stage('Build') {
            steps {
                sh '. build/envsetup.sh && lunch aosp_arm-eng && make -j$(nproc)'
            }
        }
        stage('Test') {
            steps {
                sh 'python3 run_tests.py --output results.xml'
            }
            post {
                always {
                    junit 'results.xml'        // Publish test results
                }
            }
        }
        stage('Publish Artifact') {
            steps {
                archiveArtifacts artifacts: 'out/target/product/**/*.img', fingerprint: true
            }
        }
    }
    post {
        failure {
            mail to: 'team@company.com', subject: "Build Failed: ${env.JOB_NAME}"
        }
    }
}
```

### Jenkins + Gerrit Integration

Jenkins listens to Gerrit events via the **Gerrit Trigger Plugin**.

Flow:
```
Developer pushes patchset to Gerrit
      ↓
Gerrit fires 'patchset-created' event
      ↓
Jenkins picks up the event via Gerrit Trigger Plugin
      ↓
Jenkins builds + tests the patchset
      ↓
Jenkins votes: +1 Verified (pass) or -1 Verified (fail)
      ↓
Gerrit shows vote; human reviewer adds Code-Review +2
      ↓
Change is submitted (merged)
```

**Key Gerrit Trigger environment variables in Jenkins:**
```
GERRIT_CHANGE_ID, GERRIT_PATCHSET_REVISION, GERRIT_BRANCH,
GERRIT_PROJECT, GERRIT_REFSPEC
```

---

## 3. GitLab CI — Core Concepts

### Key Terminology

| Term | Meaning |
|---|---|
| **Pipeline** | A complete CI/CD run triggered by a push/MR |
| **Stage** | A group of jobs that run in parallel |
| **Job** | A single unit of work with a script block |
| **Runner** | An agent that executes jobs (shared or specific) |
| **.gitlab-ci.yml** | The pipeline definition file at repo root |
| **Artifact** | Files persisted between jobs or for download |
| **Cache** | Directories reused between runs to speed up builds |
| **Environment** | A named deployment target (staging, production) |

### Sample .gitlab-ci.yml

```yaml
stages:
  - sync
  - build
  - test
  - deploy

variables:
  ANDROID_HOME: "/opt/android-sdk"

sync_source:
  stage: sync
  script:
    - repo init -u $MANIFEST_URL -b $CI_COMMIT_BRANCH
    - repo sync -c -j8
  cache:
    paths:
      - .repo/

build_image:
  stage: build
  script:
    - source build/envsetup.sh
    - lunch $TARGET
    - make -j$(nproc)
  artifacts:
    paths:
      - out/target/product/
    expire_in: 1 day

run_tests:
  stage: test
  script:
    - python3 test_runner.py --xml report.xml
  artifacts:
    reports:
      junit: report.xml

deploy_staging:
  stage: deploy
  environment:
    name: staging
  script:
    - ./deploy.sh staging
  only:
    - main
```

---

## 4. Buildbot — Core Concepts

Buildbot is **Python-based** and common in embedded/Android build environments.

| Concept | Description |
|---|---|
| **Master** | Central controller — schedules builds, stores results |
| **Worker** | Machine that actually runs the build |
| **Builder** | Named build config (associates scheduler + worker + steps) |
| **BuildStep** | A single action (ShellCommand, Git, Compile) |
| **Scheduler** | Decides when to trigger a build (on push, on timer, manually) |
| **Change Source** | Watches VCS for changes (Gerrit, Git poller) |

### Minimal Buildbot config snippet

```python
from buildbot.plugins import *

c = BuildmasterConfig = {}
c['workers'] = [worker.Worker("worker1", "password")]
c['schedulers'] = [
    schedulers.SingleBranchScheduler(
        name="all",
        change_filter=util.ChangeFilter(branch='main'),
        builderNames=["android-build"]
    )
]

factory = util.BuildFactory()
factory.addStep(steps.Git(repourl='ssh://gerrit/platform/manifest', mode='full'))
factory.addStep(steps.ShellCommand(command=["make", "-j8"]))

c['builders'] = [
    util.BuilderConfig(name="android-build", workernames=["worker1"], factory=factory)
]
```

---

## 5. Gerrit — Deep Dive

### What Is Gerrit?

Gerrit is a **web-based code review system** built on top of Git. Every change goes through a patchset → review → verified → submit workflow before it lands in the main branch.

### Key Terminology

| Term | Meaning |
|---|---|
| **Change** | A proposed commit under review (analogous to a PR) |
| **Patchset** | A version of a change; each `git push` creates a new patchset |
| **Ref** | A special Git ref like `refs/changes/45/12345/3` |
| **Code-Review (CR)** | Human vote: -2, -1, 0, +1, +2 |
| **Verified (V)** | CI vote: -1 (fail) or +1 (pass) |
| **Submit** | Merge the change into the target branch |
| **Abandon** | Close the change without merging |
| **Topic** | A label grouping related changes across repos |
| **Relation chain** | A stack of dependent changes (parent → child) |
| **LGTM** | Informal "Looks Good To Me" — needs +2 to actually submit |

### Gerrit Workflow (step by step)

```
1. Clone repo from Gerrit:
   git clone ssh://user@gerrit:29418/platform/frameworks/base

2. Create a local branch:
   git checkout -b my-fix

3. Make changes and commit with a Change-Id:
   git commit -m "Fix camera pipeline crash

   Bug: 12345
   Change-Id: I1234567890abcdef1234567890abcdef12345678"

4. Push for review (NOT to main branch — to refs/for/):
   git push origin HEAD:refs/for/main

5. Gerrit creates a Change at:
   https://gerrit.example.com/c/platform/frameworks/base/+/12345

6. CI (Jenkins) builds and votes Verified +1

7. Reviewer adds Code-Review +2

8. Submitter clicks Submit → change merges
```

### The Change-Id

The `Change-Id` line in the commit message is **how Gerrit tracks patchsets**. If you amend and re-push, Gerrit creates a new patchset on the same change (not a new change) because the Change-Id is the same.

```bash
# Install the commit-msg hook so Change-Id is auto-generated:
scp -p -P 29418 user@gerrit:hooks/commit-msg .git/hooks/
```

### Gerrit Push Refspecs

```bash
# Push for review on branch 'main'
git push origin HEAD:refs/for/main

# Push with a topic
git push origin HEAD:refs/for/main%topic=camera-fix

# Push as a WIP (Work In Progress)
git push origin HEAD:refs/for/main%wip

# Push bypassing review (requires special permissions)
git push origin HEAD:refs/heads/main
```

### Gerrit REST API (used for automation)

```bash
# Get change details
curl -u user:password \
  https://gerrit.example.com/a/changes/12345

# Vote on a change (Verified +1 from CI)
curl -X POST -u user:password \
  -H "Content-Type: application/json" \
  -d '{"labels":{"Verified":1},"message":"Build passed"}' \
  https://gerrit.example.com/a/changes/12345/revisions/current/review

# Submit a change
curl -X POST -u user:password \
  https://gerrit.example.com/a/changes/12345/submit
```

### Gerrit SSH Commands (used in scripts)

```bash
# Query open changes
ssh -p 29418 user@gerrit gerrit query --format=JSON status:open project:platform/frameworks/base

# Approve and submit
ssh -p 29418 user@gerrit gerrit review --verified +1 --code-review +2 --submit COMMIT_SHA

# Stream events (used by Jenkins Gerrit Trigger)
ssh -p 29418 user@gerrit gerrit stream-events
```

---

## 6. repo Tool — Android Source Management

`repo` is a Python wrapper around Git for managing **multi-repository Android builds**.

### Key Commands

```bash
# Initialize with a manifest
repo init -u https://android.googlesource.com/platform/manifest -b android-14.0.0_r1

# Sync all repos (fetch + checkout)
repo sync -c -j8              # -c: current branch only; -j8: 8 parallel jobs

# Sync a single project
repo sync frameworks/base

# Show status across all repos
repo status

# Start a new feature branch across all repos
repo start my-feature --all

# Upload to Gerrit for review
repo upload

# Show diff across all repos
repo diff

# Abandon a branch
repo abandon my-feature
```

### Manifest XML Structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <remote name="aosp"
          fetch="https://android.googlesource.com"
          review="https://android-review.googlesource.com"/>
  <default revision="android-14.0.0_r1"
            remote="aosp"
            sync-j="4"/>
  <project path="frameworks/base"
           name="platform/frameworks/base"/>
  <project path="packages/apps/Camera2"
           name="platform/packages/apps/Camera2"
           revision="camera-stable"/>
</manifest>
```

---

## 7. Linux Build Systems

### The compilation chain

```
Source (.c / .cpp)
      ↓  [Preprocessor: #include, #define expansion]
Preprocessed (.i)
      ↓  [Compiler: gcc/clang]
Assembly (.s)
      ↓  [Assembler: as]
Object file (.o)
      ↓  [Linker: ld]
Executable / .so / .a
```

### Make

```makefile
CC = gcc
CFLAGS = -Wall -O2

all: main.o utils.o
    $(CC) $(CFLAGS) -o myapp main.o utils.o

main.o: main.c
    $(CC) $(CFLAGS) -c main.c

clean:
    rm -f *.o myapp
```

**Key concepts:** targets, prerequisites, recipes, phony targets (`.PHONY: all clean`), automatic variables (`$@` = target, `$<` = first prerequisite, `$^` = all prerequisites).

### CMake

```cmake
cmake_minimum_required(VERSION 3.10)
project(MyProject)

set(CMAKE_C_STANDARD 11)

add_executable(myapp main.c utils.c)
target_include_directories(myapp PRIVATE include/)
target_link_libraries(myapp pthread)
```

```bash
mkdir build && cd build
cmake ..
make -j$(nproc)
```

### Android.bp (Soong build system)

```python
cc_binary {
    name: "my_daemon",
    srcs: ["main.cpp", "utils.cpp"],
    shared_libs: ["liblog", "libcutils"],
    cflags: ["-Wall", "-Werror"],
}
```

---

## 8. REST APIs + Databases in CI/CD

### Python Flask REST API (typical for CI tooling)

```python
from flask import Flask, jsonify, request
import sqlite3

app = Flask(__name__)

@app.route('/builds', methods=['GET'])
def get_builds():
    conn = sqlite3.connect('ci.db')
    cursor = conn.execute("SELECT id, status, timestamp FROM builds ORDER BY id DESC LIMIT 20")
    builds = [{"id": r[0], "status": r[1], "timestamp": r[2]} for r in cursor.fetchall()]
    conn.close()
    return jsonify(builds)

@app.route('/builds/<int:build_id>', methods=['GET'])
def get_build(build_id):
    conn = sqlite3.connect('ci.db')
    cursor = conn.execute("SELECT * FROM builds WHERE id=?", (build_id,))
    row = cursor.fetchone()
    conn.close()
    if row:
        return jsonify({"id": row[0], "status": row[1], "log": row[2]})
    return jsonify({"error": "not found"}), 404

@app.route('/builds', methods=['POST'])
def create_build():
    data = request.json
    conn = sqlite3.connect('ci.db')
    conn.execute("INSERT INTO builds (project, branch, status) VALUES (?,?,?)",
                 (data['project'], data['branch'], 'pending'))
    conn.commit()
    conn.close()
    return jsonify({"status": "queued"}), 201
```

### Common SQL for CI dashboards

```sql
-- Build pass rate per project
SELECT project, 
       COUNT(*) as total,
       SUM(CASE WHEN status='pass' THEN 1 ELSE 0 END) as passed,
       ROUND(100.0 * SUM(CASE WHEN status='pass' THEN 1 ELSE 0 END) / COUNT(*), 2) as pass_rate
FROM builds
GROUP BY project;

-- Slowest builds in last 7 days
SELECT project, branch, duration_seconds
FROM builds
WHERE created_at > datetime('now', '-7 days')
ORDER BY duration_seconds DESC
LIMIT 10;
```

---

## 9. Docker in CI/CD

### Why Docker in CI?

Reproducible build environments — every build runs in the same container, eliminating "works on my machine" issues.

### Dockerfile for Android build environment

```dockerfile
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    git curl python3 python3-pip \
    openjdk-17-jdk \
    make gcc g++ \
    libncurses5 libssl-dev \
    repo \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build
CMD ["/bin/bash"]
```

```bash
# Build the image
docker build -t android-builder:latest .

# Run a build inside the container
docker run --rm -v $(pwd):/build android-builder:latest \
    bash -c "repo sync && make -j8"
```

### Docker in Jenkins pipeline

```groovy
pipeline {
    agent {
        docker {
            image 'android-builder:latest'
            args '-v /cache/repo:/cache/repo'
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'make -j$(nproc)'
            }
        }
    }
}
```

---

## 10. Monitoring & Metrics

### Common tools

| Tool | Purpose |
|---|---|
| **Prometheus** | Metrics collection (scrapes /metrics endpoints) |
| **Grafana** | Dashboards on top of Prometheus / InfluxDB |
| **ELK Stack** | Elasticsearch + Logstash + Kibana for log analysis |
| **InfluxDB** | Time-series database, great for build durations |
| **Datadog** | All-in-one SaaS monitoring |

### Exposing build metrics in Python

```python
from prometheus_client import Counter, Histogram, start_http_server

build_counter = Counter('ci_builds_total', 'Total builds', ['project', 'status'])
build_duration = Histogram('ci_build_duration_seconds', 'Build duration', ['project'])

# Record a build result
build_counter.labels(project='camera', status='pass').inc()

with build_duration.labels(project='camera').time():
    run_build()

start_http_server(8000)  # Prometheus scrapes http://host:8000/metrics
```

---

## 11. Git Fundamentals (for the interview)

```bash
# Internals
git log --oneline --graph --all     # Visual branch history
git reflog                           # All HEAD movements (recovery tool)
git bisect start                     # Binary search for regression
git bisect bad                       # Current commit is bad
git bisect good v1.0                 # v1.0 was good → git narrows down

# Rebase (used heavily with Gerrit)
git fetch origin
git rebase origin/main               # Replay your commits on top of latest main
git rebase -i HEAD~3                 # Interactive: squash/edit last 3 commits

# Cherry-pick (backporting fixes)
git cherry-pick abc1234              # Apply a specific commit to current branch

# Stash
git stash                            # Save dirty working tree
git stash pop                        # Restore it

# Submodules (you may encounter in embedded repos)
git submodule update --init --recursive
```

---

## 12. Likely Interview Questions + Answers

**Q: What is the difference between CI and CD?**  
CI = automatically build and test every commit. CD = automatically deliver (or deploy) the validated build to an environment.

**Q: How does Gerrit differ from GitHub pull requests?**  
Gerrit uses a single-commit review model (one change = one commit, amended as patchsets). GitHub PRs allow multi-commit branches. Gerrit integrates more tightly with CI via verified votes.

**Q: What is a Jenkinsfile and why is it better than UI-configured jobs?**  
A Jenkinsfile is a pipeline-as-code file stored in the repo. It versions the CI config alongside the source, enables code review on pipeline changes, and avoids configuration drift.

**Q: How does repo sync work internally?**  
repo reads the manifest XML, determines the list of Git projects and their target revisions, and runs `git fetch` + `git checkout` on each project in parallel.

**Q: How would you debug a build that passes locally but fails in CI?**  
Check environment variables, toolchain version, missing dependencies in the container. Reproduce by running the exact Docker image CI uses. Compare `make` output verbosity with `make V=1`. Check if timestamps or incremental builds are the issue.

**Q: How would you expose build metrics to Prometheus?**  
Use `prometheus_client` in Python to define Counters/Histograms, instrument the build script, and expose a `/metrics` HTTP endpoint that Prometheus scrapes.

**Q: What is `refs/for/main` in Gerrit?**  
It is a virtual namespace in Gerrit. Pushing to `refs/for/main` does not directly update the `main` branch — it creates a change for review. The change only lands in `main` after review + CI approval + submit.

---

*Study priority: Gerrit workflow → Jenkins pipeline → repo commands → REST API → Docker*
