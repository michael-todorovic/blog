---
title: Don't lose time with Helm unit tests
description: How to scale your chart testing
date: 2025-05-17 00:00:00+0000
image: cover.jpg
categories:
    - helm
    - kubernetes
    - gitlab
tags:
    - unit tests
---

## Introduction

[Helm](https://helm.sh) is a powerful tool and de-facto standard to manage Kubernetes applications. With time, charts become complex and you really need to test your charts to avoid regressions. There are several ways and levels to do so. Here I'll explain what we finally do at my company with [helm-unittest](https://github.com/helm-unittest/helm-unittest).

At first, we followed the documentation and wrote stuff like:

```yaml
suite: test deployment
templates:
  - deployment.yaml
tests:
  - it: should work
    set:
      image.tag: latest
    asserts:
      - isKind:
          of: Deployment
      - matchRegex:
          path: metadata.name
          pattern: -my-chart$
      - equal:
          path: spec.template.spec.containers[0].image
          value: nginx:latest
```

We added such tests when creating a new feature or fix. This method searches for very specific items into the chart, which is nice.

However, with time, we found out that it doesn't scale well and more importantly, this doesn't cover regressions well enough. This concentrates on very specific checks: what if the value (`image.tag` here) you set in the test is used somewhere else in the chart? You might have forgotten it after weeks/months or a new contributor might not know it. This means one can break something without knowing it and make your end-users complain.

![Remove an unused line, break everything](code-removal.jpg)

Hopefully, helm-unittest supports [snapshot testing](https://github.com/helm-unittest/helm-unittest?tab=readme-ov-file#snapshot-testing)! Just like other tools in the industry (like [jest](https://jestjs.io/docs/snapshot-testing)), it will compare the fully rendered chart to a snapshot taken earlier. Any difference in the rendered chart will be reported so you make sure that touching that one little thing isn't breaking anything somewhere else in the chart.

## Snapshot testing

### Create chart

To make it easy, we're going to create a basic chart:

```bash
helm create example-chart
```

This will create a regular chart on which we'll add our tests:

```
example-chart
├── charts
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```

Now we need to add our tests.

{{< notice tip >}}
You can find the chart source [here](https://github.com/michael-todorovic/blog/tree/main/content/post/helm-unittest/example-chart).
{{< /notice >}}

### Tests structure

In order to test, we'll add this structure to our chart:

```
example-chart
├── tests
│   ├── snap_deployment.yaml
│   └── values
│       └── base.yaml
```

- `tests/snap_deployment.yaml`: describes what to test. I'm following this naming: `<testMethod>_<k8sObject>`
- `tests/values/`: this folder will hold several Helm values so we can re-use some of them, depending on what we want to test. This presents the advantage of having stable values across tests so there are no surprises due to a typo or something.

### Add tests

{{< notice info >}}
You'll need to first install Helm and [helm-unittest](https://github.com/helm-unittest/helm-unittest?tab=readme-ov-file#install) plugin
{{< /notice >}}

Create a `tests/snap_deployment.yaml`:

{{< code language="yaml" source="content/post/helm-unittest/example-chart/tests/snap_deployment.yaml" >}}

And a simple `values/base.yaml`:

{{< code language="yaml" source="content/post/helm-unittest/example-chart/tests/values/base.yaml" >}}

You can now execute the tests to generate a first snapshot:

```bash
╰ helm unittest -f 'tests/*.yaml' .

### Chart [ example-chart ] .

 PASS  Deployment	tests/snap_deployment.yaml

Charts:      1 passed, 1 total
Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshot:    2 passed, 2 total
Time:        1.913192ms
```

You should now have a `tests/__snapshot__/snap_deployment.yaml.snap`. This is how Helm templated your chart with the provided values.

You will need to review the manifests to make sure it's the result you expect. If you're happy with it, you can commit the whole thing, including the snap files.

### Break tests

You'll now update `tests/values/base.yaml`:

```yaml
replicaCount: 2
```

and run the tests again:

```bash
╰ helm unittest -f 'tests/*.yaml' .

### Chart [ example-chart ] .

 FAIL  Deployment	tests/snap_deployment.yaml
	- test base

		- asserts[0] `matchSnapshot` fail
			Template:	example-chart/templates/deployment.yaml
			DocumentIndex:	0
			Path:	
			Expected to match snapshot 1:
				--- Expected
				+++ Actual
				@@ -11,3 +11,3 @@
				 spec:
				-  replicas: 1
				+  replicas: 2
				   selector:


Snapshot Summary: 1 snapshot failed in 1 test suite. Check changes and use `-u` to update snapshot.

Charts:      1 failed, 0 passed, 1 total
Test Suites: 1 failed, 0 passed, 1 total
Tests:       1 failed, 0 passed, 1 total
Snapshot:    1 failed, 1 passed, 2 total
Time:        2.421824ms

Error: plugin "unittest" exited with error
```

The `replicas` aren't matching the previous snapshot so helm-unittest is not happy 😭

You now have several choices:

- fix the chart
- fix the values
- if the change is expected, [update the snapshots](#update-snapshots)

### Values management

Earlier, we saw we can read from values files. It's a convenient way but sometimes, it would be nice to just use that values file and set one value. It's super easy to do:

```yaml
suite: Deployment
templates:
  - deployment.yaml
tests:
  - it: test noReplica
    values:
      - ./values/base.yaml
    asserts:
      - matchSnapshot: {}
    set:
      replicaCount: 0
```

### Update snapshots

You made a change and you need to update shapshots. You could update them manually but it is recommended to refresh them with the cli:

- update a single snapshot: `helm unittest -f tests/snap_deployment.yaml . -u`
- update them all: `helm unittest -f 'tests/*.yaml' . -u`

{{< notice tip >}}
It is advised to refresh snapshots only when they're not known as modified by git. It will easier for you to detect changes from a clean state.
{{< /notice >}}


## Gitlab job

I recommend to have those unit tests running on your PRs/MRs. Here's an example for Gitlab that also reports the tests results in the MR, thanks to jUnit export format:

```yaml
variables:
  HELM_WORKING_DIR: ${CI_PROJECT_DIR}

test:helm:unittests:
  stage: test
  image:
    name: helmunittest/helm-unittest:3.17.3-0.8.2
    entrypoint: [""]
  script:
    - cd ${HELM_WORKING_DIR}
    - helm dep update
    - helm unittest -f 'tests/*.yaml' . -t jUnit -o junit-results.xml
  artifacts:
    when: always
    paths:
      - junit-results.xml
    reports:
      junit: junit-results.xml
    expire_in: 1 week
  rules:
    - if: $CI_COMMIT_TAG
      when: never
    - if: '$CI_COMMIT_BRANCH == "master"'
      when: never
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: never
    - exists:
        - tests/*.yaml
```
