# mysql-kubernetes-yamls
This article will show you how to deploy a MySQL database instance on Kubernetes using persistent volumes. This feature enables stateful apps to overcome the inherent transience of the K8s pods.
### WHAT YOU NEED
To successfully deploy a MySQL instance on Kubernetes, create a series of YAML files that you will use to define the following Kubernetes objects:

1.  Kubernetes secret for storing the database password.
2.  Persistent Volume (PV) to allocate storage space for the database.
3.  Persistent Volume Claim (PVC) that will claim the PV for the deployment .
4. deployment 
5. The Kubernetes Service.

Step 1: Create Kubernetes Secret >> sql-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: sql-secret
type: kubernetes.io/basic-auth
stringData:
  password: nguru1234

###
kubectl create -f sql-secret.yaml

##
kubectl get secret | grep sql-secret

##########################################################
Step 2: Create Persistent Volume and Volume Claim >> sql-storage.yaml


####
This deployment is for single-instance MySQL deployment. It means that the deployment cannot be scaled - it works on exactly one Pod.
This deployment does not support rolling updates. Therefore, the spec.strategy.type must always be set to Recreate.
#####
apiVersion: v1
kind: PersistentVolume
metadata:
  name: sql-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 18Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sql-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 18Gi
      
kubectl create -f sql-storage.yaml
      
######################################
Step 3: Create MySQL Deployment >>>>> sql-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: sql
spec:
  selector:
    matchLabels:
      app: sql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: sql
    spec:
      containers:
      - image: mysql:8.0
        name: sql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: sql-secret
              key: password
        ports:
        - containerPort: 3306
          name: sql
        volumeMounts:
        - name: sql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: sql-persistent-storage
        persistentVolumeClaim:
          claimName: sql-pv-claim
---
apiVersion: v1
kind: Service
metadata:
  name: sql
spec:
  ports:
  - port: 3306
  selector:
    app: sql
    
 #######################
 Access Your MySQL Instance
 
 kubectl get pod | grep sql
 
 #####
 kubectl exec --stdin --tty sql-694d95668d-w7lv5 -- /bin/bash
 kubectl exec -it sql-23444 bash
 
 
 
 #####
 
mysql -p


######incase you want to delete
####every kubernetes api resource has short forms ep - enpoints, svc - service , pv - persistenctvolume

kubectl delete deployment,svc sql
kubectl delete pvc sql-pv-claim
kubectl delete pv sql-pv-volume
kubectl delete secret mysql-secret
