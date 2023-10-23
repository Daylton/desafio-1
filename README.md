# desafio-1

# 1 - Definir as variáveis. OBS: Pegar as informações no próprio lab do desafio

````
export ZONE=
````

````
export INSTANCE_NAME=
````

````
export PORT=
````
````
export FIREWALL_NAME=
````

# 2 - Definir a zona
````
export REGION=${ZONE::-2}
````

# Tarefa 1
# 3 - Criar a instância
````

gcloud compute instances create $INSTANCE_NAME \
--zone $ZONE \
--machine-type e2-micro \
--network nucleus-vpc \
--image-family debian-10 \
--image-project debian-cloud

# Tarefa 2
# 4 - Criar os clusters

gcloud container clusters create nucleus-backend \
--num-nodes 1 \
--network nucleus-vpc \
--zone $ZONE

# 5 - Fazer deploy da aplicação

kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:2.0

# 6 - Expor a aplicação

kubectl expose deployment hello-server \
--type=LoadBalancer \
--port $PORT

# Tarefa 3
# 7 - Executar esse comando

cat > startup.sh << EOF
#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOF

# 8 - Criar web-server-template
gcloud compute instance-templates create web-server-template --region $REGION \
--metadata-from-file startup-script=startup.sh \
--network nucleus-vpc \
--machine-type e2-micro

# 9 - Criar o nginx-poll
gcloud compute target-pools create nginx-pool --region=$REGION

# 10 - Criar o web-server-group
gcloud compute instance-groups managed create web-server-group --region $REGION \
--base-instance-name web-server \
--size 2 \
--template web-server-template

# 11 - Criar as regras de firewall
gcloud compute firewall-rules create $FIREWALL_NAME \
--allow tcp:80 \
--network nucleus-vpc

# 12 - Criar os helth-checks
gcloud compute http-health-checks create http-basic-check

# 13 - Instancia de grupo
gcloud compute instance-groups managed \
set-named-ports web-server-group \
--named-ports http:80 \
--region $REGION

# 14 - Backend service
gcloud compute backend-services create web-server-backend \
--protocol HTTP \
--http-health-checks http-basic-check \
--global
gcloud compute backend-services add-backend web-server-backend \
--instance-group web-server-group \
--instance-group-region $REGION \
--global

# 15 - Web-server-map
gcloud compute url-maps create web-server-map \
--default-service web-server-backend
gcloud compute target-http-proxies create http-lb-proxy \
--url-map web-server-map
gcloud compute forwarding-rules create http-content-rule \
--global \
--target-http-proxy http-lb-proxy \
--ports 80

# 16 - Roteamento do das regras de firewall
gcloud compute forwarding-rules create $FIREWALL_NAME \
--global \
--target-http-proxy http-lb-proxy \
--ports 80

````
