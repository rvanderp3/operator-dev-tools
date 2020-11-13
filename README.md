## Prerequisites
- A sandbox OpenShift 4.x cluster
- cluster-admin privileges
- deployed internal image registry

## Deploying a test binary
1. Login to the registry

2. Identify the image that needs to be patched.  In this example, the `openshift-machine-api` operator `machine-operator` container image for `4.6.1`.

~~~
$ oc get deployment machine-api-operator -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{" "}{.image}{"\n"}{end}'
kube-rbac-proxy quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:c75977f28becdf4f7065cfa37233464dd31208b1767e620c4f19658f53f8ff8c
machine-api-operator quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:c51d44f380ecff7d36b1de4bb3bdbd4ac66abc6669724f28d81bd4af5741a8ac
~~~

3. Pull the image.  You will need your pull-secret for this.
~~~
$ podman pull --authfile ~/pull-secret.txt  quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:c51d44f380ecff7d36b1de4bb3bdbd4ac66abc6669724f28d81bd4af5741a8ac
Trying to pull quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:c51d44f380ecff7d36b1de4bb3bdbd4ac66abc6669724f28d81bd4af5741a8ac...
Getting image source signatures
Copying blob fec76f9f4e33 skipped: already exists  
Copying blob c4d668e229cd skipped: already exists  
Copying blob ec1681b6a383 skipped: already exists  
Copying blob c135b46b12e3 [--------------------------------------] 0.0b / 0.0b
Copying config 2129063b76 done  
Writing manifest to image destination
Storing signatures
2129063b76c3703ca046cf31a4b59b231dae9253971ad65e2517f8c78c16f86a    <------ save this ID
~~~

4. Create a `Dockerfile` which will allow us to overlay the executable we want to replace.  Also, be sure to copy the executable in to the same directory as the `Dockerfile`.  

~~~
FROM 2129063b76c3703ca046cf31a4b59b231dae9253971ad65e2517f8c78c16f86a     <---- ID from above
copy machine-api-operator .
~~~

5. Build the image

~~~
podman build . --tag my-machine-api-operator
~~~

6. Login to the OpenShift image registry.  You will need a non-`system:admin` user with push rights to the registry.
~~~
HOST=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')
podman login -u $(oc whoami) -p $(oc whoami -t) --tls-verify=false $HOST 
~~~

7. Tag the image in preparation for pushing to the registry.
~~~
podman tag my-machine-api-operator default-route-openshift-image-registry.apps-crc.testing/openshift-machine-api/my-machine-api-operator:latest
~~~

Note: The format of the tag must be `registry/namespace/image:tag`.  The namespace is critical as read/writes to the registry are limited to namespaces for which you have authorization.

8. Push the image:
~~~
podman push default-route-openshift-image-registry.apps-crc.testing/openshift-machine-api/my-machine-api-operator:latest --tls-verify=false
~~~

9. Prevent the CVO from managing the operator resources you are modifying.  In this example, a `Deployment` in the `openshift-machine-api` namespace.

~~~
cat <<EOF >version-patch-add-override.yaml
- op: add
  path: /spec/overrides
  value:
    - kind: Deployment
    group: apps/v1
    name: machine-api-operator
    namespace: openshift-machine-api
    unmanaged: true
EOF

oc patch clusterversion version --type json -p "$(cat version-patch-add-override.yaml)"
~~~

See [Setting Objects Unmanaged](https://github.com/openshift/cluster-version-operator/blob/master/docs/dev/clusterversion.md#setting-objects-unmanaged) for details.


6. Modify the `Deployment`, `ReplicaSet`, etc... to use the pushed image.

~~~
      - args:
        - start
        - --images-json=/etc/machine-api-operator-config/images/images.json
        - --alsologtostderr
        - --v=3
        command:
        - /machine-api-operator
        env:
        - name: RELEASE_VERSION
          value: 4.6.1
        - name: COMPONENT_NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        - name: METRICS_PORT
          value: "8080"
        image: image-registry.openshift-image-registry.svc:5000/openshift-machine-api/my-machine-api-operator:latest
        imagePullPolicy: IfNotPresent
        name: machine-api-operator
~~~

