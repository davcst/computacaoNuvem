# Atividade Prática

---

# Parte 1 — Empacotar e publicar a aplicação no Amazon ECR

---

## Passo 1 — Criar o repositório no Amazon ECR

É no ECR que as imagens das suas aplicações são enviadas.

```bash
aws ecr create-repository \
  --repository-name app-cn-2026 \
  --region us-east-1
```

O retorno deve mostrar um bloco `repository` com o `repositoryUri` (algo como `SEU_ID.dkr.ecr.us-east-1.amazonaws.com/app-cn-2026`).

> Ou, pelo Amazon ECR → Repositories → Create repository → em Visibility settings deixe Private → em Repository name digite `app-cn-2026` → Create repository. A tela seguinte já mostra a URI do repositório recém-criado — útil para comparar com o valor retornado pelo CLI.

---

## Passo 2 — Criar os arquivos da aplicação

Criação do código (`server.js`), das dependências (`package.json`) e da receita da imagem (`Dockerfile`).

```bash
mkdir app-cn-2026 && cd app-cn-2026

cat <<EOF > package.json
{
  "name": "app-cn-2026",
  "version": "1.0.0",
  "main": "server.js",
  "dependencies": {
    "express": "^4.18.2"
  }
}
EOF

cat <<EOF > server.js
const express = require('express');
const app = express();
app.get('/', (req, res) => res.send('Aplicacao rodando em uma instancia EC2!'));
app.listen(8080, () => console.log('Servidor ativo na porta 8080'));
EOF

cat <<EOF > Dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package.json .
RUN npm install
COPY . .
EXPOSE 8080
CMD ["node", "server.js"]
EOF
```

Rode `ls -la`, devem aparecer os três arquivos (`package.json`, `server.js`, `Dockerfile`) na pasta `app-cn-2026`.

---

## Passo 3 — Autenticar o Docker no ECR

O Docker precisa de uma credencial temporária para poder enviar imagens ao seu repositório privado.

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --region us-east-1 --query Account --output text)

aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com
```

A saída deve mostrar `Login Succeeded`.

---

## Passo 4 — Gerar a imagem Docker (build)

Com os arquivos prontos, é hora de compilar a imagem, já apontando para o endereço do seu repositório no ECR.

```bash
docker build -t $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/app-cn-2026:latest .
```

O log deve terminar com `naming to ...app-cn-2026:latest`, sem nenhuma linha `ERROR`.

---

## Passo 5 — Enviar (push) a imagem para o ECR

Agora a imagem gerada localmente no CloudShell é enviada para o repositório na nuvem.

```bash
docker push $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/app-cn-2026:latest
```

A última linha deve mostrar `latest: digest: sha256:...`, sinal de que o envio terminou com sucesso.

---

# Parte 2 — Provisionar a instância EC2 que vai rodar a aplicação

> Os Passos 6 a 11 podem ser feitos tanto pelo CloudShell quanto pelo console EC2. Abaixo tem a opção pelo console, e em seguida pelo CloudShell.

> Alternativa pelo Console (Passos 6 a 11):
> 1. Acesse EC2 → Launch instance.
> 2. Em Name, digite `app-cn-2026`.
> 3. Em Application and OS Images (AMI), escolha a aba Quick Start → Ubuntu e selecione Ubuntu Server 22.04 LTS.
> 4. Em Instance type, selecione `t3.micro`.
> 5. Em Key pair, selecione Proceed without a key pair. Reforça que o acesso será só via Session Manager, sem SSH.
> 6. Em Network settings, clique em Edit: confirme a VPC padrão e a subnet; em Firewall (security groups) escolha Create security group dê um nome (`app-cn-2026-sg`), remova a regra de SSH sugerida por padrão e adicione uma regra: Type = Custom TCP, Port range = `8080`, Source = Anywhere (`0.0.0.0/0`).
> 7. Abra Advanced details: em IAM instance profile, selecione LabInstanceProfile.
> 8. Ainda em Advanced details, role até User data e cole o conteúdo do script gerado no Passo 10 (`cat user-data.sh` no CloudShell).
> 9. Clique em Launch instance.


## Passo 6 — Obter a AMI Ubuntu

Uma instância EC2 precisa de uma AMI, que define o sistema operacional.

```bash
UBUNTU_AMI_ID=$(aws ssm get-parameters \
  --names /aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ebs-gp3/ami-id \
  --region us-east-1 \
  --query "Parameters[0].Value" --output text)

echo "AMI Ubuntu: $UBUNTU_AMI_ID"
```

Deve retornar um ID no formato `ami-0xxxxxxxxxxxxxxxx`.

---

## Passo 7 — Descobrir a VPC e a Subnet padrão

A instância precisa nascer em alguma rede. Vamos usar a VPC padrão da conta, que já vem pronta no laboratório.

```bash
VPC_ID=$(aws ec2 describe-vpcs --region us-east-1 --filters "Name=isDefault,Values=true" --query "Vpcs[0].VpcId" --output text)
SUBNET_ID=$(aws ec2 describe-subnets --region us-east-1 --filters "Name=vpc-id,Values=$VPC_ID" --query "Subnets[0].SubnetId" --output text)

echo "VPC: $VPC_ID | Subnet: $SUBNET_ID"
```

Ambos os valores devem aparecer preenchidos (formato `vpc-...` e `subnet-...`).

---

## Passo 8 — Criar o Security Group

É preciso liberar apenas a porta usada pela aplicação.

```bash
SG_ID=$(aws ec2 create-security-group \
  --region us-east-1 \
  --group-name app-cn-2026-sg \
  --description "Libera porta 8080 para a aplicacao" \
  --vpc-id $VPC_ID \
  --query "GroupId" --output text)

aws ec2 authorize-security-group-ingress \
  --region us-east-1 \
  --group-id $SG_ID --protocol tcp --port 8080 --cidr 0.0.0.0/0

echo "Security Group: $SG_ID"
```

Deve mostrar um ID no formato `sg-xxxxxxxxxxxxxxxxx`.

---

## Passo 9 — Confirmar o Instance Profile do laboratório

Em contas AWS Academy, criar uma IAM Role nova costuma ser bloqueado (falta a permissão `iam:CreateRole`). A solução é reaproveitar o instance profile que o próprio laboratório já disponibiliza, vinculado à `LabRole`, já possui permissão para acessar o ECR.

```bash
aws iam list-instance-profiles --query "InstanceProfiles[].InstanceProfileName" --output text
```

Deve aparecer `LabInstanceProfile` (ou nome equivalente) na lista. Esse nome que vai ser usado no passo 11.

---

## Passo 10 — Preparar o script de inicialização (User Data)

Este script roda sozinho assim que a instância liga: instala o Docker, instala a AWS CLI, autentica no ECR e sobe automaticamente a imagem publicada na Parte 1. Não é preciso entrar na instância manualmente para isso funcionar.

```bash
cat <<EOF > user-data.sh
#!/bin/bash
apt-get update -y
apt-get install -y docker.io unzip curl
systemctl enable docker
systemctl start docker

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install

aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com

docker pull $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/app-cn-2026:latest

docker run -d --restart always \
  -p 8080:8080 \
  $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/app-cn-2026:latest
EOF
```

`cat user-data.sh` deve mostrar o script completo, já com o ID da sua conta preenchido e a região `us-east-1` fixa nos comandos de `docker login` e `docker pull`.

---

## Passo 11 — Lançar a instância EC2 com a AMI Ubuntu

Agora todos os elementos (AMI, subnet, security group, instance profile e o script de inicialização) se combinam em um único comando.

```bash
INSTANCE_ID=$(aws ec2 run-instances \
  --region us-east-1 \
  --image-id $UBUNTU_AMI_ID \
  --instance-type t3.micro \
  --subnet-id $SUBNET_ID \
  --security-group-ids $SG_ID \
  --iam-instance-profile Name=LabInstanceProfile \
  --user-data file://user-data.sh \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=app-cn-2026}]' \
  --query "Instances[0].InstanceId" --output text)

echo "Instancia criada: $INSTANCE_ID"
```

Deve retornar um ID no formato `i-xxxxxxxxxxxxxxxxx`.

---

## Passo 12 — Aguardar a instância subir e obter o IP público

```bash
aws ec2 wait instance-running --region us-east-1 --instance-ids $INSTANCE_ID

PUBLIC_IP=$(aws ec2 describe-instances \
  --region us-east-1 \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)

echo "IP publico: $PUBLIC_IP"
```

Deve retornar um endereço IP válido (ex.: `52.xxx.xxx.xxx`).

> Ou vá em instâncias, clique na instância `app-cn-2026` e copie o Public IPv4 address exibido no painel de detalhes.

---

## Passo 13 — Testar o acesso pelo próprio CloudShell

O Docker leva alguns segundos para baixar a imagem e subir o contêiner depois do boot.

```bash
curl http://$PUBLIC_IP:8080
```

Deve retornar o texto `Aplicacao rodando em uma instancia EC2!`.

---

## Passo 14 — Testar pelo navegador

Abra no navegador: `http://<IP_PUBLICO>:8080`

A página carrega o texto da aplicação sem exibir nenhuma tela de bloqueio.

---

## Passo 15 — Acessar a instância via Session Manager - Opcional

Se a aplicação não subir corretamente, dá pra entrar na instância pelo Session Manager e checar os logs do Docker.

```bash
aws ssm start-session --region us-east-1 --target $INSTANCE_ID
```

Dentro da sessão:

```bash
sudo docker ps
sudo docker logs $(sudo docker ps -q)
```

`docker ps` deve mostrar o contêiner com status `Up`; os logs devem mostrar `Servidor ativo na porta 8080`.

---

## Passo 16 — Atualizar a aplicação após um novo `docker push`

Se alguém alterar o código, gerar uma nova imagem (passos 5 e 6) e der `docker push`, a instância não atualiza sozinha. É preciso mandar ela baixar a nova versão e reiniciar o contêiner.

```bash
aws ssm send-command \
  --region us-east-1 \
  --instance-ids $INSTANCE_ID \
  --document-name "AWS-RunShellScript" \
  --parameters commands="[
    \"aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com\",
    \"docker pull $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/app-cn-2026:latest\",
    \"docker stop \$(docker ps -q) || true\",
    \"docker run -d --restart always -p 8080:8080 $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/app-cn-2026:latest\"
  ]"
```

**Verificação:**
```bash
aws ssm list-command-invocations --region us-east-1 --instance-id $INSTANCE_ID --details --query "CommandInvocations[0].Status"
```
deve retornar `"Success"`.

---

## Passo 17 — Encerrar e limpar os recursos

Para não gerar custo contínuo, encerre a instância e remova o security group e o repositório ECR criados nesta atividade.

```bash
aws ec2 terminate-instances --region us-east-1 --instance-ids $INSTANCE_ID
aws ec2 wait instance-terminated --region us-east-1 --instance-ids $INSTANCE_ID
aws ec2 delete-security-group --region us-east-1 --group-id $SG_ID
aws ecr delete-repository --region us-east-1 --repository-name app-cn-2026 --force
```

**Verificação:**
```bash
aws ec2 describe-instances --region us-east-1 --instance-ids $INSTANCE_ID --query "Reservations[0].Instances[0].State.Name"
```
deve retornar `"terminated"`.

---
