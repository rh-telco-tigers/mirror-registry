# Mirroring Openshift 4 Repositories for disconnected install (air gapped)

Mirroring OpenShift images from the internet to a local disconnected environment is an effective method to expedite the installation process in isolated networks. By obtaining the necessary images from the internet and transferring them to the local environment, the installation can be carried out without relying on an internet connection.

This approach offers several advantages. It allows for a faster and more efficient installation process, as the images are readily available in the local environment. This eliminates the need to download the images from the internet during installation, which can be time-consuming and prone to interruptions in disconnected networks.

**Steps to follow**

1. Download the mirror-registry.tar.gz package for the latest version of the mirror registry for Red Hat OpenShift found on the OpenShift console Downloads page under Openshift disconnected installation tools

2. Install the mirror registry for Red Hat OpenShift on your local host with your current user account by using the mirror-registry tool

```
./mirror-registry install --quayRoot /mirror-registry/quay-config --quayStorage /mirror-registry/storage --pgStorage /mirror-registry/pg-storage --initUser admin --initPassword "Password"
```

3. Install the oc-mirror plugin from OpenShift console Downloads page under Openshift disconnected installation tools
   ```
    a. tar xvzf oc-mirror.tar.gz
    b. chmod +x oc-mirror
    c. sudo mv oc-mirror /usr/local/bin/.
   ```

5. Add mirror-registry credentials to pull-secret
    a. Download your "pull secret" from the Red Hat OpenShift Cluster Manager at 

       https://console.redhat.com/openshift/install/pull-secret.
   
    b. Execute the following commands
        ```
        jq . pull-secret.txt
        mkdir -p $HOME/.docker
        mv -v pull-secret.txt $HOME/.docker/config.json
        podman login -u admin -p password --authfile $HOME/.docker/config.json $(hostname -f):8443
        ```
      Confirm contents and create a backup
        ```
        jq . $HOME/.docker/config.json
        cp -v $HOME/.docker/config.json ~/pull-secret.json
        ```

**Create the imageset-config.yaml**

1. To create a default imageset-config.yaml run the following command
    a. oc-mirror init --registry <mirror-registry.hostname>:8443/ocp| tee imageset-config.yaml
    ```
    kind: ImageSetConfiguration
    apiVersion: mirror.openshift.io/v1alpha2
    storageConfig:
      registry:
        imageURL: mirror-registry.hostname:8443/ocp
        skipTLS: false
    mirror:
      platform:
        channels:
        - name: stable-4.16
          type: ocp
      operators:
      - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.18
        packages:
        - name: serverless-operator
          channels:
          - name: stable
      additionalImages:
      - name: registry.redhat.io/ubi8/ubi:latest
      helm: {}
    ```

**Some examples for imageset-config.yaml**

1. imageset config with few sets of operators and two versions of ocp
https://raw.githubusercontent.com/rh-telco-tigers/mirror-registry/refs/heads/main/imageset-config-with-operator-helm.yaml

2. imageset config with all operators
https://raw.githubusercontent.com/rh-telco-tigers/mirror-registry/refs/heads/main/imageset-config-allops.yaml

3. imageset config with just one operator
https://raw.githubusercontent.com/rh-telco-tigers/mirror-registry/refs/heads/main/imageset-config-gitops-operator.yaml

**Run the oc-mirror command**

Once you have created your imageset-config file now it is time to start the mirroring process. Run the following command to start the process
```
oc-mirror --config imageset-config.yaml docker://mirror-registry.hostname:8443/ocp
```
You should see successful output after oc-mirror runs, if you do not see the "Writing manifests" output then something may have gone wrong with the oc-mirror command
```
...
Writing image mapping to oc-mirror-workspace/results-1725721677/mapping.txt
Writing CatalogSource manifests to oc-mirror-workspace/results-1725721677
Writing ICSP manifests to oc-mirror-workspace/results-1725721677
```
