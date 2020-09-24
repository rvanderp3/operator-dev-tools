Patching code in to operators for quick unit testing presents some challenges.  Particularly if you just want to quickly patch an executable.  The steps below descirbe a relatively simple process for bootstrapping an alternate executable in to an operator fo unit testing.  Note, the purpose of these instructions is to provide a simple unit test platform.  It is up to you to adhere to the best practices of your environment.

## Prerequisites
- A sandbox OpenShift 4.x cluster
- cluster-admin privileges

## Deploying a test binary
1. Deploy an HTTP server in to the cluster.  In this example, I'm using the project name `bastion`.
~~~
oc new project bastion
oc new-app danjellz/http-server
~~~

Clearly, you can deploy any HTTP server you are comfortable with here.  Note that we will *not* expose the service as a route.  

2. For this specific web server, the pod needs to run in a privileged context.  

Add the `privileged` scc to the `default` service acount in the `bastion` project.
~~~
oc adm policy add-scc-to-user privileged -z default
~~~

Modify the `deployment` to add `privileged: true` to the container `securityContext`

3. Prevent the CVO from managing the operator resources you are modifying.  In this example, a `Deployment` in the `openshift-machine-api` namespace.

~~~
cat <<EOF >version-patch-add-override.yaml
- op: add
  path: /spec/overrides/-
  value:
    - kind: Deployment
    group: apps/v1
    name: machine-api-operator
    namespace: openshift-machine-api
    unmanaged: true
EOF

oc patch clusterversion version --type json -p "$(cat version-patch-add-override.yaml)"
~~~

4. Build your code!

5. Upload the binary to the HTTP server pod
~~~
oc cp ./machine-api-operator http-server-7b78d6d5cc-dh5gh:.
~~~

6. Modify the `Deployment`, `ReplicaSet`, etc... to pull in the built binary.  

~~~
          command:
            - /bin/sh
            - "-c"
          args:
            - "/usr/bin/curl -o /tmp/test http-server.bastion.svc.cluster.local:8080/machine-api-operator;chmod +x /tmp/test;/tmp/test start --images-json=/etc/machine-api-operator-config/images/images.json --alsologtostderr --v=3"
~~~

This bootstraps binary in to the desired image by:

1. Pulling the uploaded binary from the HTTP server `http-server.bastion.svc.cluster.local:8080`
2. Making that binary executable
3. Starting the uploaded binary while preserving the original flags.
