# Overview

This is a demo repo for Kyverno.

## Install Kyverno

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno -n kyverno --create-namespace # standalone installation for test/dev
helm install kyverno-policies kyverno/kyverno-policies -n kyverno
```

## Validation

Validation stands as the cornerstone of policy implementation, operating as a binary decision-making mechanism. This process efficiently categorizes resources into two distinct outcomes: compliant resources receive approval and proceed unhindered, while non-compliant resources face rejection. This clear-cut "yes" or "no" approach ensures strict adherence to defined policies, maintaining the integrity and security of the Kubernetes environment. Now let's see it in action.

Apply the validation policy.

```bash
kubectl apply -f validation_policy.yaml
```

Now check all available cluster policies.

```bash
kubectl get clusterpolicy
```

Create a namespace to test the policies.

```bash
kubectl create ns kyverno-policy-tests
```

Try creating a Deployment without the required label.

```bash
kubectl apply -f no_label_deploy.yaml
```

Notice the output:

```bash
Error from server: error when creating "no_label_deploy.yaml": admission webhook "validate.kyverno.svc-fail" denied the request: 

resource Deployment/kyverno-policy-tests/nginx-deployment was blocked due to the following policies 

require-labels:
  autogen-check-team: 'validation error: label ''team'' is required. rule autogen-check-team
    failed at path /spec/template/metadata/labels/team/'
```

Now try again with the required label. Go ahead and create the Deployment with the `proper_label_deploy.yaml` file.

```bash
kubectl apply -f proper_label_deploy.yaml
```

Check the policy report:

```bash
kubectl get policyreport
```

You should see something like this. Notice 13 polices since we downloaded a few other ones with our `kyverno-policies` helm chart.

```bash
NAME                                   KIND         NAME                                PASS   FAIL   WARN   ERROR   SKIP   AGE
ce9410a0-b06b-4668-8ea2-2f0426e80344   Deployment   nginx-deployment                    13     0      0      0       0      3m54s
a983e5bf-0d9b-44a0-94c4-a8b7310d64c8   ReplicaSet   nginx-deployment-64bc596959         13     0      0      0       0      3m28s
bd153041-0443-4079-acde-8302972d66bd   Pod          nginx-deployment-64bc596959-mzxrd   13     0      0      0       0      3m48s
```

Try describing the policyreport to get a more detailed report:

```bash
kubectl describe policyreport bd153041-0443-4079-acde-8302972d66bd
Name:         bd153041-0443-4079-acde-8302972d66bd
Namespace:    kyverno-policy-tests
Labels:       app.kubernetes.io/managed-by=kyverno
Annotations:  <none>
API Version:  wgpolicyk8s.io/v1alpha2
Kind:         PolicyReport
Metadata:
  Creation Timestamp:  2024-08-05T18:11:38Z
  Generation:          2
  Managed Fields:
    API Version:  wgpolicyk8s.io/v1alpha2
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:labels:
          .:
          f:app.kubernetes.io/managed-by:
        f:ownerReferences:
          .:
          k:{"uid":"bd153041-0443-4079-acde-8302972d66bd"}:
      f:results:
      f:scope:
      f:summary:
        .:
        f:error:
        f:fail:
        f:pass:
        f:skip:
        f:warn:
    Manager:    reports-controller
    Operation:  Update
    Time:       2024-08-05T18:11:58Z
  Owner References:
    API Version:     v1
    Kind:            Pod
    Name:            nginx-deployment-64bc596959-mzxrd
    UID:             bd153041-0443-4079-acde-8302972d66bd
  Resource Version:  7338619
  UID:               63d95d62-282f-43ec-9e5d-57743c4c9305
Results:
  Category:  Pod Security Standards (Baseline)
  Message:   validation rule 'adding-capabilities' passed.
  Policy:    disallow-capabilities
  Result:    pass
  Rule:      adding-capabilities
  Scored:    true
  Severity:  medium
  Source:    kyverno
  Timestamp:
    Nanos:    0
    Seconds:  1722881508
  Category:   Pod Security Standards (Baseline)
  Message:    validation rule 'host-namespaces' passed.
  Policy:     disallow-host-namespaces
  Result:     pass
  Rule:       host-namespaces
  Scored:     true
  Severity:   medium
  Source:     kyverno
  Timestamp:
    Nanos:    0
    Seconds:  1722881508
    
...truncated...

  Timestamp:
    Nanos:    0
    Seconds:  1722881508
  Message:    validation rule 'check-team' passed.
  Policy:     require-labels
  Result:     pass
  Rule:       check-team
  Scored:     true
  Source:     kyverno
  Timestamp:
    Nanos:    0
    Seconds:  1722881508
 
Scope:
  API Version:  v1
  Kind:         Pod
  Name:         nginx-deployment-64bc596959-mzxrd
  Namespace:    kyverno-policy-tests
  UID:          bd153041-0443-4079-acde-8302972d66bd
Summary:
  Error:  0
  Fail:   0
  Pass:   13
  Skip:   0
  Warn:   0
Events:   <none>
```

## Mutation

Mutation empowers Kyverno to dynamically alter resources before their admission to the cluster. This powerful feature allows for automatic modifications, enhancing resource configurations on-the-fly.

Now let's test the mutation policy. Check the file `mutation_policy.yaml`. Notice that we will check to see if there is a `team` label. If there isn't, we will add a `team` label to the Deployment.

Check the `redis.yaml` file. It doesn't have a label yet.

Now apply the mutation policy.

```bash
kubectl apply -f mutation_policy.yaml
```

Now apply the `redis.yaml` file.

```bash
kubectl apply -f redis.yaml
```

Run the command below to see the label applied:

```bash
kubectl get pod redis-pod --show-labels
```

Output:

```bash
NAME        READY   STATUS    RESTARTS   AGE   LABELS
redis-pod   1/1     Running   0          93s   team=bravo
```

You can try creating a pod with an existing team label and you'll see that the policy will not mutate the pod.

```bash
kubectl run newredis --image redis -l team=alpha
```

Then get this Pod back and check once again for labels.
```bash
kubectl get pod newredis --show-labels
```

Output:
```bash
NAME       READY   STATUS    RESTARTS   AGE   LABELS
newredis   1/1     Running   0          63s   team=alpha
```

## Generation

Kyverno empowers users to dynamically create new Kubernetes resources by leveraging definitions embedded within policies.

Let's see it in action. Check the `registry_secret.yaml` file for a K8s secret that simulates a real image pull secret.

Apply the secret:

```bash
kubectl apply -f registry_secret.yaml
```

Now apply the following Kyverno policy in the file `registry_secret_policy.yaml`:

```bash
kubectl apply -f registry_secret_policy.yaml
```

Establish a fresh Namespace for policy evaluation:

```bash
kubectl create ns mytestns
```

Inspect the Secrets within this newly created Namespace to verify the presence of regcred:

```bash
kubectl -n mytestns get secret
```

Output:
```bash
NAME      TYPE                             DATA   AGE
regcred   kubernetes.io/dockerconfigjson   1      5s
```

Observe how Kyverno has dynamically generated the regcred Secret in the new Namespace, using the source Secret from the default Namespace as a template. Feel free to modify the source Secret and witness Kyverno's synchronization capabilities as it propagates changes to all generated instances.


## Cleanup

To clean up the resources created in this lab, run the following commands:

```bash
kubectl delete ns kyverno-policy-tests
kubectl delete ns mytestns
kubectl delete -f registry_secret_policy.yaml
kubectl delete -f mutation_policy.yaml
kubectl delete -f validation_policy.yaml
```

## Conclusion
Great job! You've implemented Kyverno's key policy types:

**Validation:** Ensuring cluster compliance
**Mutation:** Dynamically modifying resources
**Generation:** Automating resource creation

With these powerful tools at your disposal, you're well-equipped to maintain a robust, secure, and efficient cluster. 