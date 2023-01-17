# K8s The Fast Way 🏎️

If you are comfortable with kubernetes, maybe even consider yourself good at it, then your biggest enemy on the CKA exam is time. Practically, you have fewer then 10 minutes per question. For questions that only ask you to create some pods, roles, secrets etc. - you need to be able to sail through, even when there are a lot of sub-tasks. A deployment with a serviceaccount, role, custom command, volume, and antiaffinity policy should be trivial. This guide will help you memorize the most common commands and template locations in the deocumentation.

## Imperative

The following resources can be created imperatively. The notes below highlight the most important flags.

For key-value type things (like labels, annotation etc.) pass them as a list of params. For example, `kubectl run my-webserver --image=nginx --labels="env=prod,stack=frontend"`

Limited functionality is exposed via imperative commands, which is a blessing and a curse. The bad news is that you'll never know ahead of time when a flag you think should exist is actually unsupported. The good news is that there are fewer commands to keep in mind.

The name of the game is doing as much as you can imperatively. If that doesn't suffice, write the output of the imperative command to yaml and then edit it by hand before applying it.

Tip: remember that most of these command can take the `-n <namespace>` arg as well to specify which namespace the resource should be created in. Here it is omitted to avouid repetition.

#### Pod

`kubectl run <name>`
- `--image`
- `--labels=[]`
- `--annotations=[]`
- `--env=[]`
- `--port`
- `--privileged`
- `-- [COMMAND] [args]` (at the end of the imperative command)

Opening ports on a container has sort of a funny syntax declaritively, so it is best to use the `--port` option here to do the dirty work for you.

#### Deployment

`kubectl create deployment <name>`
- `--image`
- `--replicas`
- `--port`
- `-- [COMMAND] [args]` (at the end of the imperative command)

Like pods, opening ports on containers has sort of a funny syntax declaritively, so it is best to use the `--port` option here to do the dirty work for you.

If you need to create multiple containers in a deployment, just create one container with as much config as you can first. Then write the yaml to a file and hand edit it there.

The other cool thing you can do with deployments is `kubectl scale deployment <name>`, and then pass the arg `--replicas`.

#### Job
`kubectl create job <name>`
- `--image`
- `--from`

#### Cronjob
`kubectl create cronjob <name>`
- `--image`
- `--schedule`

#### Service
You can create a service explicitly with `kubectl create service <type> <args>`. But in the exam you don't want to commit this much mental energy to the task. Luckily, we have `kubectl expose <resource>` to sort all the particulars for us.

`kubectl expose <resource> <name>`
- `--name`
- `--type`
- `--port`
- `--target-port`

#### Servicaccount
`kubectl create serviceaccount <name>`

Tip: besides perhaps a namespace, this one never needs additional args.

#### (Cluster)Role(Binding)

`kubectl create (cluster)role <name>`
- `--verb=[]`
- `--resource=[]`

`kubectl create (cluster)rolebinding <name>`
- `--role` or `--clusterrole`
- `--group` or `--serviceaccount`

In my humble opinion, it is in fact easier to create roles and rolebindings imperatively. Declaritively, you have to fumble around to find the names of the API groups. The imperative command connects all the dots for you. Nothing to it.

The only gotchya is that the value of `--serviceaccount` must be in the form of `<namespace>:<name>`. Don't fret, if you omit the namespace part `kubectl` will yell at you and tell you to add it.

#### Secret
`kubectl create secret generic <name>`
- `--from-literal="foo=bar="`
- `--from-file=/path/to/file`

Note the you always want to create a "generic" secret. There are other types but I don't think they come up on the exam.

Further note that the args for this imperative command are **not lists**! You have to specify multiple `--from-literal` args to create a secret with multiple keys in it. If you do something like `--from-literal="foo=bar,baz=buz`, then you will just end up with a single key `foo` containing the base64 value of `bar,baz=buz` which is probably not what you meant.

#### ConfigMap
`kubectl create cm <name>
- `--from-literal="foo=bar="`
- `--from-file=/path/to/file`
- `--from-env-file=/path/to/file`

The `--from-literal` arg is the same deal as secrets; it must be a single value - not a list.

#### PriorityClass
`kubectl create priorityclass <name>`
- `--value`

## CTRL-F

The syntaxes for the following resources (and configurations) can only be found through examples in the K8s documentation. The table below indicates the `resource`, `page` and `ctrl-f term` for each resource you may want to make.

Resource | Page | Term
---|---|---
PersistentVolume | Persistent Volumes | pv0003
PersistentVolumeClaim | Persistent Volumes | myclaim
Pod with PVC | Persistent Volumes | mypod
Pod with Ephemeral Volume | Volumes | cache-volume
Ingress | Ingress | minimal ingress
Daemonset | Daemonset | fluentd-elasticsearch
Secret as env | Secrets | secret-env-pod
Secret as volume | Secrets | mypod
ConfigMap as env and volume | ConfigMaps | configmap-demo-pod
Pod with toleration | Taints and Tolerations | nginx
nodeSelector | Assign Pods to Nodes | disktype: ssd
(Anti)affinity | Assign*ing* Pods to Nodes | web-store
Liveness and Readiness with command | Configure Liveness, Readiness and Startup Probes | liveness-exec
Liveness and Readiness with http | Configure Liveness, Readiness and Startup Probes | liveness-http
Pod metadata | Expose Pod Information to Containers Through Environment Variables | MY_NODE_NAME
Network Policy | Network Policies | test-network-policy
