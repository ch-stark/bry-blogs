# Tips for using templating in Governance Policies - Part 2

**Author:** Brian Jarvis

## Introduction

Red Hat Advanced Cluster Management for Kubernetes (RHACM) Governance provides an extensible framework for enterprises to introduce their own security and configuration policies that can be applied to managed OpenShift or Kubernetes clusters. For more information on RHACM policies, I recommend that you read the [Applying Policy-Based Governance at Scale Using Templates](https://cloud.redhat.com/blog/applying-policy-based-governance-at-scale-using-templates) and [Comply to standards using policy based governance](https://cloud.redhat.com/blog/comply-to-standards-using-policy-based-governance-of-red-hat-advanced-cluster-management-for-kubernetes) blogs.

In this multi part blog series I will showcase a number of techniques that can be applied when using templates in your RHACM Policies. In [part one](https://cloud.redhat.com/blog/tips-for-using-templating-in-governance-policies-part-1) I reviewed practices you can use to make your templates more readable and easier to maintain.

In part two of this series I will look at more advanced template functionality and extended use cases for using Policies to manage clusters.

**Prerequisites**:
  - [Review Governance Policy Templates and template functions](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.7/html-single/governance/index#support-templates-in-config-policies)

## Validate cluster state
RHACM Policies are typically viewed as the mechanism to apply day-2 configuration to a cluster.  This could be configuring authentication, creating infra nodes and configuring cluster workloads on them, installing operators along with a number of other day-2 configurations.  In part one of this series we looked at using templating to make these configurations more dynamic and the policies easier to maintain.

A policy to install an Operator using the Operator Lifecycle Manager (OLM) might consist of a `Namespace` definition, an `OperatorGroup` and a `Subscription`.  Applying these three objects will result in `OLM` installing the specified Operator.  Once those three objects exist the `Policy` will show as status `compliant`.  Compliance is only an indicator the objects have been created as specified and not that the operator has successfully installed and is running.

RHACM Policies have the capability to be in an "Inform" state where the Policy can be extended to validate the state of objects in the cluster.  This additional functionality opens a very powerful set of tools setting RHACM apart from other GitOps tooling to manage clusters.  As a cluster manager you can ensure all components are healthy across your entire fleet of clusters by viewing the status in RHACM.  

Let's review how we can implement this when installing an Operator like OpenShift GitOps.  In addition to the Policy to enforce creating the Subscription we can add a Policy to verify the health of the operator.  The example below will validate the health of the Subscription, the Operator Deployment, and the ArgoCD instance itself.
~~~
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: gitops-operator-health
  namespace: bry-tam-policies
spec:
  disabled: false
  policy-templates:
  - objectDefinition:
      apiVersion: policy.open-cluster-management.io/v1
      kind: ConfigurationPolicy
      metadata:
        name: gitops-operator-health
      spec:
        object-templates:
        - complianceType: musthave
          objectDefinition:
            apiVersion: operators.coreos.com/v1alpha1
            kind: Subscription
            metadata:
              labels:
                acm-policy: gitops-operator
              namespace: openshift-operators
            status:
              state: AtLatestKnown
        - complianceType: musthave
          objectDefinition:
            apiVersion: apps/v1
            kind: Deployment
            metadata:
              labels:
                olm.owner: '{{ (lookup "operators.coreos.com/v1alpha1" "Subscription" "openshift-operators" "openshift-gitops-operator").status.currentCSV }}'
              namespace: openshift-operators
            status:
              availableReplicas: 1
              conditions:
              - status: "True"
                type: Available
              readyReplicas: 1
              replicas: 1
              updatedReplicas: 1
        - complianceType: musthave
          objectDefinition:
            apiVersion: argoproj.io/v1alpha1
            kind: ArgoCD
            metadata:
              namespace: openshift-gitops
            status:
              applicationController: Running
              applicationSetController: Running
              dex: Running
              notificationsController: Running
              phase: Available
              redis: Running
              repo: Running
              server: Running
              ssoConfig: Success
        remediationAction: inform
        severity: high
~~~

We can now not only determine the objects required were created to install the operator, but the operator is installed and running successfully.  

RHACM inform policies can be used to identify other cluster issues, not just the health of our Day-2 configurations.  Common cluster health states such as [kcs-645901](https://access.redhat.com/solutions/6459071) can be identified in Policies to make cluster administrators aware of potential problems before users are impacted.  This example would become non-compliant if the openshift-marketplace Job or InstallPlan contain the indicated status conditions.  An upcoming addition to this series we will look at how we can use policies to automatically correct issues such as this.
~~~
---
kind: Job
apiVersion: batch/v1
metadata:
  namespace: openshift-marketplace
status:
  conditions:
    - type: Failed
      status: 'True'
      reason: DeadlineExceeded
      message: Job was active longer than specified deadline
  failed: 1

---
apiVersion: operators.coreos.com/v1alpha1
kind: InstallPlan
metadata:
  generateName: install-
status:
  bundleLookups:
    - conditions:
        - reason: JobIncomplete
          status: 'True'
          type: BundleLookupPending
        - message: Job was active longer than specified deadline
          reason: DeadlineExceeded
          status: 'True'
          type: BundleLookupFailed
  conditions:
    - message: >-
        bundle unpacking failed. Reason: DeadlineExceeded, and Message: Job was active longer than specified deadline
      reason: InstallCheckFailed
      status: 'False'
      type: Installed
  phase: Failed
~~~
Note for the above example to work you need to create a PolicyGenerator configuration.


## Enabling new capabilities with object-templates-raw
A new capability was added to `ConfigurationPolicies` in RHACM 2.7.2 and 2.8; [objects-template-raw](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.8/html-single/governance/index#raw-object-template-processing).  This new feature allows you to use `if statements`, assign values to variables, and make use of ranges.

All of the templating we have discussed to this point has been to return a string or a single value.  `object-templates-raw` supports advanced templating use-cases allowing a policy to generate YAML string representation.

In part-1 we looked at setting the default value for the number of replicas on the `IngressController` based on the number of infra nodes found.  But we did not configure the nodeSelector or tolerations to support running on the infra nodes.  Lets take a look at how using raw templates allows us to fully solve this.
~~~
apiVersion: policy.open-cluster-management.io/v1
kind: ConfigurationPolicy
metadata:
  name: ingressoperator-default
spec:
  remediationAction: enforce
  severity: low
  object-templates-raw: |
    - complianceType: musthave
      objectDefinition:
        apiVersion: operator.openshift.io/v1
        kind: IngressController
        metadata:
          name: default
          namespace: openshift-ingress-operator
        spec:
          httpEmptyRequestsPolicy: Respond
    {{- $infraCount := (len (lookup "v1" "Node" "" "" "node-role.kubernetes.io/infra").items) }}
    {{- if ne $infraCount 0 }}
          nodePlacement:
            nodeSelector: 
              matchLabels:
                node-role.kubernetes.io/infra: ""
            tolerations:
            - operator: Exists
              key: node-role.kubernetes.io/infra
    {{- end }}
          replicas: {{ $infraCount | default 2) | toInt }}
~~~

When the policy is applied to the cluster, if there are zero infra nodes (`$infraCount == 0`) the whole block for the `spec.nodePlacement` will not be part of the IngressController configuration. Once infra nodes are added to the cluster the policy will reevaluate and the configuration will be updated.

The raw templating also allows you to create more advanced objects where some processing of information needs to be completed before generating the `objectDefinition`.  Here we are creating the multiline string for the Thanos configuration using information from the OpenShift Data Foundation configured on the cluster.  The Thanos configuration is then processed and encoded to be stored in the thanos.yaml key of the secret generated by the policy.
~~~
apiVersion: policy.open-cluster-management.io/v1
kind: ConfigurationPolicy
metadata:
  name: thanos-secret
spec:
  remediationAction: enforce
  severity: high
  object-templates-raw: |
    {{- /* read the bucket data and noobaa endpoint access data */ -}}
    {{- $objBucket := (lookup "objectbucket.io/v1alpha1" "ObjectBucket" "" "obc-openshift-storage-obc-observability") }}
    {{- $awsAccess := (lookup "v1" "Secret" "openshift-storage" "noobaa-admin") }}
    {{- /* create the thanos config file as a template */ -}}
    {{- $thanosConfig := `
    type: s3
    config:
      bucket: %[1]s
      endpoint: %[2]s
      insecure: true
      access_key: %[3]s
      secret_key: %[4]s
    }}
    {{- /* create the secret using the thanos configuration template created above. */ -}}
    - complianceType: mustonlyhave
      objectDefinition:
        apiVersion: v1
        kind: Secret
        metadata:
          name: thanos-object-storage
          namespace: open-cluster-management-observability
        type: Opaque
        data:
          thanos.yaml: {{ (printf $thanosConfig $objBucket.spec.endpoint.bucketName 
                                                $objBucket.spec.endpoint.bucketHost 
                                                ($awsAccess.data.AWS_ACCESS_KEY_ID | base64dec) 
                                                ($awsAccess.data.AWS_SECRET_ACCESS_KEY | base64dec)
                          ) | base64enc }}
~~~

## Using range to generator objects in policies
The `range` [function](https://pkg.go.dev/text/template) creates a loop on an array, slice, map, or channel.  We can use this to loop through a list of either static values, a return from the lookup function, or parts of an object such as the labels on a Deployment.  Each iteration of the loop can be assigned to a variable using the format `{{ range $myItem := $list }} printf $myItem.property {{ end }}` a dot (.) context variable using the format `{{ range $list }} printf .property {{ end }}` or `{{ range $myItem := $list }} printf $myItem.property {{ else }} printf "empty list" {{ end }}` which will execute the else if the $list is empty.

This can be very useful for creating policies that would generate many objectDefinitions such as creating a `ConfigMap` for each namespace that meets a set requirement.  In this example we are looping through all `Pods` in the "portworx" namespace, identifying failed pods with the name containing "kvdb".  Pods found matching this condition will be removed from the cluster.
~~~
apiVersion: policy.open-cluster-management.io/v1
kind: ConfigurationPolicy
metadata:
  name: portworx-failed-pod-claner
spec:
  remediationAction: enforce
  severity: low
  object-templates-raw: |
    {{- /* find Portworx pods in terminated state */ -}}
    {{- range $pp := (lookup "v1" "Pod" "portworx" "").items }}
      {{- /* if the pod is blocked because it is in node shutdown we should delete the pod */ -}}
      {{- if and (eq $pp.status.phase "Failed") 
                 (contains "kvdb" $pp.metadata.name) }}
    - complianceType: mustnothave
      objectDefinition:
        apiVersion: v1
        kind: Pod
        metadata:
          name: {{ $pp.metadata.name }}
          namespace: {{ $pp.metadata.namespace }}
      {{- end }}
    {{- end }}
~~~

Expanding our earlier example checking the health of OpenShift GitOps instances we can make use of range to check all `ArgoCD` instances on a cluster, along with a range of label selectors to validate each `Deployment` ensuring all components are healthy and contain the expected number of replicas.
~~~
apiVersion: policy.open-cluster-management.io/v1
kind: ConfigurationPolicy
metadata:
  name: argocd-instance-status
spec:
  remediationAction: inform
  severity: high
  object-templates-raw: |
    ## Get all the ArgoCD instances we are checking health for  
    {{- range $argo := (lookup "argoproj.io/v1alpha1" "ArgoCD" "" "").items }}
      ## list all of the lookups for Argo deployments  
      {{- $selectors := list "app.kubernetes.io/name=argocd-applicationset-controller"
                            (printf "app.kubernetes.io/name=%s-dex-server" $argo.metadata.name)
                            (printf "app.kubernetes.io/name=%s-notifications-controller" $argo.metadata.name)
                            (printf "app.kubernetes.io/name=%s-redis" $argo.metadata.name)
                            (printf "app.kubernetes.io/name=%s-repo-server" $argo.metadata.name)
                            (printf "app.kubernetes.io/name=%s-server" $argo.metadata.name)
      }}
      
      ## ensure ArgoCD is reporting healthy 
    - complianceType: musthave
      objectDefinition:
        apiVersion: argoproj.io/v1alpha1
        kind: ArgoCD
        metadata:
          namespace: {{ $argo.metadata.namespace }}
        status:
          server: Running
          notificationsController: Running
          applicationController: Running
          applicationSetController: Running
          ssoConfig: Success
          repo: Running
          dex: Running
          phase: Available
          redis: Running

      ## ensure all deployments are healthy in each argo instance 
      {{- range $sel := $selectors }}
        {{- $dep := (lookup "apps/v1" "Deployment" $argo.metadata.namespace "" $sel).items }}
    - complianceType: musthave
      objectDefinition:
        kind: Deployment
        apiVersion: apps/v1
        metadata:
          namespace: {{ $argo.metadata.namespace }}
          labels:
            {{ $sel | replace "=" ": " }}
        status:
        {{- if gt (len $dep) 0 }}
          {{- $dp := (index $dep 0) }}
          replicas: {{ $dp.spec.replicas }}
          updatedReplicas: {{ $dp.spec.replicas }}
          readyReplicas: {{ $dp.spec.replicas }}
          availableReplicas: {{ $dp.spec.replicas }}
          conditions:
            - type: Available
              status: 'True'
        {{- end }}
      {{- end }}
    {{- end }}
~~~

## Summary
In part one of this series I outlined the usage of a number of template functions and examples you can use to make your templates easier to read and maintain. In part two we looked at validating cluster health with policies and how to use the `object-templates-raw` to expand templates for more complex use cases.
