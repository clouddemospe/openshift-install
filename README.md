# RH OCP Lessons Learned & Guide
Trucos & Tips de instalación de Red Hat OpenShift v4.x en VMWare


## Configuración de baremetal en IBM Cloud
### Configuración VMWare infrastructure
Para configurar el ip público seguir estos pasos
https://cloud.ibm.com/docs/infrastructure/virtualization?topic=Virtualization-enabling-public-access-to-vmware-esxi

- [ ] Configurar una nueva nic de virtual kernel con la ip pública asignada
- [ ] Crear un portgroup para la nueva nic de virtual kernel y uno para red pública, ambos atachados al vSwitch1
- [ ] Crear un nuevo virtual switch y atachar el interfaz 1, esta tiene la salida a Internet o asegurarse según lo indica el portal de IBM Cloud el interfaz público. En realidad deberían haber dos virtual switches vSwitch0 y vSwitch1. En el 0 va el VirtualKernel local y en el 1 va el interfaz público y el VirtualKernel público.
- [ ] Ingresar vía consola ssh al esxi y configurar según indica el enlace
- [ ] Deshabilitar el acceso vía ssh y a la consola web vía interfaz público

Para instalar el vCenter (requiere una cuenta en VMWare o solicitar un trial)
- [ ] Descargar de la página de VMWare el iso del appliance de vCenter
- [ ] En una máquina linux o windows con acceso al ESXi montar el iso
- [ ] Si es linux instalar interfaz gráfica (Gnome y tools) y luego ir la carpeta lin64, desplegar el instalador
- [ ] Configurar la conexión al ESXi e instalar el appliance

## Configuración pre-requisitos
- [ ] Configurar un servidor DNS y considerar el siguiente ejemplo
- [ ] Configurar un servidor DHCP con ips a hosts estáticas y MAC Address permitidas por OCP 4.x
- [ ] Configurar un servidor HTTP con un virtual host al dominio solicitado
- [ ] Configurar un balanceador de carga y considerar el siguiente ejemplo
     > **Nota:** Se pueden configurar todos los servicios y el balanceador de carga en la misma máquina virtual.

## Despliegue del cluster
- [ ] Configuración templates y máquinas virtuales
- [ ] Configuración de *bootstrap*
```
openshift-install wait-for bootstrap-complete --log-level=debug
```
- [ ] Detener o apagar la máquina de bootstrap y eliminar o comentar entrada en el balanceador de carga
- [ ] Aprobar nodos como parte del clúster
```
oc get csr --no-headers | awk '{print $1}' | xargs oc adm certificate approve
```
- [ ] Revisar nodos del cluster
```
oc get nodes
```
## Configurar el registro de imágenes.
- [ ] Opción 1: Configurar con EmptyDir (solo para ambientes poc)
```
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}}}}'
```
- [ ] Opción 2: Registro con almacenamiento compartido
```
kubectl patch storageclass thin -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
oc get pvc -n openshift-image-registry
oc delete pvc image-registry-storage -n openshift-image-registry
oc create -f pv-registry.yaml
oc create -f pvc-registry.yaml
oc edit configs.imageregistry.operator.openshift.io
```
- Archivo de configuración de volumen
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: image-registry-pv
spec:
  capacity:
    storage: 100Gi 
  accessModes:
  - ReadWriteMany
  nfs:
    path: /mnt/data
    server: <IP>
  persistentVolumeReclaimPolicy: Recycle
```
- Archivo de configuración de reclamación de volumen
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    imageregistry.openshift.io: "true"
  name: image-registry-storage
  namespace: openshift-image-registry
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
```
- [ ] Modificar el operador del registy. Durante su modificación existen dos entradas "storage", modifique ambas entradas como se muestra a continuación:
```
storage:
    pvc:
      claim: image-registry-storage
…..
storage:
    pvc:
      claim: image-registry-storage
  storageManaged: false
```
- [ ] Configurar la ruta de acceso al registro de imágenes
```
oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
```
Para loguearse en el registry de OCP v4.x:
```
oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
oc login https://api.ocp4.demo.com:6443 -u admin -p db2admin#ICP
HOST=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')
podman login -u $(oc whoami) -p $(oc whoami -t) --tls-verify=false $HOST
```
Si usa kubeadmin:
```
HOST=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')
podman login -u kubeadmin -p $(oc whoami -t) --tls-verify=false $HOST
```

## Finalizar la instalación del clúster
- [ ] Ejecutar el comando de verificación
```
openshift-install wait-for install-complete --log-level=debug
```
Se obtendrá el siguiente resultado
```
DEBUG Cluster is initialized                       
INFO Waiting up to 10m0s for the openshift-console route to be created... 
DEBUG Route found in openshift-console namespace: console 
DEBUG Route found in openshift-console namespace: downloads 
DEBUG OpenShift console route is created           
INFO Install complete!                            
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/root/icpa4/auth/kubeconfig' 
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.ocp4.demo.com 
INFO Login to the console with user: kubeadmin, password: ***********************
```
Comprobar el acceso del clúster via la consola
```
 oc login https://api.ocp4.demo.com:6443 -u kubeadmin -p ***********************
```
## Configurar Identity Providers
### Via archivo
- [ ] Opción 1: desde línea de comandos
 oc create secret generic htpass-secret --from-file=htpasswd=/etc/httpd/conf/users.htpasswd -n openshift-config
 oc apply -f htpasswd_provider.yaml

- [ ] Opción 2: (Preferida) desde el UI o consola de Openshift

### Via LDAP
- [ ] Configurar LDAP auth con JumpCloud

ldap://host:port/basedn?attribute?scope?filter
From <https://docs.openshift.com/container-platform/4.2/authentication/identity_providers/configuring-ldap-identity-provider.html> 
https://support.jumpcloud.com/support/s/article/using-ldapsearch-with-jumpcloud1
https://medium.com/@yildirimabdrhm/openshift-ldap-integration-7137088c990d

```
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
annotations:
release.openshift.io/create-only: 'true'
creationTimestamp: '2020-01-22T23:33:47Z'
generation: 5
name: cluster
resourceVersion: '614529'
selfLink: /apis/config.openshift.io/v1/oauths/cluster
uid: a30369a8-3d6f-11ea-9dfe-0050569385ae
spec:
identityProviders:
- ldap:
attributes:
email:
- mail
id:
- dn
name:
- cn
preferredUsername:
- uid
bindDN: 'uid=admin,ou=Users,o=5e18b5458d06685de549871c,dc=jumpcloud,dc=com'
bindPassword:
name: ldap-bind-password-cbcz2
insecure: false
url: >-
ldap://ldap.jumpcloud.com:389/o=5e18b5458d06685de549871c,dc=jumpcloud,dc=com?uid
mappingMethod: claim
name: ldap-provider
type: LDAP
```
Asignar permisos de administrador del cluster al nuevo usuario:
```
oc create clusterrolebinding ldap-admin --clusterrole=cluster-admin --user=admin
oc adm policy add-cluster-role-to-user cluster-admin admin

```

### Para crear volumenes con VMWare:
- Loguearse al VMWare ESXi vía root
- Ejecutar el comando
```
vmkfstools -c 100G /vmfs/volumes/datastore1/volumes/block01.vmdk
```
- Luego crear un volumen
```
[root@bastion quay-operator]# vim pv-block-quay.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-quay-block
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  vsphereVolume:
    volumePath: "[datastore1] volumes/block01.vmdk"
    fsType: ext4
```

## Configuración de Logging
- Configurar logging: https://docs.openshift.com/container-platform/4.3/logging/cluster-logging-deploying.html
	1. Crear el namespace openshift-logging
	2. Crear el PV y el PVC de mínimo 200 GB
	3. Instalar el Operador de Cluster Logging
	4. Crear la instancia de Cluster Logging
	5. Instalar el Operador de Elasticsearch
	6. Para Demos usar NFS para producción usar Bloque, para demo usar 
	ZeroRedundancy
	
From <https://docs.openshift.com/container-platform/4.1/logging/config/efk-logging-elasticsearch.html> 
Para ambientes productivos usar mínimo 2 réplicas, considere que cada réplica utiliza 16 GB RAM

- Actions > Edit Secret
- Enabling Elasticsearch to Mount the Directory
- The installation of Elasticsearch will fail because there is currently no way to grant the Elasticsearch service account permission to mount that directory during   installation. After installation is complete, do the following steps to enable Elasticsearch to mount the directory:
```
oc project openshift-logging
oc adm policy add-scc-to-user hostmount-anyuid \
   system:serviceaccount:openshift-logging:aggregated-logging-elasticsearch
oc rollout cancel $( oc get -n openshift-logging dc -l component=es -o name )
oc rollout latest $( oc get -n openshift-logging dc -l component=es -o name )
oc rollout status -w $( oc get -n openshift-logging dc -l component=es -o name )
```
From <https://github.com/ViaQ/Main/blob/master/README-install.md#enabling-elasticsearch-to-mount-the-directory> 

- Configurar Curator:
```
oc edit configmap curator -n openshift-logging
  config.yaml: |
    # Logging example curator config file

    # uncomment and use this to override the defaults from env vars
    .defaults:
      delete:
        days: 5

    # to keep ops logs for a different duration:
    .operations:
      delete:
        weeks: 1
```
From <https://docs.openshift.com/container-platform/4.4/logging/config/cluster-logging-curator.html>
