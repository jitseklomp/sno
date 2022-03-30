# Single-Node OpenShift (SNO) demo 

## Prepare workstation

Download latest Helm and OpenShift Client binaries from https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ and make sure they are added to your `$PATH`

## Prepare OpenShift

> Make sure you are logged in to OpenShift with `cluster-admin` privileges and have access to the SSH key used to install the cluster.

### Configure node

Download `openshift-config` from [GitHub](https://github.com/jitseklomp/openshift-config/releases/) and configure persistent storage:

```bash
./config-snc.sh persistent-storage -h <hostname>
```

Configure integrated container registry (optional):
```bash
./config-snc.sh registry -h <hostname>
```

### Configure OpenShift

Create new projects to deploy our demo application to:

```bash
oc new-project demo-test --display-name '[Test] Demo application'
oc new-project demo-prod --display-name '[Prod] Demo application'
```

## Installing GitLab

Now all prerequisites are in place we can deploy GitLab in to a new project:

```bash
# First create a new project
oc new-project gitlab
# GitLab needs access to `nonroot` SCC
oc adm policy add-scc-to-group nonroot system:serviceaccounts:gitlab

# Add official GitLab helm repo
helm repo add gitlab https://charts.gitlab.io/ && helm repo update

# Install GitLab using OpenShift default Ingress Controller, skip installation of runner
helm upgrade --install gitlab gitlab/gitlab \
   --timeout 600s \
   --set global.hosts.domain=apps.<CLUSTER_NAME>.<BASE_DOMAIN> \
   --set global.ingress.configureCertmanager=false \
   --set global.ingress.class=openshift-default \
   --set global.kas.enabled=false \
   --set certmanager.install=false \
   --set nginx-ingress.enabled=false \
   --set gitlab-runner.install=false
```

### Configure GitLab
Retrieve the default GitLab root password from the OpenShift web console or using the cli:

```bash
oc get secret -n gitlab gitlab-gitlab-initial-root-password -o jsonpath='{.data.password}' base64 -d > gitlab-root-pw.txt
```

Log in to the GitLab web interface using `root` and the saved password. Go to "Settings" and search for "Approval". Disable "Require admin approval for new sign-ups" and save. Return to "Overview" and click "New group". Create a new "Internal" group called "Demo"

### Add runners

Navigate to the Admin section of GitLab and go to "Overview" -> "Runners".  Copy the registration token using the "Register an instance runner" button. Make sure to reset the token first.

Add the token to the following template and save as `runner-values.yaml`:

```yaml
############################################
### GitLab Runner configuration template ###
############################################

## Environment-specific configuration

# Token
runnerRegistrationToken: <REPLACE_ME>
# Number of runners
replicas: 1
# Internal path to GitLab server instance
gitlabUrl: "http://gitlab-webservice-default.gitlab.svc.cluster.local:8080"

## Default settings
concurrent: 10
checkInterval: 30
sessionServer:
  enabled: false
rbac:
  create: false
  clusterWideAccess: false
  serviceAccountName: gitlab-runner-sa 
  podSecurityPolicy:
    enabled: false
    resourceNames:
    - gitlab-runner
metrics:
  enabled: true
  portName: metrics
  port: 9252
  serviceMonitor:
    enabled: true 
service:
  enabled: true
  type: ClusterIP
runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        namespace = "{{.Release.Namespace}}"
securityContext:
  runAsUser: 100
  fsGroup: 65533
  supplementalGroups: [0]
```

Install runners using Helm:

```bash
# Create a new project
oc new-project gitlab-runner
oc adm policy add-scc-to-group nonroot system:serviceaccounts:gitlab-runner

# Create a ServiceAccount with access to demo projects
oc -n gitlab-runner create serviceaccount gitlab-runner-sa
oc policy add-role-to-user admin system:serviceaccount:gitlab-runner:gitlab-runner-sa -n demo-test
oc policy add-role-to-user admin system:serviceaccount:gitlab-runner:gitlab-runner-sa -n demo-prod

# Install using stored runner-values.yaml
helm install --namespace gitlab-runner runner -f runner-values.yaml gitlab/gitlab-runner

# Due to a bug we need to patch the runner Deployment to use the correct entrypoint:
oc patch deployment -n gitlab-runner runner-gitlab-runner --patch \
   '{"spec":{"template":{"spec":{"containers":[{"name":"runner-gitlab-runner","command":["tini","--","/bin/bash","/configmaps/entrypoint"]}]}}}}'
```

Refresh the GitLab admin web console to see the new runner(s). 

Run `generate-kubeconfig.sh` to create `runner-kubeconfig`.

