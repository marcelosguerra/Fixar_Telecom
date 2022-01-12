## 1.0 - Documento Implantação BNG NE
Documento com anotações para a implantação do BNG Huawei e IXC

## 2.0 - Documento Implantação BNG NE
Documentos IXC para integração com NE.

### 2.1 - IXC Resumo suportado
![image](https://user-images.githubusercontent.com/69680913/148784508-348011e0-65df-487f-874e-a44abffe77ab.png)

### 2.2 - IXC link de procedimento
http://wiki.ixcsoft.com.br/index.php/CONCENTRADOR_HUAWEI


## 3.0 - Modelo de configuração NE BNG PPPOE com IXC

### 3.1 - Inicio

- NE deve estar com conectividade com IXC
- NE deve estar atualizado com o ultimo software e patch
- NE deve estar com a licença de PPPoE aplicado.

### 3.2 - Definir hostname do IXC
Definir o hostanme pois o IXC utilizará ao aplicar o script
<pre>
sysname RTD_BNG_01
commit
</pre>

### 3.3 - Definir loopback equipamento
<pre>
interface Loopback0
description GERENCIA - NE-BNG
</pre>

### 3.4 - dhcpv6 duid
Importante para o usuario poder pegar DHCPv6
<pre>
dhcpv6 duid llt
</pre>

### 3.5 - Estaticas do pool ipv6
<pre>
ipv6-pool statistic include shared-user
</pre>


### 3.6 - Configuração de um pool IPV4 - WAN
<pre>
ip pool IPV4_WAN_POOL_PPPOE_01 bas local
gateway 100.64.128.1 255.255.224.0
section 0 100.64.128.10 100.64.159.254
dns-server 177.152.85.218 8.8.8.8
</pre>


###  3.7 - Configuração de um pool IPV6 - WAN
<pre>
ipv6 prefix IPV6_WAN_PREFIX_PPPOE_01 local
    prefix 2804:dec:8100::/40 

ipv6 pool IPV6_WAN_POOL_PPPOE_01 bas local  
    dns-server 2804:dec:0:4::2 2001:4860:4860::8888
    prefix IPV6_WAN_PREFIX_PPPOE_01
</pre>



### 3.8 - Configuração de um pool IPV6 - PD Navegação.
<pre>
ipv6 prefix IPV6_PD_PREFIX_PPPOE_01 delegation
    prefix 2804:dec:8200::/40 delegating-prefix-length 56
    lifetime preferred-lifetime days 2 hours 0 minutes 0 valid-lifetime days 10 hours 0 minutes 0
    frame-ipv6 lease manage
    reserved prefix mac lease

ipv6 pool  IPV6_PD_POOL_PPPOE_01 bas delegation  
    dns-server 2804:dec:0:4::2 2001:4860:4860::8888
    prefix IPV6_PD_PREFIX_PPPOE_01
</pre>


### 3.9 - Definir usuarios do IXC
Definir o usuarios para o servidor IXC pode logar no equipamento.

<pre>
aaa
local-user IXC_BNG_XXX password irreversible-cipher XXX@Empresa@erp#300#
local-user IXC_BNG_XXX service-type ssh
local-user IXC_BNG_XXX level 3
local-user IXC_BNG_XXX state block fail-times 3 interval 5
quit
commit
</pre>


## 4.0 - Cadastrar o concentrador NE no IXC

### 4.1 Senha para o servidor Radius: 
O IXC vai pedir uma senha para o radiu, então escolha algo do tipo: Escolh3rNesteFormat@2000 

### 4.2 Guia dos passos que devem ser feito no IXC
<pre>
http://wiki.ixcsoft.com.br/index.php/CONCENTRADOR_HUAWEI
</pre>

### 4.3 Locais que o Servidor IXC realizará alteração ou inclusão no NE.

- radius-server coa-request
- radius-server
- radius-attribute
- radius-server group
- authentication-scheme
- accounting-scheme
- domain

## 5.0 - Incluir configuração POS-IXC
Depois que o IXC fez as configurações no equipamento, deve seguir incluindo o restante.

### 5.1 Configuração de um Domínio
<pre>
aaa
   domain NOME_DOMAIN_CRIADO_IXC
    ip-pool                 IPV4_WAN_POOL_PPPOE_01
    ipv6-pool               IPV6_WAN_POOL_PPPOE_01
    ipv6-pool               IPV6_PD_POOL_PPPOE_01
    dns primary-ip          177.152.85.218
    dns second-ip           8.8.8.8
    dns primary-ipv6        2804:dec:0:4::2
    dns second-ipv6         2001:4860:4860::8888
    accounting-start-delay 10 online user-type ppp ipoe static 
    user-basic-service-ip-type ipv4 
    ipv6 ppp assign-interfaceid
    qos rate-limit-mode user-queue inbound
    qos rate-limit-mode user-queue outbound
    user-max-session 65530
</pre>


### 5.1 Criar Virtual-Template 1 
<pre>
system-view
interface virtual-template 1
ip address unnumbered interface Loopback0
ppp keepalive interval 20 retransmit 3 datacheck
pppoe-server service-name-parameter NAS_BNG_01
pppoe-server ac-name NAS_BNG_01
ppp authentication-mode chap
ip urpf strict enable check subnet
ipv6 urpf strict enable check subnet
</pre>

### 5.3 Colocar o radius como source da loopback0
Importante evita problema de queda de conexão.
<pre>
radius-server source interface Loopback0
</pre>

## 6.0 Modelo/Exemplo configuração Interface
Modelos de exemplo para configurar uma interface sem vlan, com vlan e qinq.

### 6.1 Configuração sem VLAN
<pre>
interface gigabitethernet 0/X/X
description NOTEBOOK_CONECTADO_OU_AP
ipv6 enable
ipv6 address auto link-local
statistic enable mode port
8021p 0
pppoe-server bind virtual-template 1
commit
bas
access-type layer2-subscriber default-domain authentication XXXXX_NOME_DOMAIN_CRIADO_IXC
commit
</pre>

### 6.2 Configuração com VLAN de acesso 100
<pre>
interface gigabitethernet 0/X/X.25
description ID_100_OLT
ipv6 enable
ipv6 address auto link-local
statistic enable
commit
statistic enable
8021p 0
user-vlan 100
quit
pppoe-server bind virtual-template 1
commit
bas
access-type layer2-subscriber default-domain authentication XXXXX_NOME_DOMAIN_CRIADO_IXC
authentication-method ppp
quit
quit
commit
</pre>

### 6.3 Configuração com QINQ 25 
<pre>
interface Eth-TrunkX.25
 description QINQ_25_OLT_ALL_VLANS_2_4094
 ipv6 enable
 ipv6 address auto link-local
 commit
 statistic enable
 8021p 0
 user-vlan 2 4094 qinq 25
 commit
 pppoe-server bind Virtual-Template 1
 commit
 bas
 #
  access-type layer2-subscriber default-domain authentication XXXXX_NOME_DOMAIN_CRIADO_IXC
 commit
</pre>
