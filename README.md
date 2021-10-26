# rtf-custom-admission-controller-webhook

## Introduction

This is a kubernetes custom admission controller written in Mule that can modify the RTF application deployments before they are created. We are using this to inject a peristent volume claim to the deployment. This can be used to provide RTF applications access to external storage for loading certificates, files etc.


![WebHook (1)](https://user-images.githubusercontent.com/36458155/135758609-4eaa18d6-2947-44d0-aa72-2846fd2aea92.jpeg)

## Installation Steps

### Create Persistent Volume (PV)

Persistent volumes (PV) are not namespace bound, but Persistent Volume Claims (PVC) are. There is a one to one relationship between PVC and PV. So depeneding on the number of namespaces (RTF environments) you have in a cluster, you will need that many PV.

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: rtf-pv
  namespace: rtf
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /
    server: <<<NFS Host>>>
```

Update the NFS host and path above and save the file as rtf-pv.yaml
Create the PV 

``` kubectl apply -f rtf-pv.yaml ```

### Create Persistent Volume Claim (PVC)

PVC is namespace bound, so create one for each RTF environment namespace you have.

```diff
- Please note that the webhook application uses the name rtf-pvc for PVC name. So please keep the name of PVC as rtf-pvc. If you change this please update the dataweave inside the webhook applicatin to reflect the same.
```

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rtf-pvc
  namespace: <<Update namesspace here>>
spec:
  storageClassName: nfs
  accessModes:
    - ReadOnlyMany
  resources:
    requests:
      storage: 1Gi
```

Update the namspace above and save file as rtf-pvc.yaml
Create PVC by running 

```kubectl apply -f rtf-pvc.yaml```

### Deploy custom RTF admission controller webhook

```diff
+We only have to deploy one admission controller per cluster. This webhook is developed in Mule and is to be deployed through ARM. So choose one environment in the cluster where this application will be deployed.
```

#### Create TLS certificates for the webhook ####
Webhook is called by kube-api server using TLS. We need to generate TLS certificates for the https listener. This certificate is to be signed by the cluster CA.
Please use the below script to generate the TLS key and certificate.

```diff
- The TLS certificates will be created for app-name.namespace.svc. In this example we are using app-name as rtf-webhook. If you change this name, please make sure to change it everywhere.
- The namespace is the rtf environment in which you choose to deploy this application. This only has to be deployed to one of the environments in the cluster.
```
copy [ssl.sh](ssl.sh) and run it as 
```
./ssl.sh rtf-webhook 31ce1ad1-2018-4159-9749-e0decade32b6
```

This will create files called **rtf-webhook.key** and **rtf-webhook.pem**

Convert the above files to pkcs12 format

```
openssl pkcs12 -export -in rtf-webhook.pem -inkey rtf-webhook.key -name 'rtfcert' -out keystore.p12
```
In the rtf-webhook mule application use this certificate for TLS context in the https listener. 


#### Deploy WebHook Mule Application ####

After updating the tls certificate deploy the *rtf-webhook-mule* application to your cluster to the environment that you chose while creating TLS certificate.

### Create Mutating Admission Controller ###

Get the CA bundle from cluster by running below script

```
kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}'
```

Create the below yaml as mutatewebhook.yaml and replace ${CA Bundle} with the output from above command.

```
apiVersion: admissionregistration.k8s.io/v1beta1
kind: MutatingWebhookConfiguration
metadata:
  name: rtfwebhookmutate
  labels:
    app: rtfwebhookmutate
webhooks:
  - name: rtf-webhook.31ce1ad1-2018-4159-9749-e0decade32b6.svc
    admissionReviewVersions: ["v1", "v1beta1"]
    matchPolicy: Equivalent
    namespaceSelector:
    objectSelector:
    clientConfig:
      caBundle: ${CA Bundle}
      service:
        namespace: 31ce1ad1-2018-4159-9749-e0decade32b6
        name: rtf-webhook
        path: /mutate
        port: 8081
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: ["*"]
        apiVersions: ["*"]
        resources: ["deployments"]
        scope: "*"
        
 ```

Create mutating webhook by running 
```
kubectl apply -f mutatewebhook.yaml
```

If you have done all the steps above correctly when you deploy a new Mule application the persistent volume will be injected at the path 
***/opt/mule/appdata***
This path is available as an environment variable and Mule apps can refer to it as ${APP_DATA}
```
<tls:key-store type="pkcs12" path="${APP_DATA}/Cert.p12" password="mulesoft" keyPassword="mulesoft"/>
```



