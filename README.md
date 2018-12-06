# Wordpress k8s
![enter image description here](https://lh3.googleusercontent.com/NCCEs593gaqGnWHOJzZkzbWbIASLS3rLecls_FIvjUI8aLu_NSh4OV7BGIKg7dEvHMQYC3WX7hlfgA "wp")

### Clone do projeto:
```bash
git clone https://github.com/vandocouto/Wordpress-k8s.git
```
### Passo a Passo:
Passo 1 - Liste todos os labels do Cluster Kubernetes
```bash
kubectl get nodes --show-labels
NAME           STATUS   ROLES    AGE   VERSION   LABELS
kubernetes-1   Ready    master   25d   v1.12.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=kubernetes-1,node-role.kubernetes.io/master=
kubernetes-2   Ready    node     25d   v1.12.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,gitlabStorage=gitlab,kubernetes.io/hostname=kubernetes-2,nexusStorage=nexus,node-role.kubernetes.io/node=
```
Passo 2 - Crie um novo label para um dos nodes do Cluster Kubernetes
```bash
kubectl label nodes kubernetes-1 WPStorage=wordpress
node/kubernetes-1 labeled
```
Passo 3 - Crie o diretório /storage no mesmo node que recebeu o novo label WPStorage 
```bash
mkdir -p /storage
```
Passo 4 - Secret
Obs: Informe o .crt .key codificado em base64 + Senha do MySQL codificada também em base64 
```bash
apiVersion: v1
kind: Secret
metadata:
  name: wordpress-tls
  namespace: wp
type: Opaque
data:
  tls.crt: Certificate.crt encode base64
  tls.key: Certificate.key encode base64
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-pass
  namespace: wp
type: Opaque
data:
  password: YWRtaW4=
```
Passo 5 - PersistentVolume - mysql e wordpress
```bash
kind: PersistentVolume  
apiVersion: v1  
metadata:  
  name: mysql  
  labels:  
    type: local  
spec:  
  storageClassName: mysql  
  capacity:  
    storage: 100Gi  
  accessModes:  
    - ReadWriteOnce  
  hostPath:  
    path: "/storage/wordpress/mysql"  
---  
kind: PersistentVolumeClaim  
apiVersion: v1  
metadata:  
  name: mysql  
  namespace: wp  
  labels:  
   name: mysql  
spec:  
  storageClassName: mysql  
  accessModes:  
    - ReadWriteOnce  
  resources:  
    requests:  
      storage: 100Gi  
---  
kind: PersistentVolume  
apiVersion: v1  
metadata:  
  name: wordpress  
  labels:  
    type: local  
spec:  
  storageClassName: wordpress  
  capacity:  
    storage: 100Gi  
  accessModes:  
    - ReadWriteOnce  
  hostPath:  
    path: "/storage/wordpress/www"  
---  
kind: PersistentVolumeClaim  
apiVersion: v1  
metadata:  
  name: wordpress  
  namespace: wp  
  labels:  
   name: wordpress  
spec:  
  storageClassName: wordpress  
  accessModes:  
    - ReadWriteOnce  
  resources:  
    requests:  
      storage: 100Gi
```
Passo 6 - Deployment - mysql e wordpress
```bash
apiVersion: extensions/v1beta1  
kind: Deployment  
metadata:  
  name: mysql  
  namespace: wp  
spec:  
  replicas: 1  
  selector:  
    matchLabels:  
      name: mysql  
  template:  
    metadata:  
      labels:  
        name: mysql  
    spec:  
      containers:  
      - image:  mysql:5.7  
        imagePullPolicy: Always  
        env:  
        - name: MYSQL_ROOT_PASSWORD  
          valueFrom:  
            secretKeyRef:  
              name: mysql-pass  
              key: password  
        - name: MYSQL_USER  
          value: admin  
        - name: MYSQL_PASSWORD  
          valueFrom:  
            secretKeyRef:  
              name: mysql-pass  
              key: password  
        name: mysql  
        ports:  
        - containerPort: 3306  
        volumeMounts:  
         - mountPath: /var/lib/mysql  
           name: mysql  
      volumes:  
       - name: mysql  
         persistentVolumeClaim:  
          claimName: mysql  
      nodeSelector:  
        WPStorage: wordpress  
      restartPolicy: Always  
---  
apiVersion: extensions/v1beta1  
kind: Deployment  
metadata:  
  name: wordpress  
  namespace: wp  
spec:  
  replicas: 1  
  selector:  
    matchLabels:  
      name: wordpress  
  template:  
    metadata:  
      labels:  
        name: wordpress  
    spec:  
      containers:  
      - image: wordpress:latest  
        imagePullPolicy: Always  
        env:  
        - name: WORDPRESS_DB_HOST  
          value: mysql  
        - name: WORDPRESS_DB_PASSWORD  
          valueFrom:  
           secretKeyRef:  
            name: mysql-pass  
            key: password  
        name: wordpress  
        ports:  
        - containerPort: 80  
        volumeMounts:  
         - mountPath: /var/www/html  
           name: wordpress  
      volumes:  
       - name: wordpress  
         persistentVolumeClaim:  
          claimName: wordpress  
      nodeSelector:  
       WPStorage: wordpress  
      restartPolicy: Always
```
Passo 7 - Service - mysql e wordpress

```bash
apiVersion: v1  
kind: Service  
metadata:  
  name: mysql  
  namespace: wp  
spec:  
  type: NodePort  
  ports:  
  - name: mysql  
    port: 3306  
    targetPort: 3306  
    protocol: TCP  
    nodePort: 32006  
  selector:  
    name: mysql  
---  
apiVersion: v1  
kind: Service  
metadata:  
  name: wordpress  
  namespace: wp  
spec:  
  ports:  
  - name: http  
    port: 80  
    targetPort: 80  
    protocol: TCP  
  selector:  
    name: wordpress
```
Passo 8 - Ingress - wordpress https
Caso não tenha o ingress já configurado no seu Cluster Kubernetes, basta seguir a documentação do ingress: [Ingress com Nginx](https://github.com/nginxinc/kubernetes-ingress/blob/master/docs/installation.md) 

```bash
apiVersion: extensions/v1beta1  
kind: Ingress  
metadata:  
  name: wordpress  
  namespace: wp  
  annotations:  
    ingress.kubernetes.io/proxy-body-size: 100m  
    kubernetes.io/tls-acme: "true"  
  kubernetes.io/ingress.class: "nginx"  
spec:  
  tls:  
  - hosts:  
    - blog.dominio.com.br  
    secretName: wordpress-tls  
  rules:  
  - host: blog.dominio.com.br  
    http:  
      paths:  
      - path: /  
        backend:  
          serviceName: wordpress  
          servicePort: 80
```
Passo 9 - Execute o kubectl apply na ordem abaixo:
```bash
kubectl apply -f namespaces.yaml
kubectl apply -f storage.yaml
kubectl apply -f secrect.yaml
kubectl apply -f deployment.yaml
kubeclt apply -f service.yaml
kubectl apply -f ingress.yaml
```
Passo 10 - Verificando o status dos PODS:

```bash
 kubectl get pod -n wp
NAME                         READY   STATUS    RESTARTS   AGE
mysql-6c6447484c-trbfq       1/1     Running   0          15s
wordpress-79d545cb45-wmrcj   1/1     Running   0          15s
```
Passo 11 - Verificando os logs do POD do Wordpress
```bash
kubectl logs -f wordpress-79d545cb45-wmrcj -n wp

WordPress not found in /var/www/html - copying now...
Complete! WordPress has been successfully copied to /var/www/html
```
https://blog.dominio.com.br

![enter image description here](https://lh3.googleusercontent.com/laOdfxVlsEULnaoUIUrO3WbC77FmOxc9fD1wEyfSyuuahGHGd7C78k18vi1LZcjrRcc-uXycZ6yu7Q=s1024 "wp-1")
![enter image description here](https://lh3.googleusercontent.com/aGMnedmq-YBBNAHO4UpM6CZzQ7p9x9ntgFDtnfh7NmreAAP8rPOvMd3E6s9nUvo30ot_OFjPztkyeQ=s1024 "wp-2")
![enter image description here](https://lh3.googleusercontent.com/XaQjnfUA9dHukkmvXfHTPckGfq0lvW8OGSMxigxLJiVKxJfGL5C2S3st05HRDaSgoweDmMlLvHn2fA=s1024 "wp-3")
![enter image description here](https://lh3.googleusercontent.com/r_u7TEurIpwK58UWBy4B-MntP2nTc9w48TaUzwSOkriNWWCqgW0TMHSl61Z43ly4bgw__a--BIDtcw=s1024 "wp-4")
