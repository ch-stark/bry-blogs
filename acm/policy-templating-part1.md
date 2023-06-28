# Tips for using templating in Governance Policies - Part 1

**Author:** Brian Jarvis

## Introduction

Red Hat Advanced Cluster Management for Kubernetes (RHACM) Governance provides an extensible framework for enterprises to introduce their own security and configuration policies that can be applied to managed OpenShift or Kubernetes clusters. For more information on RHACM policies, I recommend that you read the [Comply to standards using policy based governance](https://cloud.redhat.com/blog/comply-to-standards-using-policy-based-governance-of-red-hat-advanced-cluster-management-for-kubernetes), and [Implement Policy-based Governance Using Configuration Management](https://cloud.redhat.com/blog/implement-policy-based-governance-using-configuration-management-of-red-hat-advanced-cluster-management-for-kubernetes) blogs.

In part one of this blog series I will review practices you can use to make using templates in your policies more readable and easier to maintain.

**Prerequisites**:
  - [Review Governance Policy Templates and template functions](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.7/html-single/governance/index#support-templates-in-config-policies)


## Use PolicyGenerator
If you are not familiar with [PolicyGenerator](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.7/html-single/governance/index#policy-generator) I recommend you read the [Generating Governance Policies Using Kustomize and GitOps](https://cloud.redhat.com/blog/generating-governance-policies-using-kustomize-and-gitops) blog post.

The PolicyGenerator greatly simplifies management of your policies by allowing you to focus on the configuration to be applied.  The generator configuration lets you control the specifics of the Policy that is generated.

Here you can see the difference in required code to create a simple namespace.  
![Namespace Policy](images/part1-policygenerator-0.png)

## "eq" function instead of || to compare 1:n
In the case where we want to compare and test an object to see if it equals one out of multiple values we would normally write our if statment as follows:
~~~
$myObject == Arg1 || $myObject == Arg2  || $myObject == Arg3
~~~

But in Go the `eq` is a [function](https://pkg.go.dev/text/template#hdr-Functions).  This allows us the power to evaluate all arguments against the object.  The above could be written as this instead
~~~
eq $myObject Arg1 Arg2 Arg3
~~~

Let's look at a full example.  Suppose we wanted our clusters in production and staging to use a different number replicas for the IngressController compared to other environments.  We would write a policy similar to below
~~~
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: default
  namespace: openshift-ingress-operator
spec:
  httpEmptyRequestsPolicy: Respond
  replicas: '{{ if eq (fromClusterClaim "env") "production" "staging" -}} 6 {{- else -}} 2 {{- end }}'
~~~

## printf to format and concatinate strings
The `printf()` [function](https://pkg.go.dev/fmt) takes a templated string which contains the text to be formatted plus some annotated verbs that tell the function how to format the remaining arguments.

Instead of using multiple template blocks like
~~~
{{ fromClusterClaim "env" }}-mycluster-{{ fromClusterClaim "name" }}
~~~

We can use the `printf()` function to rewrite it as
~~~
{{ printf("%s-mycluster-%s", (fromClusterClaim "env"), (fromClusterClaim "name")) }}
~~~

`printf()` also allows formatting of the data using a wide array of verbs.  Some of the more common usages are
Some of the most commonly used specifiers are:
- %v – formats the value in a default format
- %d – formats decimal integers
- %g – formats the floating-point numbers
- %b – formats base 22 numbers
- %o – formats base 88 numbers
- %t – formats true or false values
- %s – formats string values

You can also explicitly specify which argument index to use with the verb.  This allows you to control where the argument is used, instead of the default successive behavior, and also use an argument multiple times.  The templated text is considered zero(0) in the indexed array so arguments will start with a one(1).
~~~
{{ printf("%[1]s-mycluster-%[1]s", (fromClusterClaim "env")) }}
~~~

## use default

## 


Part 1: Basic tips to make better/more readable policies
  - Use of eq
  - use of printf
  - use of default
  - use of .ManagedClusterName 
 (edit) - use of .managedClusterLabels
 (edit) - use of copySecretData


