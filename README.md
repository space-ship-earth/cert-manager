# Cert Manager

This repo has two parts:
1. A tutorial of how to set up `cert-manager` on a k8s cluster
2. A repository of the certificates we've created

If you just want to get certificates, just look at the first section.
If you're an administrator who is managing clusters, the second section might be of use if you want `cert-manager` running there.

## Managing certificates

This section assumes that `cert-manager` is already running on the k8s cluster.
If it is not yet running, see section below, "Installation of cert-manager", to set it up.

To create your certificate, follow these steps:

1. copy the template to your certificate name. you will need to replace instances of `subdomain` and `my-fqdn` with real values (e.g., `my-fqdn` should be `igor.spaceshipearth.org`)
    ```
    $ cd certs
    $ cp template.yaml igor.spaceshipearth.org.yaml
    $ vim igor.spaceshipearth.org.yaml
    ```

1. create the certificate:
   ```
   $ kubectl apply -f igor.spaceshipearth.org.yaml
   ```

3. check if the cert is valid. it should take a few minutes for the cert to become `Status: True`:
   ```
   $ kubectl get certificate my-certificate --output=yaml
   ```
   if you're debugging why it's not yet ready, you can run:
   ```
   $ kubectl describe certificate my-certificate
   ```
   and look for the `Events` to show you what's happening

4. Your certificate should be available in the secret named like the field `secret_name`. check for it:
   ```
   $ kubectl describe secret my-fqdn-tls
   ....
   Data
   ====
   ca.crt:   0 bytes
   tls.crt:  3586 bytes
   tls.key:  1675 bytes
   ```

5. Add your new `.yaml` file to the repo and push it, so we have a record of certs created, and by whom.

That's it!
You're done!
Go use your new cert on your awesome new web service.


## Installation of cert-manager

This section shows you how to set up `cert-manager` on your cluster.
We basically follow [their tutorial](https://docs.cert-manager.io/en/latest/getting-started/install.html).
Their documentation for ACME DNS issuer was pretty poor, but thankfully [this tutorial](https://github.com/knative/docs/blob/master/serving/using-cert-manager-on-gcp.md) existed, so we followed it pretty closely.

We opted to include `yaml` files necessary for setup in this repo to keep things consistent.
If you wish to upgrade cert-manager, you may wish to include updated `.yaml` files from [the upstream project](https://github.com/jetstack/cert-manager) here and then re-apply the resources to dependent clusters.

#### Set up `cert-manager`

1. Select your cluster with `devenv kubectl` and `devenv kubectl.set`
1. Create cert-manager namespace:
   ```
   $ kubectl create namespace cert-manager
   ```
1. Disable resource validation on the cert-manager namespace:
   ```
   $ kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true
   ```
1. Elevate your own permissions:
   ```
   $ kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value core/account)
   ```
1. Install the CustomResourceDefinition resources
   ```
   $ kubectl apply -f 00-crds.yaml
   ````
1. Install cert-manager itself
   ```
   $ kubectl apply --validate=false -f cert-manager.yaml
   ```
1. Validate the install:
   ```
   $ kubectl get pods --namespace cert-manager
   NAME                                    READY     STATUS      RESTARTS   AGE
   cert-manager-58fbf74b8-pn4zj            1/1       Running     0          1m
   cert-manager-webhook-84cfc4d76f-chqf9   1/1       Running     0          1m
   cert-manager-webhook-ca-sync-s2f8t      0/1       Completed   5          7m
   ```
1. Test that you can create certs:
   ```
   $ kubectl apply -f test-resources.yaml
   $ kubectl describe certificate -n cert-manager-test
    ...
    ...
    Normal  CertIssued  32s   cert-manager  Certificate issued successfully
   $ kubectl delete -f test-resources.yaml
   ```

## Set up `cloud-dns` Issuer

You will create a service account which can modify DNS entries in this project.
Then, you'll create an `Issuer` which uses those creds to modify DNS for ACME validation.

1. check that you've got the right configuration active in gcloud.
   replace `<name>` with your project name
   ```
   $ gcloud config configurations list
   $ gcloud config configurations activate <name>
   ```

1. create the service account and download it's credentials into `key.json`:
   ```
   $ export PROJECT_NAME=`gcloud config list --format 'value(core.project)'`
   $ export SVC_NAME="cert-manager-cloud-dns-admin"
   $ gcloud iam service-accounts create $SVC_NAME --display-name "service account to support ACME DNS-01 challenge"
   $ export FULL_SVC_NAME="${SVC_NAME}@${PROJECT_NAME}.iam.gserviceaccount.com"
   $ gcloud projects add-iam-policy-binding $PROJECT_NAME --member serviceAccount:$FULL_SVC_NAME --role roles/dns.admin
   $ gcloud iam service-accounts keys create key.json --iam-account=$FULL_SVC_NAME
   ```

1. save the key as a kubernetes secret under the cert-manager namespace
   ```
   $ kubectl create secret --namespace cert-manager generic cloud-dns-creds --from-file=key.json=key.json
   $ rm key.json
   ```

1. create the issuer. you'll need to replace the string `REPLACE_WITH_PROJECT_ID` with your `$PROJECT_NAME`:
   ```
   $ sed "s/REPLACE_WITH_PROJECT_NAME/$PROJECT_NAME/g" issuer.yaml.template > issuer.yaml
   $ kubectl apply -f issuer.yaml
   ```


