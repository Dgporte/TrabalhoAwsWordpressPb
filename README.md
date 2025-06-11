# Passo a Passo: Criando VPC, Subnets e Grupos de Segurança na AWS

## 1. Página Inicial da AWS

Ao acessar a AWS, esta é a página principal onde você pode buscar por todos os serviços.

![Página Principal AWS](assests/PAGEPRINCIPAL/PagePrincipal.png)

---

## 2. Acessando o Serviço VPC

No canto superior direito, digite "VPC" para acessar o serviço de redes virtuais privadas.

![Busca por VPC](assests/VPC/DigitarVpc.png)

---

## 3. Página da VPC

Aqui está a página inicial do serviço VPC. Clique em "Criar VPC" para iniciar a configuração.

![Página da VPC](assests/VPC/PageCriarVpc.png)

---

## 4. Criação da VPC – Configuração do CIDR

Ao criar a VPC, insira o bloco CIDR IPv4 como `10.0.0.0/16`.  
**Por quê?**  
Esse bloco permite até 65.536 endereços IP, oferecendo flexibilidade para criar várias subnets públicas e privadas dentro da mesma VPC.

![Criação da VPC - CIDR](assests/VPC/CriaçãodaVpc.png)

---

## 5. Criação da Subnet

Acesse a página de Subnets e clique em "Criar Sub-rede".

![Página Subnet](assests/SUBNET/PageCriarSubNet.png)

---

## 6. Seleção da VPC ao criar Subnet

Durante a criação da subnet, selecione a sua VPC recém-criada.

![Selecionar VPC](assests/SUBNET/SelecionarVpcSubNet.png)

---

## 7. Configuração de Subnet

Após selecionar a VPC, configure as subnets conforme necessário.

![Configuração de Subnet](assests/SUBNET/OpcoesCriarSubNet.png)

---

## 8. Criação de Subnets em us-east-1b

Crie duas subnets na zona de disponibilidade `us-east-1b`:

- **Subnet Pública:** Bloco CIDR IPv4: `10.0.1.0/25`
- **Subnet Privada:** Bloco CIDR IPv4: `10.0.2.0/25`

**Justificativas:**
- O bloco `/25` separa cada subnet em 128 endereços IP, suficiente para recursos públicos/privados.
- Ambas devem estar na mesma zona para garantir alta disponibilidade e facilitar comunicação local.

![Subnets us-east-1b](assests/SUBNET/CriacaoSubNetPublicaPrivada01.png)

---

## 9. Criação de Subnets em us-east-1d

Repita o processo para a zona `us-east-1d`:

- **Subnet Pública:** Bloco CIDR IPv4: `10.0.1.128/25`
- **Subnet Privada:** Bloco CIDR IPv4: `10.0.2.128/25`

**Justificativas:**
- O bloco começa em 128, não sobrepondo endereços da subnet anterior.
- Ambas na mesma zona para redundância e tolerância a falhas.

![Subnets us-east-1d](assests/SUBNET/CriacaoSubNetPublicaPrivada02.png)

---

## 10. Criando Grupos de Segurança

Clique em "Criar grupo de segurança" para definir regras de acesso.

![Criar Grupo de Segurança](assests/GRUPODESEGURANCA/PageGupoDeSeguranca.png)

---

## 11. Página de Criação de Grupo de Segurança

Aqui você define nome, descrição e regras de entrada/saída.

![Página Grupo de Segurança](assests/GRUPODESEGURANCA/PageCriarGrupoSeguranca.png)

---

## 12. Grupo de Segurança da EC2

- **Entradas:**
  - SSH (22) → Meu IP: Para garantir que apenas você possa acessar via SSH.
  - HTTP (80) → SG do Load Balancer: Permite que apenas o Load Balancer acesse via HTTP.
  - HTTP (80) → Meu IP: Para testes diretos de acesso à instância.
- **Saída:** Todo o tráfego (`0.0.0.0/0`) para permitir acesso externo.

**Justificativa:**  
Separação clara de acessos administrativos (SSH), público (HTTP via LB) e testes.

![SG EC2](assests/GRUPODESEGURANCA/GSEC2.png)

---

## 13. Grupo de Segurança do Load Balancer

- **Entrada:** HTTP (80) de qualquer lugar (`0.0.0.0/0`), pois o LB recebe o tráfego público.
- **Saída:** HTTP (80) para qualquer destino.

**Justificativa:**  
Permite que o LB atenda requisições de qualquer origem e encaminhe para as instâncias EC2.

![SG Load Balancer](assests/GRUPODESEGURANCA/GSLOADBALANCER.png)

---

## 14. Grupo de Segurança do RDS

- **Entrada:** MySQL/Aurora (3306) permitido apenas pelo SG da EC2.
- **Saída:** Todo tráfego (`0.0.0.0/0`).

**Justificativa:**  
Restringe o acesso ao banco de dados, permitindo apenas que as instâncias EC2 possam conectar.

![SG RDS](assests/GRUPODESEGURANCA/GSRDS.png)

---

## 15. Grupo de Segurança do EFS

- **Entrada:** NFS (2049) permitido apenas pelo SG da EC2.
- **Saída:** Todo tráfego (`0.0.0.0/0`).

**Justificativa:**  
Somente as instâncias EC2 podem montar o EFS, garantindo segurança ao sistema de arquivos.

![SG EFS](assests/GRUPODESEGURANCA/GSEFS.png)

---

## 16. Página da Tabela de Rotas

Aqui está a página onde você pode visualizar e gerenciar as tabelas de rotas da sua VPC. Clique em "Criar tabela de rotas" para iniciar o processo.

![Página Tabela de Rotas](assests/TABELAROTAS/PageTabelaRotas.png)

---

## 17. Página para Criar Tabela de Rotas

Nesta tela você define o nome, descrição e associa a tabela à sua VPC.

![Criar Tabela de Rotas](assests/TABELAROTAS/PageCriarTabelaRotas.png)

---

## 18. Criando Tabela de Rotas Pública

Crie uma tabela de rotas pública.  
**Por quê?**  
A tabela de rotas pública é fundamental para subnets que precisam acessar a internet diretamente. Ao associar esta tabela às subnets públicas, você poderá adicionar uma rota apontando para o Internet Gateway, permitindo que recursos nessas subnets sejam acessados pela internet (por exemplo, instâncias EC2 que precisam servir páginas web ou Load Balancers públicos).

![Tabela de Rotas Pública](assests/TABELAROTAS/TabelaRotasPublica.png)

---

## 19. Criando Tabela de Rotas Privada

Crie também uma tabela de rotas privada.  
**Por quê?**  
A tabela de rotas privada é destinada às subnets privadas, que não devem ter acesso direto à internet. Em vez disso, o tráfego de saída dessas subnets deve passar por um NAT Gateway (que está em uma subnet pública), garantindo maior segurança para recursos internos, como bancos de dados ou aplicações internas.

![Tabela de Rotas Privada](assests/TABELAROTAS/TabelaRotasPrivada.png)

---

## 20. Associando Subnets Públicas à Tabela de Rotas Pública

Selecione a tabela de rotas pública e clique na aba “Associações de sub-rede”, depois em “Editar associações de sub-rede”.

![Editar Associações Tabela Pública](assests/ASSOCIACAODESUBNET/ClicarAssociacaoDeSubNetPublica.png)

---

Na página de associações da tabela pública, selecione as subnets públicas criadas anteriormente (ex: `10.0.1.0/25` e `10.0.1.128/25`).  
Assim, estas subnets terão acesso à internet conforme a rota configurada para o Internet Gateway.

![Selecionar Subnets Públicas](assests/ASSOCIACAODESUBNET/AssociacaoSubNetPublica.png)

---

## 21. Associando Subnets Privadas à Tabela de Rotas Privada

Agora, selecione a tabela de rotas privada, clique em “Associações de sub-rede” e em “Editar associações de sub-rede”.

![Editar Associações Tabela Privada](assests/ASSOCIACAODESUBNET/ClicarAssociacaoDeSubNetPrivada.png)

---

Na página de associações da tabela privada, selecione as subnets privadas criadas (`10.0.2.0/25` e `10.0.2.128/25`).  
Dessa forma, essas subnets utilizarão o NAT Gateway para acesso à internet apenas para saída, mantendo os recursos internos protegidos de acessos externos diretos.

![Selecionar Subnets Privadas](assests/ASSOCIACAODESUBNET/AssociacaoSubNetPrivada.png)

---

## 22. Página do Gateway de Internet

Esta é a página onde você pode visualizar e gerenciar os Internet Gateways da sua VPC. Clique em "Criar gateway de internet" para adicionar um novo gateway à sua infraestrutura.

![Página Gateway de Internet](assests/GATEWAYINTERNET/PageGatewayInternet.png)

---

## 23. Criando o Gateway de Internet

Na página de criação, defina um nome para facilitar a identificação do recurso.

**Por que criar um Internet Gateway?**  
O Internet Gateway é o componente responsável por permitir que recursos em subnets públicas da sua VPC se comuniquem com a internet. Ele serve como a porta de entrada e saída do tráfego externo, sendo fundamental para instâncias EC2 públicas, Load Balancers e outros serviços que precisam ser acessados a partir da internet.

![Criação do Gateway de Internet](assests/GATEWAYINTERNET/PageCriarGatewayInternet.png)

---

## 24. Por que criar um EFS?

O Amazon EFS (Elastic File System) permite que várias instâncias EC2 compartilhem o mesmo sistema de arquivos simultaneamente, sendo ideal para aplicações como o WordPress, onde é necessário persistir arquivos (uploads, plugins, temas) entre múltiplas instâncias em um ambiente escalável.

---

## 25. Página Principal do EFS

Esta é a página inicial do serviço EFS, onde você pode visualizar e criar sistemas de arquivos.

![Página Principal EFS](assests/EFS/PagePrincipalEFS.png)

---

## 26. Criando um novo EFS

Clique em “Criar sistema de arquivos”, defina o nome, selecione sua VPC e clique em “Personalizar”.

![Criar EFS: Nome e VPC](assests/EFS/CriarEfs.png)

---

## 27. Etapa 1 – Configurações Gerais

- **Regional:** O EFS é um serviço regional, garantindo alta disponibilidade entre zonas na mesma região.
- **Backup automático:** Não habilitar nesta etapa, pois para ambientes de teste ou controle de custos, backups automáticos podem ser desnecessários.
- **Gerenciamento de ciclo de vida:** Não habilite, ou defina como “Intermitente”, para ambientes com uso variável.
- **Performance:** Escolha “Uso geral” para garantir baixa latência em aplicações web como o WordPress.
- **Tag:** Adicione uma tag com seu nome para facilitar a identificação e controle do recurso.

![Etapa 1 EFS](assests/EFS/Etapa1EFS.png)

---

## 28. Etapa 2 – Rede

Selecione a VPC correta e altere os grupos de segurança para utilizar o grupo de segurança do EFS que você criou anteriormente. Isso garante que apenas as instâncias EC2 autorizadas possam montar o EFS.

![Etapa 2 EFS - Rede e SG](assests/EFS/Etapa2EFS.png)

---

## 29. Etapa 3 – Configurações Avançadas

Nesta etapa, apenas clique em “Next” para prosseguir, pois as configurações padrão normalmente já atendem ao cenário.

![Etapa 3 EFS](assests/EFS/Etapa3EFS.png)

---

## 30. Etapa 4 – Criar o EFS

Revise as configurações e clique em “Criar sistema de arquivos”.

![Criar EFS](assests/EFS/Etapa4EFS.png)

---

## 31. Aguardando a Criação

Aguarde até que o status do EFS mude para “Disponível” para começar a utilizá-lo.

![EFS sendo criado](assests/EFS/EFSCRIADO.png)

---
