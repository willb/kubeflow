# Setting Up IAP on GKE

These instructions walk you through using [Identity Aware Proxy](https://cloud.google.com/iap/docs/)(IAP) to securely connect to Kubeflow
when using GKE.

  * IAP allows you to control access to JupyterHub using Google logins
  * Using IAP secures access to JupyterHub using HTTPs
  * IAP allows users to safely and easily connect to JupyterHub
  * You will need your own domain which can be purchased and managed using Google domains or another provider

If you aren't familiar with IAP you might want to start by looking at those docs.

### Preliminaries

##### Create an external static IP address

```
gcloud compute --project=${PROJECT} addresses create kubeflow --global
```

Use your DNS provider (e.g. Google Domains) create a type A custom resource record that associates the host you want e.g "kubeflow"
with the IP address that you just reserved.
  * Instructions for [Google Domains](https://support.google.com/domains/answer/3290350?hl=en&_ga=2.237821440.1874220825.1516857441-1976053267.1499435562&_gac=1.82147044.1516857441.Cj0KCQiA-qDTBRD-ARIsAJ_10yKS7G1HPa1aoM8Mk_4VagV9wIi5uKkMp5UWJGDNejKxWPKUO_A6ri4aAsahEALw_wcB)

#### Create oauth client credentials

Create an OAuth Client ID to be used to identify IAP when requesting acces to user's email to verify their identity.

1. Set up your OAuth consent screen:
  * Configure the [consent screen](https://console.cloud.google.com/apis/credentials/consent).
  * Under Email address, select the address that you want to display as a public contact. You must use either your email address or a Google Group that you own.
  * In the Product name box, enter a suitable like save `kubeflow`
  * Click Save.
2.  On the [Credentials](https://console.cloud.google.com/apis/credentials) Click Create credentials, and then click OAuth client ID.
  * Under Application type, select Web application. In the Name box enter a name, and in the Authorized redirect URIs box, enter

```
  https://${FQDN}/_gcp_gatekeeper/authenticate,
```
3. After you enter the details, click Create. Make note of the **client ID** and **client secret** that appear in the OAuth client window because we will
   need them later to enable IAP.

4. Create a new Kubernetes Secret with the the OAuth client ID and secret:

```
kubectl -n ${NAMESPACE} create secret generic kubeflow-oauth --from-literal=CLIENT_ID=${CLIENT_ID} --from-literal=CLIENT_SECRET=${CLIENT_SECRET}
```

### Setup Ingress

If you haven't already, follow the instructions in the [user_guide](https://github.com/kubeflow/kubeflow/blob/master/user_guide.md#deploy-kubeflow)
to create a ksonnet app to deploy Kubeflow.

Your GKE cluster permissions should include the `https://www.googleapis.com/auth/compute` scope and a service account with the `editor` or `compute.admin` IAM role. This is the default configuration for GKE clusters version 1.9.x and earlier.
Follow [this guide](https://medium.com/google-cloud/updating-google-container-engine-vm-scopes-with-zero-downtime-50bff87e5f80) if you need to modify the scopes of your cluster and the [IAM docs](https://cloud.google.com/iam/docs/granting-changing-revoking-access) for details on how to apply roles.

[cert-manager](https://github.com/jetstack/cert-manager) is used to automatically request valid SSL certifiactes using the [ACME](https://en.wikipedia.org/wiki/Automated_Certificate_Management_Environment) issuer.

The instructions below reference the following environment variables which you will need to set for your deployment

  * **CORE_NAME** The name assigned to the core Kubeflow ksonnet components (this is the name chosen when you ran `ks generate...`).
  * **ENVIRONMENT** The name of the ksonnet environment where you want to deploy Kubeflow.
  * **FQDN** The fully qualified domain name to use for your Kubeflow deployment.
  * **IP_NAME** The name of the GCP static IP that you created above and will be associated with **DOMAIN**.
  * **NAMESPACE** The namespace where Kubeflow is deployed.
  * **ACCOUNT** The email address for your ACME account where certificate expiration notifications will be sent.

```
ks generate cert-manager cert-manager --acmeEmail=${ACCOUNT}
ks apply ${ENVIRONMENT} -c cert-manager

ks generate iap-ingress iap-ingress --namespace=${NAMESPACE} --ipName=${IP_NAME} --hostname=${FQDN}
ks apply ${ENVIRONMENT} -c iap-ingress
```

After a few minutes the IAP enabled load balancer will be ready.

### Test ingress

It can take some time for IAP to be enabled. You can test things out using the following URL

```
https://${FQDN}/noiap/whoami
```

  * This a simple test app that's deployed as part of the iap-envoy component.
  * This URL will always be accessible regardless of whether IAP is enabled or not
  * If IAP isn't enabled the app will report that you are an anyomous user
  * Once IAP takes effect you will have to authenticate using your Google account and the app will tell you
    what your email is.

We also have the app running at `https://${FQDN}/whoami` - but this path will reject traffic that didn't go through IAP so it won't work unless IAP is enabled. In this case you should get
  a `401 UNAUTHORIZED` this indicates IAP isn't enabled so the request is rejected because it wasn't authenticated

### Configure Jupyter to use your Google Identity

We can configure Jupyter to use the identity provided by IAP. This way users won't have to login to JupyterHub.

```
ks param set ${CORE_NAME} jupyterHubAuthenticator iap
ks apply ${ENVIRONMENT} -c ${CORE_NAME}
# Restart JupyterHub so it picks up the updated config
kubectl delete -n ${NAMESPACE} pods tf-hub-0
```

You should now be able to access JupyterHub at

```
https://${FQDN}/hub
```

### Adding Users

IAP will block users who haven't been granted access to your web app. You can grant access using the following command.

```
gcloud projects add-iam-policy-binding $PROJECT \
  --role roles/iap.httpsResourceAccessor \
  --member user:${USER_EMAIL}
```

## Troubleshooting

Here are some tips for troubleshooting IAP.

 * Make sure you are using https

### 502 Server Error
* A 502 usually means traffic isn't even making it to the envoy reverse proxy
* A 502 usually indicates the loadbalancer doesn't think any backends are healthy
  * In Cloud Console select Network Services -> Load Balancing
      * Click on the load balancer (the name should contain the name of the ingress)
      * The exact name can be found by looking at the `ingress.kubernetes.io/url-map` annotation on your ingress object
      * Click on your loadbalancer
      * This will show you the backend services associated with the load balancer
          * There is 1 backend service for each K8s service the ingress rule routes traffic too
          * The named port will correspond to the NodePort a service is using
      * Make sure the load balancer reports the backends as healthy
          * If the backends aren't reported as healthy check that the pods associated with the K8s service are up and running
          * Check that health checks are properly configured
            * Click on the health check associated with the backend service for envoy
            * Check that the path is /healthz and corresponds to the path of the readiness probe on the envoy pods
            * Seel also [K8s docs](https://github.com/kubernetes/contrib/blob/master/ingress/controllers/gce/examples/health_checks/READMEmd#limitations) for important information about how health checks are determined from readiness probes.

          * Check firewall rules to ensure traffic isn't blocked from the GCP loadbalancer
              * The firewall rule should be added automatically by the ingress but its possible it got deleted if you have some automatic firewall policy enforcement. You can recreate the firewall rule if needed with a rule like this

               ```
               gcloud compute firewall-rules create $NAME \
              --project $PROJECT \
              --allow tcp:$PORT \
              --target-tags $NODE_TAG \
              --source-ranges 130.211.0.0/22,35.191.0.0/16
               ```

                * To get the node tag

                ```
                # From the GKE cluster get the name of the managed instance group
                gcloud --project=$PROJECT container clusters --zone=$ZONE describe $CLUSTER
                # Get the template associated with the MIG
                gcloud --project=kubeflow-rl compute instance-groups managed describe --zone=${ZONE} ${MIG_NAME}
                # Get the instance tags from the template
                gcloud --project=kubeflow-rl compute instance-templates describe ${TEMPLATE_NAME}

                ```

              For more info [see GCP HTTP health check docs](https://cloud.google.com/compute/docs/load-balancing/health-checks)

  * In Stackdriver Logging look at the Cloud Http Load Balancer logs

    * logs are labeled with the forwarding rule
    * The forwarding rules are available via the annotations on the ingress
      ```
      ingress.kubernetes.io/forwarding-rule
      ingress.kubernetes.io/https-forwarding-rule
      ```

 * Verify that requests are being properly routed within the cluster

  * Connect to one of the envoy proxies

  ```
  kubectl exec -ti `kubectl get pods --selector=service=envoy -o jsonpath='{.items[0].metadata.name}'` /bin/bash
  ```

  * Installl curl in the pod
  ```
  apt-get update && apt-get install -y curl
  ```

  * Verify access to the whoami app

  ```
  curl -L -s -i curl -L -s -i http://envoy:8080/noiap/whoami
  ```
    * If this doesn't return a 200 OK response; then there is a problem with the K8s resources

      * Check the pods are running
      * Check services are pointing at the points (look at the endpoints for the various services)
