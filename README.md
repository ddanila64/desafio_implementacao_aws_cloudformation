# desafio_implementacao_aws_cloudformation

Este README descreve o que o template `ExemploConfigCloudFormation-corrigido.yml` faz, os parâmetros que ele expõe, os recursos criados, saídas (outputs), pontos de atenção de segurança e instruções rápidas de implantação e remoção.

Resumo do que eu fiz
- Busca a AMI via SSM (AMI mais recente do Amazon Linux 2).
- Separei Security Groups para web e banco (DB) e restrinigi SSH via parâmetro.
- Criei subnets públicas e privadas em duas AZs e usei ambas no DBSubnetGroup.
- Removi credenciais hardcoded: a senha do RDS é um parâmetro NoEcho.
- Corrigi o output do DNS público da EC2 e adicionei outputs úteis (IP público e endpoint RDS).
- Adicionei UserData simples para instalar e iniciar Apache na instância web.

O que o template cria
- VPC (10.0.0.0/16) com DNS habilitado.
- 2 subnets públicas (em AZs distintas).
- 2 subnets privadas (em AZs distintas) — usadas pelo RDS.
- Internet Gateway e rota pública associada às subnets públicas.
- Route Table pública e associações às subnets públicas.
- Security Group para a camada Web:
  - HTTP (80) aberto para 0.0.0.0/0.
  - SSH (22) permitido apenas a um CIDR parametrizado (AllowedSSHLocation).
- Security Group para o RDS:
  - Permite apenas MySQL (3306) vindo do WebServerSG.
- Instância EC2 (WebServer) em uma subnet pública:
  - Tipo parametrizado (default t3.micro).
  - AMI obtida via SSM (padrão: Amazon Linux 2 mais recente).
  - UserData instala e inicia Apache e coloca uma página simples.
  - Usa o KeyPair informado no parâmetro KeyName (deve existir na sua conta/região).
- RDS MySQL (instância única, não-publicly accessible) em subnets privadas:
  - DBSubnetGroup com as duas subnets privadas.
  - Credenciais: DBUsername (padrão admin) e DBPassword (NoEcho — obrigatório fornecer).
  - VPCSecurityGroups aponta para o DBSecurityGroup que somente abre 3306 para o WebServerSG.

Parâmetros importantes
- KeyName (AWS::EC2::KeyPair::KeyName): nome do par de chaves existente.
- LatestAmiSSM (SSM parameter): por padrão aponta para /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2.
- InstanceType: tipo da EC2 (default: t3.micro).
- AllowedSSHLocation: CIDR que terá acesso SSH (default: 0.0.0.0/0 — altere para seu_ip/32).
- DBUsername: usuário master do RDS (default: admin).
- DBPassword: senha do RDS (NoEcho — não exibida nos logs/console).
- DBInstanceClass: classe do RDS (default: db.t3.micro).

Outputs (úteis após o deploy)
- WebsiteURL: http://<PublicDnsName> da EC2.
- WebServerPublicIP: IP público da instância web.
- DBEndpoint: endpoint interno do RDS (Host).
- DBPort: porta do RDS.

Recomendações e pontos de atenção (antes de criar o stack)
- Sempre restrinja AllowedSSHLocation ao seu IP/32 (não deixe 0.0.0.0/0 em produção).
  - Para descobrir seu IP público, por exemplo: curl https://checkip.amazonaws.com
- Forneça DBPassword ao criar o stack (sem valor padrão). Considere usar AWS Secrets Manager para armazenamento/rotacionamento de segredos em vez de parâmetros.
- O template não cria NAT Gateway: instâncias em subnets privadas não terão acesso à internet para updates. Se quiser que o RDS ou outras instâncias privadas façam download/patches, adicione NAT Gateway(s) e rotas privadas.
- AMI via SSM torna o template portátil entre regiões (desde que o parâmetro SSM exista na região). Em regiões que não tenham o mesmo parâmetro, ajuste ou use mapping.
- Verifique quotas e limites (EC2, RDS, VPCs) na sua conta/região.
- Se precisar de alta disponibilidade no banco, considere Multi-AZ e classes adequadas ao tráfego/SLAs.
- Em produção, considere:
  - ALB (Application Load Balancer) e Auto Scaling para a camada web.
  - WAF para proteção HTTP.
  - Monitoramento CloudWatch e alarmes.
  - Uso de endpoints internos privados, roles IAM e políticas para maior segurança.

Como implantar (exemplo com AWS CLI)
1. Faça download do arquivo `ExemploConfigCloudFormation-corrigido.yml`.
2. Execute (exemplo):
   aws cloudformation deploy \
     --template-file ExemploConfigCloudFormation-corrigido.yml \
     --stack-name exemplo-stack \
     --parameter-overrides KeyName=meu-keypair DBPassword="SenhaSegura123!" AllowedSSHLocation="1.2.3.4/32" \
     --capabilities CAPABILITY_NAMED_IAM

Observações:
- Substitua `meu-keypair`, `SenhaSegura123!` e `1.2.3.4/32` pelos valores reais. Mesmo que o template não crie IAM roles, incluir --capabilities não faz mal; remova se preferir.
- Se a sua senha contém caracteres especiais, coloque-a entre aspas ou use um arquivo de parâmetros.

Como remover (limpar)
- Pelo Console do CloudFormation: selecione o stack e clique em "Delete".
- Pela CLI:
  aws cloudformation delete-stack --stack-name exemplo-stack

Possíveis melhorias futuras (opcionais)
- Adicionar NAT Gateway(s) e rotas privadas para permitir atualizações/egress das subnets privadas.
- Tornar o RDS Multi-AZ e/ ou usar uma classe maior conforme demanda.
- Usar Secrets Manager para o DBPassword e referenciar via CustomResource ou parâmetro seguro.
- Incluir ALB + Auto Scaling para a camada web.
- Habilitar logs (RDS logs export) e alarmes CloudWatch.
- Adicionar tags padronizadas (ambiente, projeto, dono) para governança.
