# Kubernetes

A intenção deste repositório é fornecer um **Vagrantfile** capaz de criar um cluster kubernetes, fazendo com que a interação com o cluster esteja mais próxima dos ambientes de produção.
Abaixo há uma pequena introdução a respeito do universo dos contâineres e o papel do Kubernetes (k8s) neste universo, logo após, dezenas de objetos são exemplificados com breves descrições de seu uso.

## Container

É uma espécie de virtualização, o encapsulamento e isolamento de recursos controlados a nível de processo. Uma imagem contendo todas as dependências da aplicação, podendo rodar independete de qualquer sistema. Uma aplicação auto-contida.

- **cgroups** - Permite ao host compartilhar e limitar recursos que cada processo utilizará. Limita o quanto pode usar.
- **Namespaces** - Cria a área chamada de contêiner, limitando a visualização que o processo possuí dos demais processos, filesystems, redes e componentes de usuário. Limita o quanto pode ver.
- **Union filesystems** - Conceito de copy-on-write, utiliza-se de camadas já existentes - snapshots - e cria uma camada superficial de escrita, lendo o que for necessário nas camadas inferiores e ao alterá-las, copia a modificação para a camada superior.

## Pod

É a únidade mínima do Kubernetes, pode conter um ou mais contêineres. Os pods agrupam lógicamente os contêineres em termos de rede e hardware. O processo entre estes contêineres pode acontecer sem a alta latência de ter que atravessar uma rede. Assim como dados comuns podem ser guardados em volumes que são compartilhados entre esses contêineres.

## Orquestrador

Microserviços são construídos em cima de uma rígida organização. Com o tempo, alocação de recursos, self-healing, táticas de deploy, proximidade de serviços, movimentação de containers e rollbacks começam a aumentar a complexibilidade de uma infraestrutura. É neste momento que um orquestrador é necessário, ou seja, o Kubernetes.

# Instalação

[https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm)

Escolher **calico** como gerenciador de rede interno dos pods, por ser o mais comum e implementar segurança adicional.
Considerando o uso do Vagrantfile, existem três servidores, um master e dois workers.

```
kubeadm init --apiserver-advertise-address=192.168.10.10 --pod-network-cidr=10.244.0.0/16
mkdir -p ~/.kube
cp /etc/kubernetes/admin.conf ~/.kube/config
curl -s https://docs.projectcalico.org/v3.6/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml > /root/calico.yml
sed -i "s?192.168.0.0/16?10.244.0.0/16?g" /root/calico.yml
kubectl apply -f /root/calico.yml
```

Para verificar se o cluster está funcionando correntamente:

```
kubectl get nodes
kubectl cluster-info [dump]
```

# Objetos do Kubernetes

## Pod

Crie o arquivo **cgi-pod.yml**:

```
apiVersion: v1
kind: Pod
metadata:
  name: cgi-pod
  labels:
    app: cgi
spec:
  containers:
  - name: cgi-pod
    image: hectorvido/sh-cgi
    ports:
    - containerPort: 8080
```

Adicione o pod ao Kubernetes executando o seguinte comando:

```
kubectl create -f cgi-pod.yml
kubectl get pods
kubectl describe pods/cgi-pod
kubectl get pod cgi-pod --template={{.status.podIP}}
```

Isto criará um pod com um contêiner chamado **cgi-pod**.
Veja que o pod possuí um IP privado. A partir deste momento, podemos acessar este endereço em qualquer um dos nodes de nosso cluster.

## Labels

Labels são conjuntos de chave/valor que servem para agrupar determinados objetos dentro do Kubernetes. Podemos adicioná-los no momento da criação ou a qualquer momento depois. São utilizados para a organização e seleção de um conjunto de objetos.

## Services

Um objeto service utiliza labels para selecionar e fornecer um ponto de acesso em comum para um conjunto de pods, criando automaticamente um balanceamento de carga entre os pods disponíveis:

**cgi-service.yml**
```
apiVersion: v1
kind: Service
metadata:
  name: cgi-service
  labels:
    app: cgi
spec:
  selector:
    app: cgi
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

# Deployments

Deployment é um objeto de alto nível responsável por controlar a forma como os pods são recriados através da modificação do número desejado de réplicas em relação ao número atual.
Ao criar um deployment, três objetos aparecem:

 - Pod
 - ReplicaSet
 - Deployment

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cgi-deploy
  labels:
    app: cgi
spec:
  replicas: 8
  selector:
    matchLabels:
      app: cgi-deploy
  minReadySeconds: 5
  strategy:
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: cgi-deploy
    spec:
      containers:
      - name: cgi-deploy
        image: hectorvido/sh-cgi
        ports:
        - containerPort: 8080
```

# Dashboard

É possível acessar uma aplicação web para visualizar com mais facilidade alguns recursos do Kubernetes:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

No caso desta instalação com Vagrant, altere o arquivo de configuração do serviço da dashboard para o tipo NodePort, e verifique a porta disponibilizada pelo serviço:
```
kubectl -n kube-system edit service kubernetes-dashboard

### Disso ###
...
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: ClusterIP
...

### Para isso ###
...
    targetPort: 8443
    nodePort: 31000
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: NodePort
...

kubectl -n kube-system get service kubernetes-dashboard
```

Acesse o endereço "**https://172.27.11.10:31000**" para abrir a dashboard.

Para entrar na dashboard, será preciso criar um usuário, uma role e extrair seu token:

**dashboard-adminuser.yaml**
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
```

**clusterrolebinding-admin.yaml**
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```

Feito isso, aplique as configurações e extraia o token:

```
kubectl apply -f dashboard-adminuser.yaml
kubectl apply -f clusterrolebinding-admin.yaml
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

# Secrets e ConfigMaps

Os secrets e os configmaps são capazes de gerar valores que podem ficar disponíveis por todo o cluster, removendo a necessidade de atualizar variáveis de ambiente especificadas a um **Deployment**.
Ao contrário do que pareça ser, os Secrets não são nada mais do que ConfigMaps - com a excessão de que seus valores são codificados em base64. A única diferença real entre ambos é o fato de poder bloquear através de RBAC o acesso a qualquer um dos dois, geralmente o **Secret**. Dessa forma, usa-se a convenção de que o **ConfigMap** carrega informações não sigilosas.

##### Obs:
Caso se utilize **Secrets** ou **ConfigMaps** para preencher valores de variáveis de ambiente e haja necessidade de atualização destes valores, os pods precisarão ser recriados. O mesmo não é verdadeiro para o caso de utilizar qualquer um dos dois como volume.

**configmap.yml**
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: lighttpd-config
  labels:
    name: test
data:
  lighttpd.conf: |
    server.modules = (
        "mod_access",
        "mod_accesslog"
    )
    include "mime-types.conf"
    server.username      = "lighttpd"
    server.groupname     = "lighttpd"
    server.document-root = "/var/www/localhost/htdocs"
    server.pid-file      = "/run/lighttpd.pid"
    server.errorlog      = "/var/log/lighttpd/error.log"
    accesslog.filename   = "/var/log/lighttpd/access.log"
    server.indexfiles    = ("index.html", "index.sh")
    static-file.exclude-extensions = (".cgi", ".sh")
    server.modules += ("mod_cgi")
    cgi.assign = (
        ".sh" => "/bin/sh",
    )
```

Para colocar os valores no secret, devemos codificá-los para base64:
```
echo -n 'mysql.k8s.com' | base64 #bXlzcWwuazhzLmNvbQ==
echo -n '3306' | base64 #MzMwNg==
echo -n 'k8s' | base64 #azhz
echo -n 'kub3rn3ts' | base64 #a3ViM3JuM3Rz
```

**secret.yml**
```
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  labels:
    name: test
type: Opaque
data:
  db_host: bXlzcWwuazhzLmNvbQ==
  db_port: MzMwNg==
  db_user: azhz
  db_pass: a3ViM3JuM3Rz
```

Existem várias formas de tornar o secret disponível dentro do pod, um dos exemplos é como variáveis de ambiente e o outro como um arquivo em um volume:

```
apiVersion: v1
kind: Pod
metadata:
  name: pod
  labels:
    name: test
spec:
  containers:
  - name: container
    image: alpine
    stdin: true
    tty: true
    env:
      - name: HOST
        valueFrom:
          secretKeyRef:
            name: db-secret
            key: db_host
      - name: PORT
        valueFrom:
          secretKeyRef:
            name: db-secret
            key: db_port
      - name: USERNAME
        valueFrom:
          secretKeyRef:
            name: db-secret
            key: db_user
      - name: PASSWORD
        valueFrom:
          secretKeyRef:
            name: db-secret
            key: db_pass
    volumeMounts:
    - name: config-volume
      mountPath: /etc/lighttpd/
  volumes:
  - name: config-volume
    configMap:
      name: lighttpd-config
```

## Scheduling

O comportamento padrão do Kubernetes é espalhar os pods entre os nodes, dando preferência para os nodes com o menor número de instâncias daquele pod em questão.
Mas é possível adicionar restrições quanto ao CPU  e a memória.

**cgi-constraints.yml**
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cgi-constraints
  labels:
    app: cgi
spec:
  replicas: 3
  selector:
    matchLabels:
      name: cgi-constraints
  template:
    metadata:
      labels:
        name: cgi-constraints
    spec:
      containers:
      - name: cgi-constraints
        image: hectorvido/sh-cgi    
        ports:
        - containerPort: 8080
        resources:
          limits:
            memory: "512Mi"
            cpu: "500m"
```

Neste caso, o **1500m** equivale a 1.5 cpus, como cada máquina possuí dois, apenas um pod poderá ser executada em cada uma. O terceiro pod, ficará com status "pending", e ao detalharmos o pod, veremos a mensagem de CPU insuficiente.

# Scaling

Antes dos Deployments o escalamento dos pods eram feitos através do **ReplicaController**. Como este tipo de objeto se tornou obsoleto em relação ao **ReplicaSet** utilizado no Deployment, os exemplos serão apenas com Deployments.

**cgi-scaling.yml**
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cgi-deploy
  labels:
    name: cgi-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      name: cgi-deploy
  template:
    metadata:
      labels:
        name: cgi-deploy
    spec:
      containers:
      - name: cgi-deploy
        image: hectorvido/sh-cgi
        ports:
        - containerPort: 8080
```

**cgi-scaling-service.yml**
```
apiVersion: v1
kind: Service
metadata:
  name: cgi-deploy
  labels:
    name: cgi-deploy
spec:
  type: NodePort
  sessionAffinity: ClientIP
  ports:
  - port: 8080
    nodePort: 30000
  selector:
    name: cgi-deploy
```

Para escalar o número de pods e conseguir acompanhar a atualização, edite o deployment e modifique a imagem para **bitnami/nginx**:
```
kubectl edit deployment cgi-deploy
kubectl rollout status deployment cgi-deploy
kubectl rollout history deployment cgi-deploy
```

Veja que enquanto os PODs mudam e se atualizam, nada é preciso fazer no service, ele ainda se atualiza automaticamente conforme os contêineres vem e vão.

Agora volte para a versão anterior, com 2 replicas executando o seguinte comando:

```
kubectl rollout undo deployment cgi-deploy
kubectl rollout status deployment cgi-deploy
```

## Ingress

Apesar de ser possível expor serviços distribuídos no cluster diretamente com LoadBalancer ou NodePort, existem cenários de roteamento mais avançados. O **ingress** é utilizado para isso, pense nele como uma camada extra de roteamento antes que a requisição chegue ao serviço. Assim como uma aplicação possuí um serviço e seus pods, os recursos do ingress precisam de uma entrada no ingress e um controlador que executa uma lógica customizada. A entrada define a rota e o controlador faz o roteamento.
Pode ser configurado para fornecer URLs externas para os serviços, terminar o SSL, load balacing, e fornecer nomes para hosts virtuais como vemos em web servers.

Em serviços de cloud como DigitalOcean ou GCP, já existe um controlador de ingress pronto para ser utilizado, mas quando provisionamos o cluster por conta própria, precisamos instalar esse controlador. Um dos mais conveninentes e conhecidos parece ser o **nginx**:

[https://www.nginx.com/products/nginx/kubernetes-ingress-controller](https://www.nginx.com/products/nginx/kubernetes-ingress-controller)

### Instalação

Clone o repositório, troque o diretório por questões de comodidade:

```
git clone https://github.com/nginxinc/kubernetes-ingress.git
```

Então, adicione o NameSpace e a ServiceAccount, o Secret com o certificado TLS e o ConfigMap que poderá ser customizado com configurações para o nginx e o Role-Based Access Control para permitir acesso aos segredos:

```
kubectl apply -f kubernetes-ingress/deployments/common/ns-and-sa.yaml
kubectl apply -f kubernetes-ingress/deployments/common/default-server-secret.yaml
kubectl apply -f kubernetes-ingress/deployments/common/nginx-config.yaml
kubectl apply -f kubernetes-ingress/deployments/rbac/rbac.yaml
```

Existe a opção de criar um Deployment para especificar onde os pods do ingress ficarão, mas aqui criarei o DaemonSet, para termos um pod em cada node:

```
kubectl apply -f kubernetes-ingress/deployments/daemon-set/nginx-ingress.yaml
```

Para acelerar as coisas, crie o namespace, o deployment (altere para 4 réplicas) e o serviço através da linha de comando:

```
kubectl create ns cgi
kubectl create deployment cgi --image=hectorvido/sh-cgi -n cgi
kubectl patch deployment cgi -p '{"spec" : {"replicas" : 4}}' -n cgi
kubectl expose deployment cgi --port 80 --target-port 8080 -n cgi
kubectl get all -n cgi
```

Vamos criar um ingress e utilizá-lo para fazer roteamento através de hostname e terminação TLS, isso significa que nosso cluster passará a responder por um host e fará a comunicação segura entre o cliente e o cluster, mas do cluster para o pod a comunicação será comum.
Antes disso, vamos criar nosso certificado x509 auto-assinado, se você usa *Let's s Encrypt* o processo é o mesmo, você só não precisará gerar o certificado:

```
# Crie um novo certificado x509 sem criptografia na chave privada.
# Colocando a chave no arqivo key.pem e o certificado em cert.pem
openssl req -x509 -nodes -keyout key.pem -out cert.pem
```

Os certificados utilizados pelo ingress são salvos em um **secret**, portanto obrigatoriamente no formato **base64**. Felizmente existe um comando que facilita esta conversão e criação:

```
kubectl create secret tls cgi --key key.pem --cert cert.pem -n cgi
# Para visualizar o arquivo puro, como deveria ser feito:
kubectl edit secret cgi -n cgi
```

Com tudo pronto, definiremos o ingress que irá responder pelo **hostname** e apontar para um determinado serviço:

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cgi
  namespace: cgi
spec:
  tls:
  - hosts:
    - cgi.example.com
    secretName: cgi
  rules:
  - host: cgi.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: cgi
          servicePort: 80
```

Provisione o serviço no cluster e teste o endereço adicionando o ip de qualquer um dos **minions** no */etc/hosts* ou utilizando o parâmetro --resolv do curl:

```
kubectl apply -f sh-cgi-ingress.yml
curl -kL --resolv cgi.example.com:443:172.27.11.20 https://cgi.example.com
# Ao adicionar no /etc/hosts basta executar "curl cgi.example.com -Lk"
```

Pelo fato de utilizarmos um certificado auto-assinado, o parâmetro -k é obrigatório para o funcionamento.

# RBAC

Podemos limitar que determinados usuários consigam enxergar apenas um determinado namespace por questões de segurança. Para isso precisamos criar 4 objetos:

- Namespace
- ServiceAccount
- Role
- RoleBinding

O exemplo abaixo limitará um usuário específico a acessar apenas alguns poucos objetos dentro de um determinado namespace.

## Namespace

O namespace agrupa todos meus objetos em um único local, facilitando sua remoção:

**namespace.yml**
```
apiVersion: v1
kind: Namespace
metadata:
  name: belchior
```

O service account cria uma conta dentro do cluster:

**service-account.yml**
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: belchior
  namespace: belchior
```

A role define qual serão as permissões deste usuário dentro do cluster:

**role.yml**
```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: belchior
  namespace: belchior
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["*"]
  verbs: ["*"]
```

A diretiva **apiGroups** define o grupo de recursos que o usuário pode utilizar, o valor "" representa os tipos principais, já o "apps" representa os Deployments, RaplicaSets e ReplicaControllers.
A diretiva **resources** especifica quais objetos dentro dos **apiGroups** poderão ser acessados.

O role binding conecta um **ServiceAccount** a uma **Role**:

**role-binding.yml**
```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: belchior
  namespace: belchior
subjects:
- kind: ServiceAccount
  name: belchior
  namespace: belchior
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: belchior
```

Com a conta criada, será necessário criar um arquivo de configuração para que o **kubectl** possa simular as interações deste usuário. É preciso extrair o segredo do usuário e decodificar o valor que está em base64:

```
kubectl get secrets -n belchior
kubectl get secrets belchior-token-xxxxx -n belchior -o "jsonpath={.data.token}" | base64 -d
kubectl get secrets belchior-token-xxxxx -n belchior -o "jsonpath={.data['ca\.crt']}"
```

O valor do token e do certificado deverão ser colocados em um arquivo semelhante ao ~/.kube/config:

```
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: <certificado>
    server: https://192.168.10.10:6443
  name: kubernetes

users:
- name: belchior
  user:
    client-key-data: <certificado> 
    token: <token>
contexts:
- context:
    cluster: kubernetes
    namespace: belchior
    user: belchior
  name: belchior

current-context: belchior
```

Para executar comandos no cluster utilizando estas configurações, podemos passar o arquivo em um variável de ambiente:

```
KUBECONFIG=$(pwd)/belchior.yml kubectl create deployment cgi --image=hectorvido/sh
```

# Network Policy

O Calico é capaz de isolar o acesso da rede em muitos níveis, seguindo o exemplo anterior podemos limitar mais ainda a conta, isolando a rede deste namespace. Desta forma estes pods não podem ser acessados fora do namespace e não podem acessar outros pods fora do namespace:

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny
  namespace: belchior
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: belchior
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: belchior
```

# Volumes

O Kubernetes suporta diversos tipos de volume, por exemplo:

- NFS
- iSCSI
- GlusterFS
- CephFS
- Cinder

Cada volume é definido através de um **PersistentVolume**, que disponibiliza o volume para o cluster, e os usuário requisitam estes volumes através de um **PersistentVolumeClaim**.

## Mode

É possível através da diretiva **volumeMode** especificar o uso de dispositivos inteiros com **block** ou pontos de montagem com **filesystem**, que é o padrão.

## Access Modes

Cada volume possuí três tipos de acesso:

- **ReadWriteOnce** – o volume pode ser montado para leitura e escrita por apenas um node - RWO
- **ReadOnlyMany** – o volume pode ser montado somente para leitura em vários nodes - ROX
- **ReadWriteMany** – o volume pode ser montado para leitura e escrita em vários nodes - RWX

## NFS

Por questões de facilidade, exemplificarei o uso de NFS. Os passos de instalação e criação dos pontos de montagem são executados durante o provisionamento das máquinas pelo Vagrant.
Instalar o pacote **nfs-kernel-server** na máquina que será o nosso storage e criar os diretórios que serão utilizados:

```
apt-get install -y nfs-kernel-server
mkdir -p /volumes/v{1,2,3,4,5}
```

Feito isso, edite o arquivo **/etc/exports** para adicionar os pontos de montagem que serão disponibilizados no cluster:

**/etc/exports**
```
/volumes/v1 192.168.10.0/255.255.255.0(rw,no_root_squash,no_subtree_check)
/volumes/v2 192.168.10.0/255.255.255.0(rw,no_root_squash,no_subtree_check)
/volumes/v3 192.168.10.0/255.255.255.0(rw,no_root_squash,no_subtree_check)
/volumes/v4 192.168.10.0/255.255.255.0(rw,no_root_squash,no_subtree_check)
/volumes/v5 192.168.10.0/255.255.255.0(rw,no_root_squash,no_subtree_check)
```

Execute o comando **exportfs** para habilitar os pontos de montagem:

```
exportfs -a
exportfs
```

Nas máquinas que rodam os pods instalar o **nfs-common** e verificar se é possível montar os diretórios remotamente:

```
apt-get install -y nfs-common
mount -t nfs 192.168.10.40:/volumes/v1 /mnt
echo kubernetes > /mnt/k8s
umount /mnt
```

Verificar na máquina **storage** se o arquivo passou a existir:

```
cat /mnt/k8s
```

Para disponibilizar os volumes para o cluster, precisamos criar um tipo de objeto chamado **PersistentVolume** ou pv:

**nfs-pv.yml**
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-v1
spec:
  capacity:
    storage: 256Mi
  accessModes:
  - ReadWriteMany
  nfs:
    server: 172.27.11.40
    path: "/volumes/v1"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-v2
spec:
  capacity:
    storage: 512Mi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 172.27.11.40
    path: "/volumes/v2"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-v3
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 172.27.11.40
    path: "/volumes/v3"
```

Para que um usuário possa utilizar um destes volumes, é preciso que um **PersistentVolumeClain** seja criado. Desta forma, um volume persistente que atenda as exigências do pedido passará a estar disponível para ser atachado a algum pod:

**nfs-pvc1.yml**
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc1
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 256Mi
```

```
kubectl get pv
kubectl get pvc
```

Para atachar o volume para os pods de um deployment que agora poderão compartilhar arquivos em qualquer lugar do cluster. Veja que o volume faz referência ao **PersistentVolumeClaim** não ao **PersistentVolume**:

**volume-deploy.yml**
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lighttpd
  labels:
    app: lighttpd
spec:
  selector:
    matchLabels:
      app: lighttpd
  replicas: 3
  template:
    metadata:
      labels:
        app: lighttpd
    spec:
      containers:
      - image: hectorvido/sh-cgi
        name: lighttpd
        ports:
        - containerPort: 80
        volumeMounts:
        - name: shared-data
          mountPath: /var/shared-data
      volumes:
      - name: shared-data
        persistentVolumeClaim:
          claimName: nfs-pvc1
```

Além disso, outros pods podem utilizar o volume, sem nenhum problema:

**alpine-pod.yml**
```
apiVersion: v1
kind: Pod
metadata:
  name: alpine
spec:
  containers:
  - image: alpine
    name: alpine
    tty: true
    stdin: true
    volumeMounts:
    - name: shared-data
      mountPath: /var/shared-data
  volumes:
  - name: shared-data
    persistentVolumeClaim:
      claimName: nfs-pvc1
```

# HPA

Com o Horizontal Pod Autoscaler é possível configurar métricas e fazer com que nossos pods cresçam ou diminúam em quantidade conforme a carga sobre eles aumenta.
Para isso é preciso instalar o [metrics-server](https://github.com/kubernetes-incubator/metrics-server/) no cluster:

```
git clone https://github.com/kubernetes-incubator/metrics-server.git
```

Para que as métricas funcionem no cluster gerador pelo **Vagrant** é preciso adicionar e modificar o padrão **command** do container **metric-server**. Um deles especificando o tipo de IP a ser considerado para cada cluster e o outro para aceitar os certificados auto-assinados:

```
vim metrics-server/deploy/1.8+/metrics-server-deployment.yaml
      ...
      containers:
      - name: metrics-server
        image: k8s.gcr.io/metrics-server-amd64:v0.3.1
        imagePullPolicy: Always
        command:
        - /metrics-server
        - --kubelet-preferred-address-types=InternalIP
        - --kubelet-insecure-tls
        volumeMounts:
        - name: tmp-dir
          mountPath: /tmp
```

Feito isso, faça o deploy do metric-server no cluster:

```
kubectl apply -f metrics-server/deploy/1.8+/
```

Espere em torno de 1 minuto para que o pod de métricas consiga se inicializar, e então verifique seu funcionamento consultando as métricas de uso do cluster:

```
kubectl top node
```

Para testar a aplicação, crie um pequeno deployment e um serviço:

```
kubectl run php-apache --image=k8s.gcr.io/hpa-example --requests=cpu=200m --expose --port=80
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
kubectl get hpa
```
