# Atividade Prática — Deploy em EC2 (Ubuntu) com AWS CloudFormation


# Parte 1 — Repositório ECR (via CloudFormation) e imagem Docker

Usaremos **duas stacks**: a primeira cria o repositório ECR; a segunda cria a instância. A imagem precisa estar publicada **antes** da segunda stack, porque a instância baixa a imagem assim que liga.

## Passo 1 — Criar o template do repositório ECR

Este template tem só um recurso (o repositório) e uma saída (a URI dele). O `EmptyOnDelete` faz o repositório ser apagado junto com a stack, mesmo contendo imagens.

```bash
cd ~
cat <<'EOF' > ecr.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Repositorio ECR da atividade pratica

Parameters:
  RepositoryName:
    Type: String
    Default: app-cn-2026
    Description: Nome do repositorio ECR

Resources:
  AppRepository:
    Type: AWS::ECR::Repository
    Properties:
      RepositoryName: !Ref RepositoryName
      EmptyOnDelete: true

Outputs:
  RepositoryUri:
    Description: URI do repositorio
    Value: !GetAtt AppRepository.RepositoryUri
EOF
```

Rode `ls -la ecr.yaml`, o arquivo deve aparecer na lista.

---

## Passo 2 — Criar a stack do ECR

O comando `deploy` valida o template, cria a stack e espera terminar. Se a stack já existir, o mesmo comando faz uma atualização.

```bash
aws cloudformation deploy \
  --region us-east-1 \
  --template-file ecr.yaml \
  --stack-name app-cn-2026-ecr
```

A saída deve terminar com `Successfully created/updated stack - app-cn-2026-ecr`. Para ver a URI do repositório:

```bash
aws cloudformation describe-stacks \
  --region us-east-1 \
  --stack-name app-cn-2026-ecr \
  --query "Stacks[0].Outputs" --output table
```

> Ou, pelo Console: no CloudShell, clique em Actions → Download file e informe `/home/cloudshell-user/ecr.yaml`. Depois vá em CloudFormation → Stacks → Create stack → With new resources (standard) → Template is ready → Upload a template file → escolha o `ecr.yaml` → Next → em Stack name digite `app-cn-2026-ecr` → Next → Next → Submit. Acompanhe a aba Events até o status `CREATE_COMPLETE` e veja a URI na aba Outputs.

---

## Passo 3 — Criar os arquivos da aplicação

Criação do código (`server.js`), das dependências (`package.json`) e da receita da imagem (`Dockerfile`), já na porta 8080.

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

## Passo 4 — Autenticar o Docker no ECR

O Docker precisa de uma credencial temporária para enviar imagens ao seu repositório privado.

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --region us-east-1 --query Account --output text)

aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com
```

A saída deve mostrar `Login Succeeded`.

---

## Passo 5 — Gerar a imagem Docker (build)

```bash
docker build -t $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/app-cn-2026:latest .
```

O log deve terminar com `naming to ...app-cn-2026:latest`, sem nenhuma linha `ERROR`.

---

## Passo 6 — Enviar (push) a imagem para o ECR

```bash
docker push $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/app-cn-2026:latest
```

A última linha deve mostrar `latest: digest: sha256:...`, sinal de que o envio terminou com sucesso.

---

# Parte 2 — Instância EC2 (Ubuntu) via CloudFormation

## Passo 7 — Criar o template da instância

Este template substitui, de uma vez, os passos manuais da versão anterior (descobrir AMI, criar security group, lançar instância, montar user data):

- **Parameters:** tipo da instância, nome do instance profile, nome do repositório e a AMI Ubuntu (resolvida automaticamente pelo SSM Parameter Store).
- **Resources:** o security group (só a porta 8080, nenhuma porta SSH) e a instância EC2.
- **Outputs:** ID da instância, IP público e a URL da aplicação.

A conta e a região entram no script automaticamente pelas variáveis do CloudFormation (`${AWS::AccountId}` e `${AWS::Region}`). Por isso o heredoc usa `'EOF'` entre aspas, para o CloudShell não tentar expandir essas variáveis.

```bash
cd ~
cat <<'EOF' > ec2.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Instancia EC2 Ubuntu que roda a imagem Docker do ECR na porta 8080

Parameters:
  RepositoryName:
    Type: String
    Default: app-cn-2026
    Description: Nome do repositorio ECR onde a imagem foi publicada
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues:
      - t3.micro
      - t3.small
    Description: Tipo da instancia EC2
  InstanceProfileName:
    Type: String
    Default: LabInstanceProfile
    Description: Instance profile que ja existe no laboratorio (nao e criado aqui)
  UbuntuAmiId:
    Type: 'AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>'
    Default: /aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ebs-gp2/ami-id
    Description: AMI Ubuntu 22.04 mais recente, resolvida pelo SSM Parameter Store

Resources:
  AppSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Libera apenas a porta 8080 para a aplicacao
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 8080
          ToPort: 8080
          CidrIp: 0.0.0.0/0

  AppInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref UbuntuAmiId
      InstanceType: !Ref InstanceType
      IamInstanceProfile: !Ref InstanceProfileName
      SecurityGroupIds:
        - !GetAtt AppSecurityGroup.GroupId
      Tags:
        - Key: Name
          Value: app-cn-2026
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          apt-get update -y
          apt-get install -y docker.io unzip curl
          systemctl enable docker
          systemctl start docker
          cd /tmp
          curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
          unzip -q awscliv2.zip
          ./aws/install
          /usr/local/bin/aws ecr get-login-password --region ${AWS::Region} | docker login --username AWS --password-stdin ${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com
          docker pull ${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/${RepositoryName}:latest
          docker run -d --restart always -p 8080:8080 ${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/${RepositoryName}:latest

Outputs:
  InstanceId:
    Description: ID da instancia
    Value: !Ref AppInstance
  PublicIp:
    Description: IP publico da instancia
    Value: !GetAtt AppInstance.PublicIp
  AppUrl:
    Description: URL da aplicacao
    Value: !Sub 'http://${AppInstance.PublicIp}:8080'
EOF
```

Rode `ls -la ec2.yaml`, o arquivo deve aparecer na lista.

> O template não define VPC nem subnet: o security group e a instância nascem na VPC padrão da conta, que já existe no laboratório.

---

## Passo 8 — Validar o template

Antes de criar qualquer recurso, o CloudFormation confere se o template está bem formado.

```bash
aws cloudformation validate-template \
  --region us-east-1 \
  --template-body file://ec2.yaml
```

A saída deve listar os quatro parâmetros (`RepositoryName`, `InstanceType`, `InstanceProfileName`, `UbuntuAmiId`). Se aparecer erro, ele indica a linha do problema.

---

## Passo 9 — Criar a stack da instância

```bash
aws cloudformation deploy \
  --region us-east-1 \
  --template-file ec2.yaml \
  --stack-name app-cn-2026-ec2
```

O comando fica aguardando por 1 a 3 minutos e termina com `Successfully created/updated stack - app-cn-2026-ec2`.

> Ou, pelo Console: repita o fluxo do Passo 2 com o arquivo `ec2.yaml` (baixe com Actions → Download file, caminho `/home/cloudshell-user/ec2.yaml`) e o nome de stack `app-cn-2026-ec2`. A tela de parâmetros mostra os quatro parâmetros do template com seus valores padrão; avance até Submit e acompanhe a aba Events.

> Se a stack terminar em `ROLLBACK_COMPLETE`, abra a aba Events (ou rode `aws cloudformation describe-stack-events --region us-east-1 --stack-name app-cn-2026-ec2`) e leia o motivo na coluna Status reason. Depois apague a stack com `aws cloudformation delete-stack --region us-east-1 --stack-name app-cn-2026-ec2` e rode o `deploy` de novo.

---

## Passo 10 — Consultar os recursos e as saídas (Outputs)

Veja o que o CloudFormation criou e leia o IP público direto das saídas da stack.

```bash
aws cloudformation describe-stack-resources \
  --region us-east-1 \
  --stack-name app-cn-2026-ec2 \
  --query "StackResources[].[LogicalResourceId,ResourceType,ResourceStatus]" --output table

INSTANCE_ID=$(aws cloudformation describe-stacks --region us-east-1 --stack-name app-cn-2026-ec2 \
  --query "Stacks[0].Outputs[?OutputKey=='InstanceId'].OutputValue" --output text)

PUBLIC_IP=$(aws cloudformation describe-stacks --region us-east-1 --stack-name app-cn-2026-ec2 \
  --query "Stacks[0].Outputs[?OutputKey=='PublicIp'].OutputValue" --output text)

echo "Instancia: $INSTANCE_ID | IP publico: $PUBLIC_IP"
```

A tabela deve mostrar `AppSecurityGroup` e `AppInstance` com status `CREATE_COMPLETE`, e o `echo` deve mostrar um ID `i-...` e um IP válido.

> Ou vá em CloudFormation → Stacks → `app-cn-2026-ec2` e navegue pelas abas: Resources (recursos criados), Outputs (IP e URL), Parameters (valores usados), Events (histórico) e Template (o YAML aplicado).

---

## Passo 11 — Testar a aplicação

O Docker leva alguns segundos para baixar a imagem e subir o contêiner depois do boot. Se não responder de imediato, tente de novo após alguns segundos.

```bash
curl http://$PUBLIC_IP:8080
```

Deve retornar o texto `Aplicacao rodando em uma instancia EC2!`. Depois, abra no navegador a URL que aparece na saída `AppUrl` (`http://<IP_PUBLICO>:8080`); a página deve carregar o mesmo texto.

---

## Passo 12 — Acessar a instância via Session Manager - Opcional

Se a aplicação não subir, dá pra entrar na instância pelo Session Manager e checar os logs do Docker.

```bash
aws ssm start-session --region us-east-1 --target $INSTANCE_ID
```

Dentro da sessão:

```bash
sudo docker ps
sudo docker logs $(sudo docker ps -q)
```

`docker ps` deve mostrar o contêiner com status `Up`; os logs devem mostrar `Servidor ativo na porta 8080`.

> Ou, pelo Console: EC2 → Instances → selecione `app-cn-2026` → Connect → aba Session Manager → Connect. Abre um terminal no navegador, sem porta SSH.

---

## Passo 13 — Atualizar a stack (update)

Vamos trocar o tipo da instância de `t3.micro` para `t3.small` **sem mexer no template**, só alterando o parâmetro. O CloudFormation calcula a diferença e altera apenas a instância; o security group fica intacto.

```bash
aws cloudformation deploy \
  --region us-east-1 \
  --template-file ec2.yaml \
  --stack-name app-cn-2026-ec2 \
  --parameter-overrides InstanceType=t3.small
```

O comando termina com `Successfully created/updated stack - app-cn-2026-ec2`. Para confirmar o novo tipo:

```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].InstanceType" --output text
```

Deve retornar `t3.small`.

> Para trocar o tipo, o CloudFormation precisa parar e iniciar a instância. Como ela não tem IP fixo, **o IP público muda**. Releia as saídas e teste de novo:
>
> ```bash
> PUBLIC_IP=$(aws cloudformation describe-stacks --region us-east-1 --stack-name app-cn-2026-ec2 \
>   --query "Stacks[0].Outputs[?OutputKey=='PublicIp'].OutputValue" --output text)
> curl http://$PUBLIC_IP:8080
> ```
>
> A aplicação volta sozinha porque o contêiner foi criado com `--restart always`.

> Para ver o que *vai* mudar antes de aplicar, adicione `--no-execute-changeset` ao comando `deploy`. Ele cria o *change set* sem executá-lo, e você pode inspecioná-lo no Console (CloudFormation → Stacks → `app-cn-2026-ec2` → Change sets).

> Ou, pelo Console: CloudFormation → Stacks → `app-cn-2026-ec2` → Update → Make a direct update → Use existing template → Next → em InstanceType escolha `t3.small` → Next → Next → Submit. Acompanhe a aba Events até `UPDATE_COMPLETE`.

---

## Passo 14 — Atualizar a aplicação após um novo `docker push` - Opcional

Se alguém alterar o código, gerar uma nova imagem (Passos 5 e 6) e der `docker push`, a instância não atualiza sozinha: é preciso mandar ela baixar a nova versão e reiniciar o contêiner. O CloudFormation não participa dessa parte, quem faz isso é o Run Command do Systems Manager.

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --region us-east-1 --query Account --output text)

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

> Ou, pelo Console: Systems Manager → Run Command → Run command → documento `AWS-RunShellScript` → em Commands cole os quatro comandos, um por linha, **trocando `$ACCOUNT_ID` pelo número da sua conta** (no Console não existe a variável do CloudShell) → em Targets escolha Choose instances manually e selecione a instância → Run. O status por instância aparece na parte inferior da tela.

---

## Passo 15 — Encerrar e limpar os recursos (delete)

Cada stack é apagada com **um único comando**: o CloudFormation remove a instância e o security group juntos, sem você listar item por item. Apague primeiro a stack da instância e depois a do ECR.

```bash
aws cloudformation delete-stack --region us-east-1 --stack-name app-cn-2026-ec2
aws cloudformation wait stack-delete-complete --region us-east-1 --stack-name app-cn-2026-ec2

aws cloudformation delete-stack --region us-east-1 --stack-name app-cn-2026-ecr
aws cloudformation wait stack-delete-complete --region us-east-1 --stack-name app-cn-2026-ecr
```

**Verificação:**
```bash
aws cloudformation describe-stacks --region us-east-1 --stack-name app-cn-2026-ec2
```
deve retornar um erro dizendo que a stack `does not exist`, o que confirma que foi removida.

> Se a stack do ECR ficar em `DELETE_FAILED`, apague as imagens do repositório pelo console do ECR (ou com `aws ecr batch-delete-image`) e rode o `delete-stack` de novo.

> Ou, pelo Console: CloudFormation → Stacks → selecione `app-cn-2026-ec2` → Delete → Delete. Quando terminar, repita para `app-cn-2026-ecr`.

---
