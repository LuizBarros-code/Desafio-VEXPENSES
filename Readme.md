
# Desafio VEXPENSES

Esse desafio foi proposto para a vaga de estagio de Devops.




## Descrição Técnica da primeira tarefa

Este arquivo main.tf é um script Terraform para criação de uma infraestrutura na AWS.

provider "aws" {
  region = "us-east-1"
}

1. Nessa parte do codigo escolhe o provedor de serviços da nuvem que no caso seria o AWS, em seguida escolhe a região(us-east-1).
--

variable "projeto" {
  description = "Nome do projeto"
  type        = string
  default     = "VExpenses"
}

variable "candidato" {
  description = "Nome do candidato"
  type        = string
  default     = "SeuNome"
}

--

2. Nessa parte do codigo está definido as variaveis que serão utilizadas. onde usa:
  - description: que é opcional, é usada apenas como uma forma de documentar a variável para facilitar o entendimento.
  - Type: Define o tipo da variavel
  - default: opcional tambem usada para atribuir um valor padrão à variavel, caso o usuario não forneça.

--

resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

resource "aws_key_pair" "ec2_key_pair" {
  key_name   = "${var.projeto}-${var.candidato}-key"
  public_key = tls_private_key.ec2_key.public_key_openssh
}

--

3. Nessa etapa esta gerendo chaves para acessa a EC2, tls_private_key esta gerando uma chave privada com o algoritmo RSA. Essa chave é necessária para acessar a instância EC2, no aws_key_pair está a criar um par de chaves AWS. Usa a chave pública da chave gerada anteriomente e atribui um nome baseado nas variáveis projeto e candidato.

-- 

resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.projeto}-${var.candidato}-vpc"
  }
}

4.Nessa parte, esta criado uma VPC(Virtual private cloud), esqueci de explica na anterior o que é "resource", resource defini que você está criando um novo recurso no AWS, é nesse caso aqui é um resurso VPC, identificado pelo tipo aws_vpc, "main_vpc" é o nome que você atribuiu a esse recurso localmente no terraform. Esse nome pode ser usado em outras partes do codigo em referenciar a essa VPC.
- cidr_block: define o intervalo de endereços IP que a VPC poderá usar.
- enable_dns_support = true: Habilita o suporte a DNS dentro da VPC. Isso significa que a VPC será capaz de resolver nomes de domínio internamente.
- enable_dns_hostnames = true: Habilita a criação de nomes DNS para instâncias EC2.
- tags: A chave tags permite que você adicione rótulos (tags) a esse recurso no AWS. As tags são úteis para organizar e identificar seus recursos dentro da conta AWS. Esta tag cria um rótulo chamado Name, cujo valor será dinâmico, composto pelas variáveis projeto e candidato.

--

resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "${var.projeto}-${var.candidato}-subnet"
  }
}

5. Nessa etapa está a criar uma sub-rede dentro da VPC.
- vpc_id: Refere-se à VPC criada anteriormente.
- cidr_block: Define o bloco de endereços IP da sub-rede (10.0.1.0/24).
- availability_zone: Define a zona de disponibilidade onde a sub-rede será criada (us-east-1a).
- tags: fazendo a mesma coisa que a tag anterior da vpc. colocando -subnet.

--

resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main_vpc.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-igw"
  }
}

resource "aws_route_table" "main_route_table" {
  vpc_id = aws_vpc.main_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main_igw.id
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table"
  }
}

resource "aws_route_table_association" "main_association" {
  subnet_id      = aws_subnet.main_subnet.id
  route_table_id = aws_route_table.main_route_table.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table_association"
  }
}

6. Nesse bloco de código definem a criação de um Internet Gateway, uma tabela de rotas e a associação da tabela de rotas a uma sub-rede.
 - main_igw: é um componente da VPC que permite que as instâncias dentro de sub-redes públicas se comuniquem com a Internet.
    - vpc_id: Aqui estamos associando este Internet Gateway à VPC principal que foi criada anteriormente (main_vpc). O ID da VPC é recuperado através de aws_vpc.main_vpc.id, o que garante que o IGW será corretamente vinculado à VPC.
    - tags: Adicionamos tags para facilitar a identificação no console da AWS. O nome da tag será gerado dinamicamente usando as variáveis projeto e candidato.
- aws_route_table: é um componente essencial para gerenciar o roteamento dentro de uma VPC. Ela define como o tráfego deve ser roteado a partir de suas sub-redes. Neste caso, a tabela de rotas está configurada para permitir que o tráfego da VPC seja roteado para a Internet por meio do Internet Gateway.
    - vpc_id:  aws_vpc.main_vpc.id: A tabela de rotas está associada à VPC principal (main_vpc), garantindo que o tráfego dentro dessa VPC seja roteado conforme as regras especificadas nesta tabela.
    - route: Aqui é onde definimos as regras de roteamento.
        - cidr_block:  "0.0.0.0/0": Este é um bloco CIDR que representa todo o tráfego de saída (0.0.0.0/0 é o equivalente a "qualquer destino"). Isso significa que qualquer tráfego que não for local à VPC deve ser roteado para fora.
        - gateway_id:  aws_internet_gateway.main_igw.id: Qualquer tráfego destinado à Internet (definido pelo CIDR 0.0.0.0/0) será roteado para o Internet Gateway (main_igw) criado anteriormente. Isso garante que o tráfego de saída das instâncias que precisam acessar a Internet seja encaminhado corretamente.
    - tags: Assim como os outros recursos, as tags são usadas para identificar a tabela de rotas. O nome será gerado dinamicamente com as variáveis projeto e candidato.
- aws_route_table_association: define qual tabela de rotas está associada a uma determinada sub-rede. Neste caso, estamos associando a tabela de rotas criada anteriormente (main_route_table) com a sub-rede principal (main_subnet).
    - subnet_id: aws_subnet.main_subnet.id: Aqui, especificamos que a tabela de rotas será associada à sub-rede que criamos anteriormente (main_subnet), garantindo que o tráfego da sub-rede seja roteado conforme as regras da tabela.
    - route_table_id = aws_route_table.main_route_table.id: Este é o ID da tabela de rotas que será associada à sub-rede. No caso, estamos associando a main_route_table, que contém a rota para o Internet Gateway, permitindo que as instâncias na sub-rede possam acessar a Internet.
    - tags: Assim como os outros recursos, usamos tags para identificar a associação de tabela de rotas. O nome será gerado dinamicamente com as variáveis projeto e candidato, resultando em algo como VExpenses-SeuNome-route_table_association.

--

resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Permitir SSH de qualquer lugar e todo o tráfego de saída"
  vpc_id      = aws_vpc.main_vpc.id

  ingress {
    description      = "Allow SSH from anywhere"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  egress {
    description      = "Allow all outbound traffic"
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-sg"
  }
}

7. Nesse pedaço do codigo define um grupo de segurança (Security Group)
- aws_security_group define um grupo de segurança (Security Group) na AWS, que atua como um firewall virtual para controlar o tráfego de entrada e saída das instâncias EC2. O grupo de segurança especifica as regras de quais tipos de tráfego são permitidos para acessar as instâncias, além de controlar o tráfego de saída.
    - name: Define o nome do grupo de segurança. Ele é construído dinamicamente usando as variáveis projeto e candidato. Isso ajuda na identificação fácil desse grupo de segurança no console da AWS.
    - description: Fornece uma descrição clara do propósito deste grupo de segurança. Neste caso, a descrição indica que ele permite conexões SSH de qualquer lugar e todo o tráfego de saída.
    - vpc_id: Especifica a qual VPC este grupo de segurança será associado. Aqui, o ID da VPC é recuperado a partir do recurso aws_vpc.main_vpc, garantindo que o grupo de segurança seja vinculado corretamente à VPC.
- Ingress: define quais tipos de tráfego são permitidos para acessar as instâncias protegidas por este grupo de segurança. Neste caso, é uma regra que permite o tráfego SSH de qualquer lugar.
    - description: Uma breve descrição do que esta regra faz, que é "Permitir SSH de qualquer lugar".
    - from_port e to_port: Especifica a porta para a qual o tráfego é permitido. Neste caso, as portas de entrada e saída são ambas configuradas como 22, o que significa que apenas o tráfego para a porta 22 (porta padrão do SSH) será permitido.
    - protocol: Define o protocolo a ser utilizado. Aqui, estamos usando o TCP, que é o protocolo de transporte para conexões SSH.
    - cidr_blocks: Define os blocos CIDR para os quais o tráfego será permitido. Neste caso, ["0.0.0.0/0"] significa que o tráfego é permitido de qualquer lugar na Internet (IPv4). Essa regra abre a porta SSH para todos os endereços IP públicos, o que pode ser conveniente para acesso remoto, mas potencialmente inseguro se não for gerenciado adequadamente (como não limitar o acesso por IPs específicos).
    - ipv6_cidr_blocks: Semelhante ao cidr_blocks, mas para endereços IPv6. Aqui, ["::/0"] permite o tráfego SSH de qualquer lugar usando o protocolo IPv6.
- Egress: define o tráfego que as instâncias protegidas por este grupo de segurança podem enviar para fora (tráfego de saída). Neste caso, ela permite todo o tráfego de saída.
    - description: A descrição indica que "Todo o tráfego de saída é permitido", ou seja, qualquer instância associada a este grupo de segurança poderá fazer conexões de saída sem restrições.
    - from_port e to_port: Neste caso, ambos estão configurados como 0, indicando que não há restrições quanto à porta de origem ou destino, permitindo o tráfego de saída em todas as portas.
    - protocol = "-1": Aqui, o protocolo -1 é um valor especial que indica que todos os protocolos (TCP, UDP, ICMP, etc.) são permitidos.
    - cidr_blocks: ["0.0.0.0/0"] permite todo o tráfego de saída via IPv4, ou seja, as instâncias poderão se comunicar com qualquer IP público.
    - ipv6_cidr_blocks: ["::/0"] permite todo o tráfego de saída via IPv6, sem restrições de destino.
- tags: aqui estamos criando uma tag com o nome do grupo de segurança, construído com as variáveis projeto e candidato.

--

data "aws_ami" "debian12" {
  most_recent = true

  filter {
    name   = "name"
    values = ["debian-12-amd64-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["679593333241"]
}

resource "aws_instance" "debian_ec2" {
  ami             = data.aws_ami.debian12.id
  instance_type   = "t2.micro"
  subnet_id       = aws_subnet.main_subnet.id
  key_name        = aws_key_pair.ec2_key_pair.key_name
  security_groups = [aws_security_group.main_sg.name]

  associate_public_ip_address = true

  root_block_device {
    volume_size           = 20
    volume_type           = "gp2"
    delete_on_termination = true
  }

  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get upgrade -y
              EOF

  tags = {
    Name = "${var.projeto}-${var.candidato}-ec2"
  }
}

8. Este trecho de código em Terraform configura uma instância EC2 (máquina virtual) na AWS, usando a AMI (Amazon Machine Image) do Debian 12.
 - data: é usado para buscar e filtrar informações existentes na AWS, como uma AMI. No caso, ele encontra a AMI mais recente do Debian 12.
    - most_recent = true: Especifica que queremos a versão mais recente da AMI que corresponde aos filtros fornecidos.
    - filter: Filtra as AMIs disponíveis com base no nome e no tipo de virtualização.
        - name: "debian-12-amd64-*": Filtra as AMIs cujo nome contém a string debian-12-amd64-*, que se refere à versão Debian 12 de 64 bits.
        - irtualization-type = "hvm": Filtra para AMIs que usam a virtualização do tipo HVM (Hardware Virtual Machine), que é o tipo mais comum e eficiente para instâncias EC2 modernas.
    - owners = ["679593333241"]: Restringe a busca para as AMIs criadas pelo proprietário com o ID 679593333241, que é o proprietário oficial da AMI do Debian 12.
- aws_instance: cria uma instância EC2 na AWS usando a AMI filtrada no bloco anterior.
    - ami = data.aws_ami.debian12.id: Especifica a AMI (imagem de máquina) que será usada para inicializar a instância EC2. O ID da AMI vem do recurso data que busca a AMI Debian 12.
    - instance_type = "t2.micro": Define o tipo da instância EC2, aqui é usado t2.micro, que é uma instância de baixo custo com recursos limitados (1 vCPU, 1 GB de RAM). Esse tipo é elegível para o nível gratuito da AWS.
    - subnet_id = aws_subnet.main_subnet.id: Associa a instância à subnet criada anteriormente no VPC, garantindo que a instância seja provisionada na rede definida.
    - key_name = aws_key_pair.ec2_key_pair.key_name: Refere-se à chave SSH criada anteriormente (aws_key_pair), que será usada para fazer login na instância.
    - security_groups = [aws_security_group.main_sg.name]: Associa o grupo de segurança criado anteriormente (main_sg) a esta instância, garantindo que as regras de segurança (como permitir SSH) se apliquem à instância.
    - associate_public_ip_address = true: Este parâmetro garante que a instância receba um endereço IP público, tornando-a acessível pela Internet.
- root_block_device: define as configurações do volume de armazenamento principal (EBS) da instância.
    - volume_size = 20: Especifica que o volume root da instância terá 20 GB de espaço em disco.
    - volume_type = "gp2": Define o tipo do volume como GP2 (General Purpose SSD), que é adequado para cargas de trabalho comuns e oferece um bom equilíbrio entre custo e desempenho.
    - delete_on_termination = true: Garante que o volume será automaticamente excluído quando a instância for terminada, economizando custos ao evitar volumes órfãos.
- user_data: contém um script que será executado automaticamente na primeira vez que a instância for inicializada. No caso, o script faz atualizações no sistema.
    - apt-get update -y: Atualiza a lista de pacotes disponíveis no Debian.
    - apt-get upgrade -y: Atualiza todos os pacotes instalados para suas versões mais recentes.
- tags: Aqui, o nome da instância é gerado dinamicamente usando as variáveis projeto e candidato.

--

output "private_key" {
  description = "Chave privada para acessar a instância EC2"
  value       = tls_private_key.ec2_key.private_key_pem
  sensitive   = true
}

output "ec2_public_ip" {
  description = "Endereço IP público da instância EC2"
  value       = aws_instance.debian_ec2.public_ip
}


9. trecho de código em Terraform define as saídas do script, ou seja, as informações que serão exibidas após a execução da infraestrutura, para facilitar o uso e a visualização de informações importantes.
- private_key: exibe a chave privada gerada para acessar a instância EC2 via SSH.
    - description = "Chave privada para acessar a instância EC2": Isso é uma breve descrição do que essa saída representa. Neste caso, indica que é a chave privada SSH gerada para acessar a instância EC2.
    - value = tls_private_key.ec2_key.private_key_pem: Aqui, o valor exibido será o conteúdo da chave privada gerada no recurso tls_private_key anterior. A chave é necessária para autenticação quando você tentar se conectar à instância EC2 via SSH.
    - sensitive = true: Esta opção marca a saída como sensível, o que significa que o valor da chave privada não será exibido diretamente no terminal ou em logs para proteger sua confidencialidade. Isso evita a exposição acidental de dados sensíveis, como chaves privadas, o que é essencial para a segurança.
### Observações:  A chave privada é fundamental para você se conectar via SSH à instância EC2. Ao marcar essa saída como sensível, o Terraform protege o acesso direto, mas ainda permite que você veja o valor em um ambiente seguro, como gravando-o em um arquivo.

- ec2_public_ip: exibe o endereço IP público da instância EC2, que será usado para acessá-la pela internet.
    - description = "Endereço IP público da instância EC2": Uma breve descrição informando que essa saída mostrará o IP público da instância EC2 criada.
    - value = aws_instance.debian_ec2.public_ip: O valor exibido será o endereço IP público da instância EC2, obtido diretamente da instância criada (aws_instance.debian_ec2). Esse IP é atribuído automaticamente à instância pelo serviço de rede da AWS (porque associate_public_ip_address = true foi configurado).

--

## Aplicação de melhorias de segurança

resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Permitir SSH de um IP específico e todo o tráfego de saída"
  vpc_id      = aws_vpc.main_vpc.id

  ingress {
    description = "Allow SSH from a specific IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["<YOUR_IP>/32"]  
  }

  egress {
    description      = "Allow all outbound traffic"
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-sg"
  }
}

1. Restringir o Acesso SSH no Security Group: O SSH atualmente está configurado para permitir conexões de qualquer lugar (CIDR "0.0.0.0/0"), o que expõe a instância a ataques de força bruta ou tentativas de exploração.
- Em vez de permitir o SSH de qualquer lugar, restrinja o acesso ao SSH apenas ao endereço IP específico que você usará para se conectar.

## Automação da Instalação do Nginx

resource "aws_instance" "debian_ec2" {
  ami             = data.aws_ami.debian12.id
  instance_type   = "t2.micro"
  subnet_id       = aws_subnet.main_subnet.id
  key_name        = aws_key_pair.ec2_key_pair.key_name
  security_groups = [aws_security_group.main_sg.name]

  associate_public_ip_address = true

  root_block_device {
    volume_size           = 20
    volume_type           = "gp2"
    delete_on_termination = true
  }

  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get upgrade -y

              # Instalar o Nginx
              apt-get install -y nginx

              # Iniciar o serviço do Nginx
              systemctl start nginx
              
              # Habilitar o Nginx para iniciar automaticamente após reboot
              systemctl enable nginx
              EOF

  tags = {
    Name = "${var.projeto}-${var.candidato}-ec2"
  }
}

- apt-get update -y e apt-get upgrade -y: Atualiza a lista de pacotes disponíveis e aplica quaisquer upgrades disponíveis para garantir que a instância esteja atualizada.
- apt-get install -y nginx: Instala o servidor web Nginx.
- systemctl start nginx: Inicia o serviço do Nginx assim que ele é instalado.
- systemctl enable nginx: Configura o Nginx para iniciar automaticamente sempre que a instância for reiniciada.
















  




