# Configuração de Acesso Remoto SSH e Firewall UFW no Ubuntu

## 1. Introdução

O principal objetivo é configurar e testar o acesso remoto por meio do protocolo SSH e implementar regras de segurança utilizando o UFW em um ambiente Ubuntu 24.04.

O teste foi realizado em um contêiner Docker baseado na imagem Ubuntu 24.04, permitindo configurar o servidor SSH, criar um usuário para acesso remoto, definir parâmetros de autenticação e estabelecer regras de entrada e saída de conexões.

Essa documentação contém os procedimentos executados, explica a finalidade dos principais comandos, registra os resultados dos testes e indica as fontes oficiais utilizadas como referência.

## 2. Ambiente utilizado

| Item | Configuração |
|---|---|
| Sistema operacional do contêiner | Ubuntu 24.04 |
| Ambiente de execução | Docker |
| Servidor SSH | OpenSSH Server |
| Firewall | UFW |
| Usuário criado | `gcsi` |
| Porta SSH no contêiner | TCP 22 |
| Porta publicada no hospedeiro | TCP 2222 |
| Nome do contêiner | `ubuntu2404` |
| Hostname do contêiner | `ubuntu24` |

A configuração de portas utilizada foi `2222:22`, em que a porta 2222 da máquina hospedeira é encaminhada para a porta 22 do contêiner.

**Fontes:** [Docker Docs — Port publishing](https://docs.docker.com/engine/network/port-publishing/) e [Ubuntu Server — OpenSSH](https://ubuntu.com/server/docs/how-to/security/openssh-server/).

## 3. Criação do ambiente Docker

O contêiner foi criado com a imagem oficial do Ubuntu 24.04 e com as capacidades necessárias para executar os procedimentos de rede e firewall.

```bash
docker run -dit \
  --name ubuntu2404 \
  --hostname ubuntu24 \
  --cap-add=NET_ADMIN \
  --cap-add=NET_RAW \
  -p 2222:22 \
  ubuntu:24.04 \
  bash -c "tail -f /dev/null"
```

### Explicação dos parâmetros

- `-dit`: executa o contêiner em segundo plano, com terminal interativo.
- `--name ubuntu2404`: define o nome do contêiner.
- `--hostname ubuntu24`: define o nome de máquina dentro do contêiner.
- `--cap-add=NET_ADMIN`: adiciona uma capacidade necessária para operações administrativas de rede.
- `--cap-add=NET_RAW`: permite determinadas operações de rede que utilizam sockets raw.
- `-p 2222:22`: publica a porta 22 do contêiner na porta 2222 do hospedeiro.
- `ubuntu:24.04`: especifica a imagem utilizada.
- `tail -f /dev/null`: mantém o contêiner em execução.

Para acessar o terminal do contêiner:

```bash
docker exec -it ubuntu2404 bash
```

Os comandos das próximas etapas foram executados dentro do contêiner.

**Fonte:** [Docker Docs — Port publishing](https://docs.docker.com/engine/network/port-publishing/).

## 4. Instalação dos pacotes necessários

```bash
apt update
```

Pacotes necessários para configurar o acesso remoto, o firewall e as ferramentas de rede:

```bash
apt install -y openssh-server ufw sudo iptables
apt install -y iproute2
```

### Finalidade dos pacotes

- `openssh-server`: fornece o servidor SSH responsável por receber conexões remotas.
- `ufw`: permite gerenciar regras do firewall de maneira simplificada.
- `sudo`: possibilita executar comandos administrativos com as permissões apropriadas.
- `iptables`: fornece ferramentas para consultar e administrar regras de filtragem de pacotes.
- `iproute2`: disponibiliza ferramentas de rede, incluindo o comando `ss`.

**Fontes:**

- [Ubuntu Server — OpenSSH](https://ubuntu.com/server/docs/how-to/security/openssh-server/)
- [Ubuntu Server — Firewalls](https://documentation.ubuntu.com/server/how-to/security/firewalls/)
- [Ubuntu Manpages](https://manpages.ubuntu.com/)

## 5. Criação e configuração do usuário

Foi criado o usuário `gcsi`, destinado ao acesso remoto. Também foi definido seu shell e concedida participação no grupo `sudo`.

```bash
mkdir -p /run/sshd

id gcsi || useradd -m -s /bin/bash gcsi

usermod -s /bin/bash gcsi

usermod -aG sudo gcsi

passwd gcsi
```

### Explicação dos comandos

- `mkdir -p /run/sshd`: cria o diretório de execução necessário ao funcionamento do serviço SSH, caso ele ainda não exista.
- `id gcsi`: verifica se o usuário já existe.
- `useradd -m -s /bin/bash gcsi`: cria o usuário com diretório pessoal e shell Bash, caso ainda não exista.
- `usermod -s /bin/bash gcsi`: define o Bash como shell do usuário.
- `usermod -aG sudo gcsi`: adiciona o usuário ao grupo `sudo`, preservando seus demais grupos.
- `passwd gcsi`: define a senha utilizada na autenticação.

**Referência:** [Ubuntu Manpages](https://manpages.ubuntu.com/).

## 6. Configuração do servidor SSH

Foi criado um arquivo específico para as configurações da atividade:

```bash
printf 'Port 22\nPermitRootLogin no\nPasswordAuthentication yes\n' \
  > /etc/ssh/sshd_config.d/99-atividade.conf
```

O arquivo contém as seguintes diretivas:

| Diretiva | Finalidade |
|---|---|
| `Port 22` | Define a porta de escuta do servidor SSH dentro do contêiner. |
| `PermitRootLogin no` | Impede o login SSH direto da conta `root`. |
| `PasswordAuthentication yes` | Permite autenticação por senha para usuários autorizados. |

A configuração permite utilizar o usuário `gcsi` para acesso remoto sem permitir o login direto como `root`.

### Validação da configuração

Iniciando o serviço:

```bash
service ssh start
```

Para verificar se o serviço está escutando na porta 22, também é possível utilizar:

```bash
ss -lntp
```

**Referências:**

- [Ubuntu Server — OpenSSH](https://ubuntu.com/server/docs/how-to/security/openssh-server/)
- [OpenBSD — Manual do sshd_config](https://man.openbsd.org/sshd_config.5)

## 7. Configuração do firewall UFW

O firewall foi configurado para bloquear conexões de entrada por padrão, permitir conexões de saída e liberar a porta SSH.

### 7.1 Definição das políticas padrão

```bash
ufw default deny incoming
ufw default allow outgoing
```

- `ufw default deny incoming`: define o bloqueio padrão das conexões de entrada.
- `ufw default allow outgoing`: permite conexões de saída por padrão.

Essas políticas seguem o princípio de restringir conexões de entrada que não tenham sido explicitamente autorizadas.

### 7.2 Liberação da porta SSH

```bash
ufw allow 22/tcp
```

Esse comando cria uma regra para permitir conexões TCP destinadas à porta 22 do contêiner.

A liberação é necessária para que o firewall não impeça o acesso remoto ao servidor SSH.

### 7.3 Ativação do firewall

```bash
ufw enable
```

O comando ativa o UFW e aplica as regras configuradas.

### 7.4 Verificação das regras

```bash
ufw status verbose
```

O resultado obtido apresentou o seguinte estado:

```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
```

O resultado demonstra que o UFW estava ativo, com bloqueio padrão das conexões de entrada, permissão padrão para conexões de saída e regras de liberação para SSH .

**Referência:** [Ubuntu Server — Firewalls](https://documentation.ubuntu.com/server/how-to/security/firewalls/).

## 8. Verificação das regras de firewall

Também foi utilizado o comando:

```bash
iptables -L
```

Esse comando lista as cadeias e regras de filtragem de pacotes do `iptables`.

**Referências:**

- [Ubuntu Server — Firewalls](https://documentation.ubuntu.com/server/how-to/security/firewalls/)
- [Ubuntu Manpages](https://manpages.ubuntu.com/)

## 9. Testes de acesso remoto SSH

Foram realizados testes para verificar a autenticação e o acesso ao ambiente configurado.

### 9.1 Teste dentro do contêiner

O primeiro teste foi executado dentro do próprio contêiner:

```bash
ssh gcsi@localhost
```

**Resultado:** conexão estabelecida com sucesso e sessão autenticada para o usuário `gcsi`.

### 9.2 Teste a partir da máquina hospedeira

Em seguida, foi testado o acesso a partir do terminal da máquina hospedeira:

```bash
ssh -p 2222 gcsi@localhost
```

O parâmetro `-p 2222` indica que o cliente deve utilizar a porta 2222 do hospedeiro, publicada pelo Docker e encaminhada para a porta 22 do contêiner.

**Resultado:** conexão SSH externa ao contêiner estabelecida com sucesso, inclusive após a ativação do UFW.

Esse teste confirma que o servidor SSH estava acessível pela porta publicada e que a regra de firewall permitia a conexão SSH.

## 10. Referências

CANONICAL. **OpenSSH server**. Ubuntu Server documentation. Disponível em: https://ubuntu.com/server/docs/how-to/security/openssh-server/. Acesso em: 10 out. 2026.

CANONICAL. **Firewalls**. Ubuntu Server documentation. Disponível em: https://documentation.ubuntu.com/server/how-to/security/firewalls/. Acesso em: 10 out. 2026.

OPENBSD PROJECT. **sshd_config(5) — OpenSSH daemon configuration file**. Disponível em: https://man.openbsd.org/sshd_config.5. Acesso em: 10 out. 2026.

DOCKER, INC. **Port publishing and mapping**. Docker Docs. Disponível em: https://docs.docker.com/engine/network/port-publishing/. Acesso em: 10 out. 2026.

CANONICAL. **Ubuntu Manpages**. Disponível em: https://manpages.ubuntu.com/. Acesso em: 10 out. 2026.