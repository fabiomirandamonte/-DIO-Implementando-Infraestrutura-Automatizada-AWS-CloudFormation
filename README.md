# Implementação de Infraestrutura Automatizada com AWS CloudFormation 🚀

Bem-vindo ao repositório de documentação do **Laboratório AWS CloudFormation**, desenvolvido como parte do Bootcamp da DIO (Digital Innovation One)!

Este projeto demonstra como implementar **Infraestrutura como Código (IaC)** utilizando o **AWS CloudFormation**. Ao definir recursos em templates estruturados (YAML/JSON), automatizamos a criação, atualização e gerenciamento de infraestrutura em nuvem na AWS de forma repetível, consistente e segura.

---

## 📌 Sumário
- [Visão Geral e Arquitetura](#-visão-geral-e-arquitetura)
- [Conceitos-Chave](#-conceitos-chave)
- [Fluxo de Trabalho e Arquitetura do Projeto](#-fluxo-de-trabalho-e-arquitetura-do-projeto)
- [Pré-requisitos](#-pré-requisitos)
- [Passo a Passo da Implementação](#-passo-a-passo-da-implementação)
  - [1. Criando o Template CloudFormation](#1-criando-o-template-cloudformation)
  - [2. Implantando a Stack pelo Console AWS](#2-implantando-a-stack-pelo-console-aws)
  - [3. Implantando a Stack via AWS CLI](#3-implantando-a-stack-via-aws-cli)
- [Verificação e Testes](#-verificação-e-testes)
- [Insights e Aprendizados](#-insights-e-aprendizados)
- [Exclusão de Recursos (Teardown)](#-exclusão-de-recursos-teardown)
- [Referências](#-referências)

---

## 📐 Visão Geral e Arquitetura

O AWS CloudFormation permite modelar toda a sua infraestrutura em arquivos de texto. Neste projeto, criamos uma *stack* que provisiona:
1. **AWS Virtual Private Cloud (VPC)** com bloco CIDR customizado.
2. **Subrede Pública (Subnet)** dentro da VPC.
3. **Internet Gateway (IGW)** anexado à VPC para conectividade externa.
4. **Tabela de Roteamento (Route Table)** direcionando o tráfego externo para o Internet Gateway.
5. **Grupo de Segurança (Security Group)** permitindo tráfego HTTP (Porta 80) e SSH (Porta 22).
6. **Instância Amazon EC2** (Amazon Linux 2023) rodando automaticamente um servidor web Apache via script `UserData`.

---

## 🔑 Conceitos-Chave

| Conceito | Descrição |
| :--- | :--- |
| **Template** | Arquivo de texto formatado em JSON ou YAML que descreve os recursos da AWS que você deseja criar e configurar. |
| **Serviço CloudFormation** | O mecanismo da AWS que lê o template e executa chamadas de API para provisionar e configurar os recursos. |
| **Stack (Pilha)** | Unidade única de gerenciamento para um conjunto de recursos AWS relacionados, criados e gerenciados juntos pelo CloudFormation. |
| **Change Sets** | Um resumo das alterações propostas em uma stack, permitindo visualizar como as atualizações afetarão os recursos em execução antes de executá-las. |

---

## 🔄 Fluxo de Trabalho e Arquitetura do Projeto

O fluxo de trabalho principal do CloudFormation segue uma progressão em três etapas:

![Fluxo AWS CloudFormation](images/cloudformation-flow.png)

1. **Definição do Template**: Criação de um arquivo YAML ou JSON especificando os recursos, parâmetros e saídas (*outputs*) desejados.
2. **Execução no CloudFormation**: Envio do template para o AWS CloudFormation via Console de Gerenciamento ou AWS CLI.
3. **Criação da Stack**: O CloudFormation provisiona todos os recursos especificados no template como uma pilha unificada.

---

## 🛠️ Pré-requisitos

Antes de começar, certifique-se de ter:
- Uma **Conta AWS** ativa com permissões suficientes (usuário/role IAM com acesso ao CloudFormation, EC2 e VPC).
- **AWS CLI** instalado e configurado (opcional, se optar por implantar via linha de comando).
- Um editor de código (ex: VS Code) com suporte a YAML.

---

## 💻 Passo a Passo da Implementação

### 1. Criando o Template CloudFormation

Salve o seguinte template YAML como `template.yaml` no seu diretório local:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Bootcamp DIO - Infraestrutura de Servidor Web Basico com VPC, Subnet, EC2 e Security Group'

Parameters:
  EnvironmentName:
    Type: String
    Default: DIOLab
    Description: Prefixo do nome para os recursos criados.

  InstanceTypeParam:
    Type: String
    Default: t2.micro
    AllowedValues:
      - t2.micro
      - t3.micro
    Description: Tipo de Instancia EC2.

Resources:
  # 1. Criacao da VPC
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-VPC'

  # 2. Subrede Publica
  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 10.0.1.0/24
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-PublicSubnet'

  # 3. Internet Gateway
  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-IGW'

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref MyVPC
      InternetGatewayId: !Ref InternetGateway

  # 4. Tabela de Roteamento e Rota para o IGW
  RouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MyVPC
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-PublicRouteTable'

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref RouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  SubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet
      RouteTableId: !Ref RouteTable

  # 5. Grupo de Seguranca (Security Group)
  WebServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Permite trafego de entrada HTTP e SSH
      VpcId: !Ref MyVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0

  # 6. Instancia EC2
  WebServerInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceTypeParam
      ImageId: ami-0c101bfb6ee701317 # Amazon Linux 2023 (us-east-1)
      SubnetId: !Ref PublicSubnet
      SecurityGroupIds:
        - !Ref WebServerSecurityGroup
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          dnf update -y
          dnf install -y httpd
          systemctl start httpd
          systemctl enable httpd
          echo "<h1>Bem-vindo ao Laboratorio de AWS CloudFormation da DIO!</h1>" > /var/www/html/index.html
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-WebServer'

Outputs:
  WebsiteURL:
    Description: URL do Servidor Web
    Value: !Sub 'http://${WebServerInstance.PublicDnsName}'
  VPCId:
    Description: ID da VPC criada pela stack
    Value: !Ref MyVPC
```

---

### 2. Implantando a Stack pelo Console AWS

1. Faça login no **Console de Gerenciamento da AWS**.
2. Pesquise e acesse o serviço **CloudFormation**.
3. Clique em **Create stack** (Criar pilha) > **With new resources (standard)**.
4. Selecione **Template is ready** (O modelo está pronto) e escolha **Upload a template file** (Enviar um arquivo de modelo).
5. Faça o upload do arquivo `template.yaml` e clique em **Next** (Próximo).
6. Digite o nome da stack (ex: `dio-cloudformation-lab`).
7. Revise os parâmetros e clique em **Next**.
8. Ajuste as opções adicionais se desejado (as configurações padrão funcionam perfeitamente) e clique em **Next**.
9. Revise todas as informações e clique em **Submit** (Enviar).
10. Acompanhe os eventos até que o status mude para **`CREATE_COMPLETE`**.

---

### 3. Implantando a Stack via AWS CLI

Se preferir utilizar a linha de comando da AWS:

```bash
aws cloudformation create-stack   --stack-name dio-cloudformation-lab   --template-body file://template.yaml   --parameters ParameterKey=EnvironmentName,ParameterValue=DIOLab
```

Para verificar o status do implantação:

```bash
aws cloudformation describe-stacks --stack-name dio-cloudformation-lab
```

---

## ✅ Verificação e Testes

1. Assim que o status da stack mostrar **`CREATE_COMPLETE`**, acesse a aba **Outputs** (Saídas) no Console do AWS CloudFormation.
2. Copie o valor da chave `WebsiteURL` (ou pegue o IP Público da instância diretamente no console do EC2).
3. Abra o navegador e acesse o endereço copiado.
4. Você deverá ver a página com a mensagem:
   > **Bem-vindo ao Laboratorio de AWS CloudFormation da DIO!**

---

## 💡 Insights e Aprendizados

- **Declarativo vs Imperativo**: O CloudFormation permite definir *o que* você quer que sua infraestrutura seja, deixando para a AWS o trabalho de descobrir *como* construí-la.
- **Gerenciamento de Dependências**: O CloudFormation gerencia automaticamente a ordem de criação dos recursos (por exemplo, criando a VPC antes de tentar anexar uma Subnet ou Internet Gateway).
- **Detecção de Desvio (Drift Detection)**: Permite identificar alterações manuais feitas fora da stack que possam ter modificado a infraestrutura original.
- **Reutilização e Parâmetros**: O uso de parâmetros torna o mesmo template reaproveitável para diferentes ambientes (Desenvolvimento, Staging, Produção).

---

## 🧹 Exclusão de Recursos (Teardown)

Para evitar cobranças indesejadas na sua conta AWS:

**Pelo Console AWS:**
1. Selecione a stack (`dio-cloudformation-lab`).
2. Clique em **Delete** (Excluir).
3. Confirme a exclusão. O CloudFormation removerá todos os recursos criados na ordem inversa de dependência.

**Via AWS CLI:**
```bash
aws cloudformation delete-stack --stack-name dio-cloudformation-lab
```

---

## 📚 Referências

- [Guia do Usuário do AWS CloudFormation](https://docs.aws.amazon.com/pt_br/AWSCloudFormation/latest/UserGuide/Welcome.html)
- [DIO - Digital Innovation One](https://www.dio.me/)
- [Guia de Sintaxe Markdown no GitHub](https://docs.github.com/pt/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-formatting-syntax)
