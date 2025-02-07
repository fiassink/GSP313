## SETANDO PROJETO ##
>Comando para setar projeto no Gcloud

`gcloud config set project` **ID DO SEU PROJETO**

## CREATE A JUMPHOST INSTANCE ##
>Definir no cloudshell: nome, vpc, zona, tipo, imagem e projeto:

`gcloud compute instances create` **NOME DO SEU JUMPHOST** `--network nucleus-vpc --zone us-east1-b --machine-type f1-micro --image-family debian-9 --image-project debian-cloud`

## CREATE A KUBERNETES SERVICE CLUSTER ##
>Definir no cloudhsell: nome, numero de nós, vpc e região do cluster

`gcloud container clusters create nucleus-backend --num-nodes 1 --network nucleus-vpc --region us-east1`
>Comando para credenciamento requer: nome e região do cluster

`gcloud container clusters get-credentials nucleus-backend --region us-east1`
>Comando para criar deploy do cluster requer: nome do cluster e url do app

`kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:2.0`
>Comando para expor cluster requer: nome do cluster, tipo e porta

`kubectl expose deployment hello-server --type=LoadBalancer --port` **NÚMERO DA SUA PORTA**

## CREATE A WEBSERVER FRONTEND ##
>Utilize o código abaixo para configurar os webservers

`cat << EOF > startup.sh
#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOF`

>Definir no cloudshell: template, target pool, grupo de instancia, firewall rule para permitir tráfego na porta 80/tcp, health check, backend service, url map

>Criando Instance Template: nome, arquivo com metadados, vpc, tipo da instancia, região

`gcloud compute instance-templates create web-server-template --metadata-from-file startup-script=startup.sh --network nucleus-vpc --machine-type f1-micro --region us-east1`

>Criando Instance Group: nome, instancia de referencia, tamanho, template de referencia, região

`gcloud compute instance-groups managed create web-server-group --base-instance-name` **NOME DO SEU JUMPHOST**`--size 2 --template web-server-template --region us-east1`


>Criando Firewall rules: nome, liberar porta, vpc

`gcloud compute firewall-rules create` **NOME DA SUA FIREWALL RULE** --allow tcp:80 --network nucleus-vpc`

>Criando Health Checks: nome

`gcloud compute http-health-checks create http-basic-check`

>Attachando instance group com porta 80: grupo de instancias, porta, região

`gcloud compute instance-groups managed set-named-ports web-server-group --named-ports http:80 --region us-east1`

>Criando backend e setando no instance group: nome, ptorocolo, health check, grupo de instancia, região

`gcloud compute backend-services create web-server-backend --protocol HTTP --http-health-checks http-basic-check --global`

`gcloud compute backend-services add-backend web-server-backend --instance-group web-server-group --instance-group-region us-east1 --global`

>Criando URL map: nome, backend de referencia

`gcloud compute url-maps create web-server-map --default-service web-server-backend`

>Setando target do HTTP proxy para rotear requests para URL map: nome, url map de referencia

`gcloud compute target-http-proxies create http-lb-proxy --url-map web-server-map`

>Criando forwarding rule: nome, setando como global, http proxy de referencia, porta

`gcloud compute forwarding-rules create http-content-rule --global --target-http-proxy http-lb-proxy --ports 80`

>Visualizando lista de forwarding rules: nome das regras

`gcloud compute forwarding-rules list`
