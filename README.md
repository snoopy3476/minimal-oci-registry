# Minimal Private OCI Image Registry

A very simple registry yaml representing K8s objects, with additional following features:

- Multi-user auth support: read-only users, administrators
- Image deletion support & automatic garbage collection on image startup

Useful if you don't want to use external projects to manage your private registry, but supports minimal features for sharing with others.


## Abstract Structure
![Abstract Structure](/readme-asset/mreg.svg)


## Usage


### Getting Started

  - `podman`
    ```shell
    podman kube play mreg.yml
    ```
  - `k8s`
    ```shell
    kubectl apply -f mreg.yml
    ```


### Manage Users
By default, there is no user exist.  
You should add users (and toggle some of them to admin) to use registry.  

  - Basic usages

    - podman
      ```shell
      podman exec -it mreg-pod-auth manage-user [args...]
      ```
    - k8s
      ```shell
      kubectl exec -it deploy/mreg -n mreg -- manage-user [args...]
      ```

  - Argument details  
    (Run commands inside a *AUTH* container, as 'Basic usages' above)

    - List users
      ```shell
      manage-user ls
      ```
    - Add a user
      ```shell
      manage-user add <user-name>
      ```
    - Toggle a user between user <=> admin
      ```shell
      manage-user toggle <user-name>
      ```
    - Delete a user
      ```shell
      manage-user rm <user-name>
      ```

  - Remarks

    - Normal users: `GET` `HEAD` are allowed
    - Administrators: `GET` `HEAD` `POST` `PUT` `DELETE` `PATCH` are allowed


### Manage TLS
When cert/key files are added, the pod will serve as https.

  - Basic usages

    - podman
      ```shell
      podman exec -it mreg-pod-auth manage-tls [arg]
      ```
    - k8s
      ```shell
      kubectl exec -it deploy/mreg -n mreg -- manage-tls [arg]
      ```

  - Argument details  
    (Run commands inside a *AUTH* container, as 'Basic usages' above)

    - Print current TLS info
      ```shell
      manage-tls print
      ```
    - Write or delete cert file (tls.crt)
      ```shell
      manage-tls cert
      ```
    - Write or delete key file (tls.key)
      ```shell
      manage-tls key
      ```

  - Remarks

    - You can create your own cert/key files yourself, or with the script [`script/mreg-gen-tls`](script/mreg-gen-tls).
      ```shell
      script/mreg-gen-tls <address-to-access-registry>
      ```

    - It is possible to redirect STDIN when writing cert/key files:

      - podman
        ```shell
        podman exec -i mreg-pod-auth manage-tls cert < your-tls-file.crt
        podman exec -i mreg-pod-auth manage-tls key < your-tls-file.key
        ```

      - k8s
        ```shell
        kubectl exec -i deploy/mreg -n mreg -- manage-tls cert < your-tls-file.crt
        kubectl exec -i deploy/mreg -n mreg -- manage-tls key < your-tls-file.key
        ```




### Untag Image Manifest Remotely

* Important Notes
  * Guide below deletes not only specified tag, but also all img:tags referencing the same manifest!
    * (e.g. If you push the same image with both `a:latest` & `a:v1.1`, then delete `a:v1.1`, then `a:latest` is also removed)
  * If you want to delete only the specified tag, push a dummy image to the target image:tag, then delete according to the following.
  * Or you might use external utility such as [`regctl`](https://github.com/regclient/regclient), etc.

(Check [`script/mreg-manage`](script/mreg-manage) for one-shot untag)

  1. Get a digest of target image tag manifest
     ```shell
     # first set required variables for commands below:
     # ADMIN_ID="<id>" REG_ADDR="<registry-addr-port>" IMG_NAME="<img>" IMG_TAG="<tag>"

     DIGEST="$(curl -u "${ADMIN_ID:?}" \
       -H "Accept: application/vnd.oci.image.manifest.v1+json" \
       -I "${REG_ADDR:?}"/v2/"${IMG_NAME:?}"/manifests/"${IMG_TAG:?}" \
       | grep "^Docker-Content-Digest" \
     )"
     ```

  2. Delete manifest (untag)  
     Warning: This DELETE request will untag & delete ALL TAGS REFERENCING THE SAME TARGET DIGEST!  
     ```shell
     curl -u "${ADMIN_ID:?}" -X DELETE \
       "${REG_ADDR:?}"/v2/"${IMG_NAME:?}"/manifests/"${DIGEST:?}"
     ```

### Automatic Image Garbage Collection
(Delete unused blob which is untagged above)

  - When `minimal-oci-registry` image starts, unused files are removed before registry starts up.
  - Consider scheduling image restart periodically (every 5 AM, etc.) to garbage collect storage!



## Author
Kim Hwiwon \<kim.hwiwon@outlook.com\>
