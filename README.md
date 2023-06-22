# Eramba_on_DigitalOcean
Leveraging DigitalOcean to serve multi-tenant Eramba GRC software based on namespaces in one K8S cluster

⚠️⚠️⚠️PLEASE PERFORM SECURITY PENETRATION TESTING IF YOU ARE DEPLOYING THIS TO PRODUCTION⚠️⚠️⚠️

# Eramba Deployment on DO

On the ubuntu VM

`sudo snap install doctl`

`mkdir ~/.config`

`doctl auth init --context eramba-test` eramba-test is the authentication context, feel free to change it

- You will need a personal access token for this step

`doctl auth list`

`doctl auth switch --context <NAME>` to switch to the context we had just created

`doctl account get` to get the account, you should see the account email

`doctl compute droplet create --region tor1 --image ubuntu-18-04-x64 --size s-1vcpu-1gb test-droplet` test if you have write access

`doctl compute droplet delete <DROPLET-ID>`

`doctl kubernetes cluster list` to see all clusters

`doctl kubernetes cluster kubeconfig save eramba-test-cluster` to use the GRC cluster as default

May need to run `sudo snap connect doctl:kube-config` if this is needed, you’ll have to run the command again ⬇️

`doctl kubernetes cluster kubeconfig save eramba-test-cluster`

`kubectl get nodes` you should only see the nodes for the DO cluster nodes now

## Successful Deployment

`kubectl create ns eramba-1`

`kubectl create -f eramba-cm.yaml`

`kubectl create -f eramba2-cm.yaml`

`helm upgrade -i eramba bitnami/mariadb --set auth.rootUsername=root,auth.rootPassword=SOME_PASSWORD_HERE,auth.database=erambadb,initdbScriptsConfigMap=eramba2,volumePermissions.enabled=true,mariadb.volumePermissions.enabled=true,primary.existingConfigmap=eramba2cm --namespace eramba-1 --set mariadb.volumePermissions.enabled=true --set data.persistence.storageClass=do-block-storage`

`kubectl create -f eramba-web.yaml`

**Once the application pod is running, exec into the pod and configure all cron jobs**

### Debugging

- If system health check fails, check the cluster firewalls. The default firewall for pod communication should be properly configured.

### Considerations

- Traffic isolation between namespaces
- Resourc quota

## Ingress Controller Deployment

`kubectl create ns eramba-1`

`kubectl create -f eramba-cm.yaml`

`kubectl create -f eramba2-cm.yaml`

`helm upgrade -i eramba bitnami/mariadb --set auth.rootUsername=root,auth.rootPassword=SOME_PASSWORD_HERE,auth.database=erambadb,initdbScriptsConfigMap=eramba2,volumePermissions.enabled=true,mariadb.volumePermissions.enabled=true,primary.existingConfigmap=eramba2cm --namespace eramba-1 --set mariadb.volumePermissions.enabled=true --set data.persistence.storageClass=do-block-storage`

`kubectl create -f eramba-web.yaml`

- The service will have a type of ClusterIP

**NOTE if you are deploying another APPLICATION instance (aka another Eramba in a different namespace), DO NOT deploy another ingress controller, jump to** Deploy multiple instances for more details

`helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx`

`helm repo update`

`helm install nginx-ingress ingress-nginx/ingress-nginx --set controller.publishService.enabled=true --namespace eramba-1`

- Note - i dont think i deployed ingress in the eramba-1 ns (if upgrading the helm chart, omit the `—-namespace` config option)

`kubectl create -f eramba-ingress.yaml`

- Note - you CAN deploy the ingress for initial application instance without any `host` value at this stage. When TLS is configured at later step using `eramba-ingress-tls.yaml`, you MUST config the `host` value in the yaml file.

Grab the IP (will be used to create domain) 

- The IP should work
<img width="1440" alt="Screen Shot 2022-01-27 at 2 17 41 PM" src="https://github.com/regneisokgen/Eramba_on_DigitalOcean/assets/54826294/c054cd6c-600a-478a-a982-639c8e4d6e85">

Once the domain is created, the ns should be changed to digitalocean’s. An A record will then be created for that domain and IP. 

## Deploy multiple instances

1. Right after deploying the `eramba-web.yaml` file, `kubectl create -f`  another yam file for the ingress resource. Except, the host will be a subdomain (the example yaml file is `eramba-anotherinstance-ingress.yaml` on TD laptop /desktop) 
2. Create a CNAME record on digitalocean (for example, there is a CNAME record of *.example.application.ca on goDaddy) 
   
<img width="1157" alt="Screen Shot 2022-01-27 at 3 47 13 PM" src="https://github.com/regneisokgen/Eramba_on_DigitalOcean/assets/54826294/67227b65-9653-4928-8baf-cffd85b8e957">

## For TLS deployment

The following to to provide TLS to deployed application instances on digitalocean (followed by above deployment steps)

- I recommend making sure the instance is properly deployed and accessible via *.example.application.ca before following the below steps for TLS
1. If the cluster does not have a cert-manager, install it using https://cert-manager.io/docs/installation/helm/#4-install-cert-manager
2. Config a Let’s Encrypt Issuer (the type of `ClusterIssuer`). You can use the `eramba-cluster-cert-manager.yaml` file to deploy it. 
    1. You may change the server parameter to the staging Let’s Encrypt url for testing. 
3. Create a certificate for the application instance using `cmd-for-creating-cert.txt` 
    1. https://faun.pub/wildcard-k8s-4998173b16c8 for more info
4. Check the status of the cert using `kubectl describe certificates <<cert name>> -n eramba-1`
    1. `kubectl get certificate -n eramba-1` shall show the certificate is `true` for READY 
5. Re-deploy the ingress using the `eramba-ingress-tls.yaml` file so the ingress can use the Certificate we created in step 4.

The above steps are inspired from https://cert-manager.io/docs/tutorials/acme/nginx-ingress/#step-7---deploy-a-tls-ingress-resource

## Renewing TLS certificates

Certificate should be automatically renewed 30 days prior to expiration. If past the 30 days mark, the certificate is not renewed. Manual renewal must be performed. 

### Manual renew

### Renew

`cmctl` allows you to manually trigger a renewal of a specific certificate. This can be done either one certificate at a time, using label selectors (`-l app=example`), or with the `--all` flag:

For example, you can renew the certificate `example-com-tls`:

`$ kubectl get certificateNAME                       READY   SECRET               AGEexample-com-tls            True    example-com-tls      1d$ cmctl renew example-com-tlsManually triggered issuance of Certificate default/example-com-tls$ kubectl get certificaterequestNAME                              READY   AGEexample-com-tls-tls-8rbv2         False    10s`

You can also renew all certificates in a given namespace:

`$ cmctl renew --namespace=app --all`

The renew command allows several options to be specified:

- `-all` renew all Certificates in the given Namespace, or all namespaces when combined with `-all-namespaces`
- `A` or `-all-namespaces` mark Certificates across namespaces for renewal
- `l` `-selector` allows set a label query to filter on as well as `kubectl` like global flags like `-context` and `-namespace`.

## Debugging

1. The above shall not cause issues on digitalOcean. In most cases, it is the node pool that is causing the resources to NOT be deployed (if you see “mysql cant connect to database at the URL endpoint, increase node pool for the pods to run on digitalocecan)
