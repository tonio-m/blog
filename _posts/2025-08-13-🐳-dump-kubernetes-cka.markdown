# intro
isso é um dump do que eu aprendi e anotei estudando kubernetes pra CKA.

essas anotações foram feitas pra mim, então se não entendeu algo se vira (ou me pergunta github@mtonio.com).

hoje em dia tudo da pra copiar e colar pra uma LLM, então o importante é manter palavras-chave, explicações de conceitos e edge cases, os especificos 


# core
## kubectl
k get all -A

lembra de usar a API reference ao invés de ficar perdido na documentação deles https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands


## pods 
k get pods --selector env=dev

k run podname --image=x --labels="a=foo,b=bar"

k run podname --image=x --dry-run=client -o=yaml

use k get pod x -o=yaml > pod.yaml pra recriar o pod

k get pods -o=wide (pode ver em que node os pods estão)

você pode especificar **Requests** minimos e **Limits** máximos para os cpu e memória de um pod

quando você coloca --expose --port 80, ele cria um service clusterIp pra você junto com o pod

CrashLoopBackOff num pod pode ser varias coisas, se você der k get pod, dentro do yaml tem a razão pela qual ele deu erro

k logs podname --container=x pra ver os logs do pod

quando um pod é filho de um controller dá pra ver em k describe pod podname \| grep ControlledBy:

volumes hostPath montam a pasta do pod diretamente no host, é muito usado em static pods

imagePullSecrets dentro da spec do pod pra usar um registry do docker privado

k delete pod --force pra não ter que ficar esperando

lembra de olhar os configmaps dos pods (que são montados como volumes), as vezes uma config importante está num configmap do pod e vc não sabe


## sidecar containers e init containers
sidecar containers são tipos especificos de init containers que tem restartPolicy: Always

se um init container tem **restartPolicy: Always**, ele só morre quando o container principal morrer tambem

Init Containers são usados na spec de pods pra criar um container itermediario pra fazer algum build ou setup antes do container principal, da pra dividir um volume entre os dois containers com **volumeMounts** pra eles terem acesso aos mesmos arquivos

init containers rodam um depois do outro, não simultaneamente 

pra ver pq o init container falhou, usa k logs podname -c init-container-name


## nodes
 pods tambem podem rodar no node controlplane
 
você pode forçar um node manualmente num pod com k edit -> nodeName (manual scheduling)

k get nodes -o wide para pegar os ips dos nós

ssh node-ip para dar ssh num nó

k top pods/nodes (precisa do metrics server rodando)

kubectl drain node01 --ignore-daemonsets (adiciona taint de unschedulable pra fazer manutencao e mata todos os pods dentro)

drain não funciona pra dar evict em pods sem controler, pq ele deletaria algo pra sempre

kubectl uncordon node01 (deixa o node schedulable de novo)

use o comando `ip a` pra ver todas as network interfaces 
a interface com (172.0.0.0/16) é a dos containeres
(kube-controller-manager.yaml dentro dos manifests é onde você encontra o range de ips dos pods)

`ip route show default` mostra o gateway padrão do nó (ip do roteador)

containeres tem um ip do roteador virtual que não precisa estar na mesma subnet da rede, se o nó for um container tipo minikube já sabe


## deployments
deployments são controllers feitos só pra pods e replicasets

depois de uma mudança, você tem que dar scale=0 ou deletar os pods manualmente  pro controller recriar

k create deploy --image=x --replicas=3

k scale --replicas=5 rs/new-replica-set

selector labels e template labels no spec de um controller é como o controller sabe de quais pods ele é dono

deployments do tipo RollingUpdate trocam pods um por um sem precisarem ser deletados


## services
dns dos serviços tbm funciona se quebrado em partes (tanto service-name quanto service-name.default.svc.cluster.local) funcionam

o service tipo clusterIP é virtual, expõe uma porta acessado só internamente por outros pods 

o service tipo nodeport expõe uma porta alta em todos nodes, a **nodePort**, mas ele tambem expõe uma porta interna **port** e leva em sua config a **targetPort**, do pod

da pra usar pod: nameselector pra escolher quais pods o service vai servir de proxy


## kube-system, static pods,  daemonsets and etcd
static pod é um pod sem controller definido por padrão em em /etc/kubernetes/manifests/ ()

daemonset é um controller que roda um pod pra cada nó do cluster, e é muito comum em ferramentas de logging, monitoramento e networking

kube-proxy é um daemonset que implementa os services, ele tbm lida com firewalls de services

coredns roda no kube-system e é o que resolve os dns tipo .svc.cluster.local

o coredns é definidos nos pods como um servidor dns mesmo, na porta 53 e tudo mais

se você fizer ps aux \| grep kubelet você consegue ver onde ficam os arquivos de configuração do kubelet, incluindo o --config que é o mais importante

é possivel deployar mais de um scheduler no kubernetes, e as vezes pra setups de infraestrutura muito customizados, ou que sofrem trafego de criacao/destruicao de pods muito alto é o que eles fazem

da pra usar o help dos binarios dos containeres que você não faz ideia do que fazem, tipo k exec -it kube-apiserver-controlplane -n kube-system -- kube-apiserver --help

crictl ps -a (é tipo docker ps -a mas para os containeres que implementam o kubernetes)

crictl logs container-id

etcd node roda como pod mas você pode usar etcdctl no controlplane pra usar ele

etcd guarda todas as resource definitions e metadata de pods rodando

etc é um static pod em /etc/kubernetes/manifests/etcd.yaml, tipo o kube-apiserver

pra fazer um backup do etcd:
```
etcdctl snapshot save --endpoints --cacert --cert --key  /opt/snapshot-pre-boot.db
etcdctl snapshot restore /opt/snapshot-pre-boot.db data-dir/
rm -rf /var/lib/etcd/ &&  mv data-dir /var/lib/etcd
```

o port do etcd 2379 é pra comunicação controlplane -> node

o port etcd 2380 é pra comunicação controlplane <-> controlplane (peer to peer)


## dicas para debugging
se for application failure: olhe a config do service e do pod

se for failure no controlplane: olha os pods do kube-system e o /etc/kubernetes/manifests

se for failure no worker node: olha o systemctl status kubelet, a /var/lib/kubelet/config.yaml, e o /etc/kubernetes/kubelet.conf (acho que essas coisas estão no comando de instalação do kubelet ou no service)

se for network failure: olha se o CNI tá instalado, olha os pods do CNI (eles ficam em outros namespaces normalmente)


## pvs, pvc
persistent volumes são como definições de espaço a ser usado em disco, tipo definições de uma pasta

da pra cirar um persistentvolume de tipo hostPath, que monta direto numa pasta do host

volumes com tipo readwritemany podem ser montados em varios nodes, readwriteonce só em um

se vc usar a config do pv retain ele fica unavailable depois de usado pra evitar que ele autodelete quando outro pod quiser usar ele

persistent volume claims são "escrituras/reservas" de persistent volume

pvc é como os pods usam o storage do pv, você referencia a pvc no spec

enquanto o pvc tiver criado, os dados vão continuar lá, reservados

1 pvc = 1 pv, sempre são de um pra um, nunca da pra fazer varios pvcs pra 1 pv, são monogamicos

PVCs com storageclass especificada não precisam de nenhum PV pré-criado, criam automaticamente

**storageclassname** é especificado na spec do pvc

kubectl describe pvc pvcname


# pod management
## taints, tolerations
taints afastam pods que não tenha toleration

tolerations toleram pods com taints especificas

taints normalmente são um key-value pair

 k taint nodes node1 afasta=tudo:NoSchedule
 

## node affinity
node affinity é uma config do pod pra preferir/requisitar nodes com certas labels

k label node node01 gpu=True

o node affinity é editado no spec do deployment ou do pode

node affinity pode ser required ou preferred, with conditions key has value x  or key exists


## priorityclasses
k get priorityclasses

k create priorityclass low-priority --value=1000 --description="bla" --global-default=false

priorityclasses são classes que fazerm um pod ter prioridade sobre outros pods

**priorityClassName** pode ser especificado no spec de pods

a priorityclass do tipo PreemptLowerPriority dá evict em outros pods, a Never não


## configmaps e secrets
configmap é tipo um secret só que sem criptografia

dá pra usar os valores do configmap em environment variables 

o único outro jeito de usar é em volumes (ai cada par key-value vira um arquivo no volume)

tem muitas vezes onde secrets são do tipo TLS, onde tem arquivos .crt, .key and .ca 

tem secret do tipo docker-registry tambem, pra registries privados do docker


## Horizontal Pod autoscaler e Vertical Pod autoscaller
HPA precisa de requests / limits especificados no pod, pq ele calcula o threshold pra criar uma nova replica baseado nesses requests

VPA é um custom resource definition que vc precisa instalar, ele deleta o pod e cria de novo com mais memoria/cpu 

o VPA usa machine learning pra dar update em limites de CPU e memoria

o VPA da erro quando tem só uma replica do que você tá autoscaling, pq o serviço ficaria indisponivel na recriação


# security settings
## users and certificates
o jeito mais comum sem um provider externo de criar um user no kubernetes é criado um certificado e uma clusterrole

certificado tls é composto por .crt, .key and .ca 

.ca pode ser compartilhado as vezes, as vezes não, pq certificados TLS conseguem fazer uma chain de pai CAzão pra filho CAzinho pra filho CAzito

quando tu usa o comando do openssl x509, vem varios campos

nesses campos Subject CN: dentro do .crt é o CommonName, mostra o que ele representa, tipo o username dele

o campo Issuer CN mostra o pai dele, o CAzão

o Subject Alternative Name: tem todos os DNSs onde o certificado é usado

certificatesigningrequest é um recurso que é usado pra pedir aprovação de certificados TLS pra usuários da API

k get csr

k certificate approve fulano


## RBAC, roles e clusterroles
a diferença de roles e clusterroles é que as clusterroles não se restringem por namespace

uma rolebinding binda uma role com permissões em um ou vários subjects, mas ela não consegue usar mais de uma role ao mesmo tempo não

dar acesso pra alguem é: criar role com permissão x + criar rolebinding dessa role no user

nas roles, os "apigroups" na spec são o que fica em cima da spec de todos recursos (o api:)

nas roles, resourceNames é tipo prefixo da role do IAM da aws, e.g.: quero permissão x pra todos pods com prefixo bonito-*

cluster roles são pra coisas tipo permissões de usuário ou pra dar permissão pra recursos sem namespace como nodes ou storageclasses


## admission controllers
admission controllers podem ter o tipo mutating ou validating, 

admission controllers são hookzinhos validadores de template que rodam depois que tu dá o comando de criar o recurso, mas antes dele serem efetivamente criados

exemplos de admission controller validating  NamespaceExists (quando tu cria e ele da erro, valida se ja existe)

exemplos de admission controlller mutating NamespaceAutoProvision (quando tu da k create namespace e ele nao da erro)

tem recursos builtin pra fazer admission controllers customizados com webhooks


## serviceaccounts
serviceaccounts são usados no campo serviceAccountName: dos pods pra dar direitos de kubectl pra eles

k create serviceaccount saname

k create token saname


# networking
## networkpolicies
é o firewall do kubernetes basicamente, tem ingress egress allow deny

o deny é meio que por padrao, se vc criar uma netpol e não botar nenhuma regra é tipo um deny all

k get netpol

da pra usar pod: nameselector na network policy e filtrar por pods

essa porra é mto complexa então usa essas receitas aqui https://github.com/ahmetb/kubernetes-network-policy-recipes


## cni
container network interface é quem decide como os ips entre os pods vão funcionar

eles são como se fossem plugins e você escolhe qual vai usar 

flannel nao tem suporte pra networkpolicy

calico tem suporte pra networkpolicy
 
plugins cni são instalados em /etc/cni/net.d/

aliás, paths com unix:// paths são sockets unix (que são diferentes de sockets de rede)


## ingress e gateway
um ingress no k8s é tipo um nginx, ele faz port forwarding usando http e regras de dns

vc tem que instalar um ingress especifico pra usar, no meu caso foi o nginx-ingress

o ingress associa um subdomínio tal com um service x e manda toda requisição q vc faz no subdominio pro service, você até associa um service default em caso de 404 no deployment do ingress

no ingress resourceannotation no metadata costuma importar, pelo menos no nginx-ingress entao toma cuidado

gateway é tipo um ingress só que tunado e muito mais complexo

no gateway as gateway routes definem quais namespaces podem usar o gateway, 

a gatewayclass define um tipo de gateway, e tem o  gateway que é a implementacao do gateway per se

no gateway tem o httproute que é um resource dedicado só pra uma rota subdominio -> service


# cluster
## cluster installation
add repo to apt repositories

`apt install kubeadm='1.33.0-*' kubelet='1.33.0-*' kubectl='1.33.0-*'`

no controlplane você dá kubeadm init

no worker você dá kubeadm join

depois você instala uma CNI, tipo o flannel


## upgrade de cluster
em cada upgrade de 1.32 -> 1.33 você tem que mudar o repositorio do APT, pra fazer upgrade do kubeadm kubectl e kubelet

em cada node:
```
apt install kubeadm='1.33.0-*' kubelet='1.33.0-*' kubectl='1.33.0-*'
kubeadm upgrade plan 
kubeadm upgrade apply v1.33.0
kubeadm upgrade node

systemctl daemon-reload
systemctl restart kubelet
```


## kubeconfig
k config use-context research

a environment variable  KUBECONFIG pode ser usada pra especificar o diretorio da kubeconfig tbm

k get pods --as my-context pra usar outro contexto programaticamente


# plugins and exstensions
## custom resource definitions
custom resource definitions sao bem faceis de entender

vc define um plural e um grupo (que é  o que fica na api version la em cima)


## helm
comandos de repos

helm search hub

helm repo add

helm install releasename repo/chart

helm upgrade releasename repo/chart --version 2.0.0

helm get manifest releasename

helm rollback releasename


## kustomize:
touch ./kustomization.yaml && k apply -k .

o nome é kustomize pq vc pode pegar yamls prontos e dar override em valores deles num lugar central com mta facilidade, usando vários kustomize.yaml, um em cada folder

kustomize tem configs como dar overwrite em tudo com uma commonLabel, commonAnnotation, commonNamespace, e prefixos de nomes em comum tambem

no kustomize você exclui/inclui recursos separando eles em diretorios e referenciando eles no kustomize.yaml

kustomize tem os tipos de patch. change patch, $patch: delete e JSON 6902 patch

change patch exemplo: images: name: newName: pra trocar o nome da img, newTag pra trocar a tag

$patch: delete consegue deletar containeres de um pod

patch JSON 6902 permite trocar qualquer config de qualquer recurso

k patch é um comando default do kubernetes que edita recursos com um json

kustomize tem o conceito de bases e overlays

bases você importa

overlays são como dev/prod/staging, e tem patches e components

