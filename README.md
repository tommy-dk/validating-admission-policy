# Evaluating Validating Admission Policy in Kubernetes 1.27 with input parameters from CRD

## Introduction

In [Kubernetes 1.27 an alpha feature](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/) was introduced that will offer an alternative to validating admission webhooks by having it directly in Kubernetes. This feature uses the CEL language (Commen Expression Language) and gives the cluster administrator the powers to create admission policies for using the Kubernetes cluster.

In this guide I explore how to utilise this new feature paired with using a Custom Resource Definition (CRD) as input parameters for easy customisation of the policies.

ValidatingAdmissionPolicy is a native alternative to the webhooks-based [OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper) and [Kyverno](https://kyverno.io/).

## Setting up Kubernetes 1.27 locally

As I'm trying to evaluate an alpha feature, we need to setup [kind](https://github.com/kubernetes-sigs/kind) with the following config file:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
featureGates:
  "ValidatingAdmissionPolicy": true
runtimeConfig:
  "admissionregistration.k8s.io/v1alpha1": true
nodes:
- role: control-plane
  image: kindest/node:v1.27.0
- role: worker
  image: kindest/node:v1.27.0
```

Start the cluster with:
```sh
kind create cluster --config kind-config.yaml
```

You now have a Kubernetes 1.27 cluster named `kind` with the needed features enabled.

## Creating your first policy

I want to test whether a Deployment have less than or equal 3 in `spec.replicas` as this is a test cluster and there's no need to run many replicas for testing purposes.

First I create a `ValidatingAdmissionPolicy` that matches all `Deployments` that are either `CREATE`'d or `UPDATE`'d. Then I give the actual validation that disallows `spec.replicas > 3`:

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicy
metadata:
  name: max-replicas
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   ["apps"]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["deployments"]
  validations:
    - expression: "object.spec.replicas <= 3"
      message: "spec.replicas must be no greater than 3"
      reason: Invalid
```

Binding the policy to Kubernetes and matches it to all namespaces in Kubernetes:

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: max-replicas
spec:
  policyName: max-replicas
  matchResources: # matches all namespaces in k8s
```

Apply it to the Kubernetes:

```sh
cd max-replicas-simple
kubectl apply -k .
```

Test out a deployment with more than 3 replicas set:

```sh
kubectl apply -f test-deployment.yaml
```
The output will then be:

> ```The deployments "test-deployment" is invalid: : ValidatingAdmissionPolicy 'max-replicas' with binding 'max-replicas' denied request: spec.replicas must be no greater than 3```

The policy denied the `Deployment` to be created because it didn't match the rule we created.

This is really neat and simple, but what if I want to give different input parameters depending to my policies?

## Extending with input parameters from CRD

The policy above is useful, but by having constants in the code it is not very extensible. So the cool thing with `ValidatingAdmissionPolicy` is that we can extend it by specifying parameters via a Custom Resource Definition (CRD).

I first created a CRD called `parameters.example.com`:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: parameters.example.com
  annotations:
    admission.kubernetes.io/is-policy-configuration-definition: "true"
spec:
  group: example.com
  names:
    kind: parameters
    plural: parameters
  versions:
  - name: v1
    schema:
      openAPIV3Schema:
        type: object
        properties:
          parameters:
            type: object
            properties:
              maxReplicas:
                type: integer
              revisionHistoryLimit:
                type: integer
              allowedProductKeys:
                type: array
                items:
                  type: string
                  properties:
                    product:
                      type: string
              bannedTags:
                type: array
                items:
                  type: string
                  properties:
                    tag:
                      type: string
              bannedRegistries:
                type: array
                items:
                  type: string
                  properties:
                    registry:
                      type: string
              allowedEnvLabels:
                type: array
                items:
                  type: string
                  properties:
                    envLabel:
                      type: string
    served: true
    storage: true
  scope: Cluster
```

This creates the following input parameters structure:

```yaml
apiVersion: example.com/v1
kind: parameters
metadata:
  name: validatingadmissionpolicyparameters
parameters:
  maxReplicas: 3
  revisionHistoryLimit: 3
  allowedProductKeys:
    - foo
    - bar
    - foobar
  bannedTags:
    - latest
  bannedRegistries: 
    - docker.io
    - gcr.io
  allowedEnvLabels:
    - dev
    - test
    - uat
    - cdt
    - preprod
    - prod
    - perf
    - poc
    - qa
    - sandbox
    - shadow
    - sit
    - spt
    - stage
```

We now have a way to bring parameters into our `expressions` in the CEL language.

Looking at the `prod-ready` folder, we can now create a `ValidatingAdmissionPolicy` and extend it with the following:

```yaml
  paramKind:
    apiVersion: example.com/v1
    kind: parameters
```

With this we now have a way to bring parameters into our `expressions` in the CEL language via the `params.parameters.<name>` variable.

Looking at the first rule, it's easy to see how to get rid of a constant and replace it with a variable:
```yaml
    - expression: "object.spec.replicas < params.parameters.maxReplicas"
      message: "You need at least 3 replicas for running in production"
```

:exclamation: Caveat! `message` still can't understand variables, but [there's work going on](https://github.com/kubernetes/kubernetes/commit/4e26f680a9e10f0da94830bbaba9633807e22aba) to extend it and create a `messageExpression` which understands variables.

### Multiple input parameters to the same rule

The second rule is a little more advanced, since I'm iterating over a list called `params.parameters.bannedRegistries` and comparing them on by one with the starting value of `spec.template.spec.containers`:

```yaml
    - expression: |
        params.parameters.bannedRegistries.all(
          registry, object.spec.template.spec.containers.exists_one(
            container, !(container.image.startsWith(registry))
          )
        )
      message: "Image is coming from an untrusted registry"
```

And since the bannedRegistries list is:
```yaml
   bannedRegistries: 
    - docker.io
    - gcr.io
```

All pods where a container is using an image from one of those registries will be denied to be deployed to Kubernetes.

The last rule:

```yaml
    - expression: "has(object.metadata.labels.env) && object.metadata.labels.env in params.parameters.allowedEnvLabels"
      message: "Deployment should have an environment label called `env` and set to the allowed naming standards"
```

Here I am comparing the label `env` with a list of allowed labels from the parameters CRD. I see whether the pods label is in the list. If not the `ValidatingAdmissionPolicy` fails the deployment.

The binding for these policies is bound like this, where it will only match namespaces with the label `environment=prod`:

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: prod-ready
spec:
  policyName: prod-ready
  paramRef:
    name: "validatingadmissionpolicyparameters"
  matchResources:
    namespaceSelector:
      matchExpressions:
      - key: environment
        operator: In
        values:
        - prod
```

Before testing out the deployment, configure the above label on the namespace:

```sh
kubectl label namespace default environment=prod
```

Test out the rules with the `test-deployment.yaml`:
```sh
cd prod-ready
kubectl apply -k .
kubectl apply -f test-deployment.yaml
```
The output will then be:
> ```The deployments "test-deployment" is invalid: : ValidatingAdmissionPolicy 'prod-ready' with binding 'prod-ready' denied request: Image is coming from an untrusted registry```

Try to play around with the parameters as well as the values in `test-deployment.yaml`.

## Conclusion

From my evalution `ValidatingAdmissionPolicy` is a really powerful engine utilising the CEL language which is very easy to write and understand. I found it very easy to write policies and extending them with a Custom Resource Definition, and from my viewpoint this will quickly make [OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper) and [Kyverno](https://kyverno.io/) redundant once this is out of alpha.