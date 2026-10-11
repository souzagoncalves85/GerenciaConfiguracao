# Configuração de acesso remoto SSH e Firewall UFW no Ubuntu 24.04 (Docker)

## 1. Objetivo

Este documento descreve, passo a passo, como preparar um ambiente Ubuntu 24.04 em um contêiner Docker sem presumir que o servidor SSH ou o UFW já estejam instalados. Ao final, será possível iniciar o serviço SSH, configurar regras básicas de firewall e testar a conexão.

> **Importante sobre Docker:** contêineres normalmente compartilham o kernel do host e não funcionam exatamente como uma máquina virtual. O UFW dentro de um contêiner pode não controlar o tráfego da maneira esperada, pois as regras de rede costumam ser gerenciadas pelo host Docker. Este roteiro serve para estudar instalação e configuração. Para proteger uma máquina real, configure o firewall no host ou em uma VM apropriada. Não exponha SSH à internet sem autenticação por chave e outras medidas de segurança.

## 2. Pré-requisitos

- Docker instalado e em funcionamento no computador.
- Terminal com acesso ao comando `docker`.
- Cliente SSH instalado no computador de onde será feita a conexão.
- Conexão de rede local, caso o teste seja feito a partir de outra máquina.

Confira o Docker:

```bash
docker --version
docker info
```

## 3. Baixar a imagem Ubuntu 24.04

Atualize a lista local de imagens e baixe a imagem oficial:

```bash
docker pull ubuntu:24.04
docker image ls
```

## 4. Criar o contêiner

Crie um contêiner com nome `ubuntu-ssh-lab`, publique a porta 2222 do host para a porta 22 do contêiner e mantenha um terminal interativo:

```bash
docker run -it --name ubuntu-ssh-lab -p 2222:22 ubuntu:24.04 bash
```

O comando acima abre um shell dentro do contêiner. A porta `2222` foi escolhida no host para evitar conflito com um eventual serviço SSH já instalado na máquina. Os próximos comandos desta seção devem ser executados **dentro do contêiner**.

Se o contêiner já tiver sido criado e estiver parado, inicie-o e entre nele em terminais separados:

```bash
docker start ubuntu-ssh-lab
docker exec -it ubuntu-ssh-lab bash
```

## 5. Preparar o Ubuntu e instalar os pacotes

Atualize o índice de pacotes:

```bash
apt update
```

Instale o servidor OpenSSH, o cliente SSH (útil para testes) e o UFW:

```bash
apt install -y openssh-server openssh-client ufw iproute2
```

Verifique se os programas estão disponíveis:

```bash
ssh -V
sshd -V
ufw --version
```

A opção `sshd -V` pode escrever a versão na saída de erro; isso é normal.

## 6. Configurar o servidor SSH

### 6.1 Criar o diretório de execução e verificar a configuração

```bash
mkdir -p /run/sshd
sshd -t
```

Se `sshd -t` não mostrar mensagens, a configuração passou na verificação sintática. Se houver erro, corrija-o antes de iniciar o serviço.

### 6.2 Criar um usuário comum para acesso remoto

Evite usar o usuário `root` para login remoto. Crie um usuário de laboratório e defina uma senha:

```bash
adduser aluno
```

Siga as perguntas do comando e defina uma senha de teste. Para um ambiente real, prefira autenticação por chave SSH.

### 6.3 Ajustar as opções essenciais

Faça uma cópia de segurança do arquivo de configuração:

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

Abra o arquivo com um editor, se disponível:

```bash
apt install -y nano
nano /etc/ssh/sshd_config
```

Confira ou ajuste estas diretivas, removendo o `#` inicial se necessário:

```text
Port 22
PermitRootLogin no
PasswordAuthentication yes
PubkeyAuthentication yes
```

`PasswordAuthentication yes` é usado aqui apenas para simplificar o laboratório. Em um servidor real, configure chaves públicas e, depois de testar o acesso por chave, considere desabilitar senhas. Não desabilite a autenticação por senha antes de confirmar que o acesso por chave funciona.

Salve e feche o editor. Valide novamente:

```bash
sshd -t
```

### 6.4 Iniciar o servidor SSH

Em imagens mínimas, o `systemd` geralmente não está ativo. Portanto, inicie o daemon diretamente:

```bash
/usr/sbin/sshd
```

Confira se a porta 22 está escutando:

```bash
ss -lntp | grep ':22'
```

Se não houver saída, confira os erros de configuração e a existência de `/run/sshd`.

> Se o contêiner for encerrado, o processo SSH também será encerrado. Para um laboratório persistente, é possível criar uma imagem própria com `Dockerfile` e um processo de inicialização apropriado; não se deve presumir que `systemctl` funcione dentro de um contêiner Ubuntu comum.

## 7. Configurar o UFW

### 7.1 Verificar o estado inicial

```bash
ufw status verbose
```

### 7.2 Definir políticas padrão e permitir SSH antes de ativar

Configure as políticas padrão e permita a porta SSH:

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
```

A regra `allow 22/tcp` é importante antes de ativar o UFW em uma máquina remota, para reduzir o risco de bloquear o próprio acesso SSH.

### 7.3 Ativar e verificar

```bash
ufw --force enable
ufw status numbered
ufw status verbose
```

Para remover uma regra, use o número apresentado por `ufw status numbered`. Por exemplo, confirme o número antes de executar:

```bash
ufw delete NUMERO_DA_REGRA
```

Substitua `NUMERO_DA_REGRA` pelo número real mostrado no terminal.

> **Limitação no Docker:** ativar o UFW dentro do contêiner pode falhar ou não filtrar o tráfego publicado pelo Docker, dependendo das capacidades e da configuração de rede. Não use a saída de `ufw status` como prova de que a porta publicada está protegida pelo UFW. Para avaliar as regras efetivas do host, use as ferramentas de firewall do próprio host e a documentação do Docker.

## 8. Testar o acesso SSH

### 8.1 Obter o endereço IP do contêiner

Dentro do contêiner:

```bash
hostname -I
ip addr
```

Quando o teste é feito no mesmo computador que executa o Docker e a porta foi publicada com `-p 2222:22`, conecte-se pela porta do host.

### 8.2 Conectar a partir do host

Abra outro terminal no computador host e execute:

```bash
ssh -p 2222 aluno@localhost
```

Digite a senha definida para `aluno`. Na primeira conexão, confirme a impressão digital do servidor somente se estiver esperando por essa conexão. Após entrar, confira o usuário e encerre a sessão:

```bash
whoami
exit
```

Para conectar de outra máquina na rede, substitua `localhost` pelo endereço IP do computador host que executa o Docker:

```bash
ssh -p 2222 aluno@IP_DO_HOST
```

O acesso externo depende do firewall do host, das regras de rede e de como o Docker foi iniciado.

## 9. Diagnóstico de problemas

### O cliente recebe “Connection refused”

Verifique se o daemon está ativo dentro do contêiner:

```bash
ss -lntp | grep ':22'
ps aux | grep '[s]shd'
```

Se necessário, inicie-o:

```bash
mkdir -p /run/sshd
/usr/sbin/sshd
```

### O cliente recebe “Permission denied”

- Confirme que o usuário `aluno` existe: `id aluno`.
- Verifique a senha e as diretivas de autenticação em `/etc/ssh/sshd_config`.
- Valide a configuração com `sshd -t`.
- Reinicie o daemon de forma controlada, encerrando o processo antigo se necessário e iniciando `/usr/sbin/sshd` novamente.

### O comando UFW falha

O UFW pode depender de recursos de rede/capacidades que não estão disponíveis no contêiner. Para um laboratório que exija demonstrar filtragem real, utilize uma VM Ubuntu 24.04 ou configure o firewall no host Docker. Evite adicionar permissões privilegiadas ao contêiner sem compreender o impacto de segurança.

### A porta não está acessível fora do computador host

- Confirme que o contêiner foi criado com `-p 2222:22`.
- No host, confira o mapeamento com `docker port ubuntu-ssh-lab`.
- Confira o firewall do host e as regras de rede.
- Não confunda a porta interna `22` do contêiner com a porta `2222` publicada no host.

## 10. Comandos úteis de consulta

Dentro do contêiner:

```bash
sshd -t
ss -lntp
ufw status verbose
id aluno
```

No host Docker:

```bash
docker ps
docker port ubuntu-ssh-lab
docker logs ubuntu-ssh-lab
```

## 11. Encerrar o laboratório

Saia da sessão SSH com `exit`. Para parar o contêiner, execute no host:

```bash
docker stop ubuntu-ssh-lab
```

Para iniciar novamente:

```bash
docker start ubuntu-ssh-lab
docker exec -it ubuntu-ssh-lab bash
```

Para apagar o contêiner (isso remove o sistema de arquivos gravável do contêiner):

```bash
docker rm -f ubuntu-ssh-lab
```

A imagem `ubuntu:24.04` permanece disponível localmente e pode ser usada para criar outro contêiner.

## 12. Publicar a documentação no GitHub

Dentro do repositório local `GerenciaConfiguracao`, salve este arquivo com o nome `README-SSH-UFW.md` ou incorpore o conteúdo ao `README.md`, conforme a orientação do professor.

Se o repositório já estiver clonado:

```bash
cd GerenciaConfiguracao
git status
git add README-SSH-UFW.md
git commit -m "Adiciona documentação de SSH e UFW no Ubuntu 24.04"
git push
```

Se ainda não clonou o repositório, copie a URL HTTPS exibida na página do projeto no GitHub e execute:

```bash
git clone URL_DO_REPOSITORIO
cd GerenciaConfiguracao
```

Depois crie/salve o arquivo, adicione-o, faça o commit e envie com `git push`. Substitua `URL_DO_REPOSITORIO` pela URL real; não publique credenciais, senhas ou chaves privadas.

## 13. Checklist final

- [ ] Imagem `ubuntu:24.04` baixada.
- [ ] Contêiner criado com publicação de porta `2222:22`.
- [ ] `openssh-server`, `openssh-client` e `ufw` instalados.
- [ ] Usuário comum `aluno` criado.
- [ ] Configuração SSH validada com `sshd -t`.
- [ ] Daemon SSH escutando na porta 22.
- [ ] Regra UFW para SSH adicionada antes de ativar o firewall.
- [ ] Teste de conexão executado com `ssh -p 2222 aluno@localhost`.
- [ ] Documentação commitada e enviada ao repositório GitHub.

## Referências

- Ubuntu Server — OpenSSH: https://ubuntu.com/server/docs/openssh-server
- Ubuntu Server — Firewall (UFW): https://ubuntu.com/server/docs/security-firewall
- Docker Docs — Port publishing: https://docs.docker.com/engine/network/port-publishing/
- Docker Docs — Networking: https://docs.docker.com/engine/network/
