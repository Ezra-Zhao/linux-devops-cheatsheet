# Linux / DevOps — Folha de referência em uma página 🇧🇷

Guia rápido de comandos para sysadmins, SREs e DevOps. Cada comando vem com uma explicação de uma linha e exemplos reais de diagnóstico.

> Convenção: `#` = executar como root (sudo). Caminhos e nomes entre `<>` devem ser substituídos.

---

## 1. Arquivos e diretórios

| Comando | Para que serve |
|---|---|
| `ls -lah` | Lista com tamanhos legíveis, arquivos ocultos e permissões |
| `tree -L 2 -h` | Árvore do diretório, 2 níveis, com tamanhos |
| `find /var/log -name "*.log" -mtime -7` | Arquivos `.log` modificados nos últimos 7 dias |
| `find / -xdev -size +500M -exec ls -lh {} \;` | Arquivos maiores que 500 MB (sem atravessar filesystems) |
| `stat <arquivo>` | Metadados completos: inode, permissões, datas de acesso/modificação |
| `cp -a origem destino` | Copia preservando permissões, links e datas (modo arquivo) |
| `mv -i origem destino` | Move/renomeia pedindo confirmação antes de sobrescrever |
| `rm -i <arquivo>` | Apaga pedindo confirmação (use sempre com curingas) |
| `touch -d "2024-01-01" <arquivo>` | Altera a data de um arquivo (útil em testes) |
| `realpath <arquivo>` | Resolve o caminho absoluto seguindo links simbólicos |
| `file <arquivo>` | Diz o tipo real do arquivo (não confia na extensão) |

🔧 **Diagnóstico 1 — O disco encheu e você não sabe com o quê:**
```bash
du -x --max-depth=2 / 2>/dev/null | sort -rn | head -20
# Desça até o diretório culpado e repita até achar o arquivo gigante.
```

🔧 **Diagnóstico 2 — Um arquivo apagado continua ocupando espaço:**
```bash
lsof +L1 | grep -i deleted
# O processo ainda mantém o descritor aberto. Reinicie o serviço ou faça
# : > /proc/<PID>/fd/<FD> para truncar sem reiniciar.
```

---

## 2. Processamento de texto: grep, sed, awk, jq

| Comando | Para que serve |
|---|---|
| `grep -RniE "error\|timeout" /var/log/app/` | Busca (recursiva, sem diferenciar maiúsculas, com nº da linha, regex estendida) |
| `grep -v "^#" nginx.conf` | Mostra o arquivo sem as linhas comentadas |
| `grep -A5 -B2 "OOM" syslog` | 5 linhas depois e 2 antes de cada ocorrência |
| `sed -n '100,200p' app.log` | Imprime só as linhas 100–200 |
| `sed -i 's/foo/bar/g' config.yaml` | Substitui no arquivo (faça backup antes!) |
| `awk '{print $1, $9}' access.log` | Imprime as colunas 1 e 9 (IPs e códigos HTTP) |
| `awk '$9 >= 500' access.log \| wc -l` | Conta as requisições com erro 5xx |
| `cut -d: -f1 /etc/passwd` | Primeira coluna separada por `:` (lista de usuários) |
| `sort access.log \| uniq -c \| sort -rn \| head` | Ranking de linhas repetidas (IPs mais ativos) |
| `jq '.status, .latency' resposta.json` | Extrai campos de um JSON |
| `jq -r '.[] \| "\(.ip) \(.code)"' dados.json` | Saída em texto puro, um registro por linha |
| `tr 'A-Z' 'a-z' < dados.txt` | Converte tudo para minúsculas |
| `wc -l <arquivo>` | Conta linhas (mais rápido que `cat \| wc`) |

🔧 **Diagnóstico 1 — Quais IPs estão batendo mais forte no servidor?**
```bash
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10
```

🔧 **Diagnóstico 2 — Erros 5xx por minuto na última hora:**
```bash
awk '$9 >= 500 {print substr($4, 2, 17)}' access.log | uniq -c
# $4 é [10/Oct/2026:08:15:33 ; o substr corta até o minuto. Picos = incidentes.
```

---

## 3. Permissões e usuários

| Comando | Para que serve |
|---|---|
| `chmod 640 <arquivo>` | rw para o dono, r para o grupo, nada para os outros |
| `chmod -R u+rwX,g+rX,o-rwx <dir>` | Permissões saudáveis numa árvore: X só em dirs/executáveis |
| `chown -R app:app /opt/app` | Troca dono e grupo recursivamente |
| `umask` / `umask 027` | Mostra/troca a máscara padrão de permissões |
| `id <usuário>` | UID, GID e grupos do usuário |
| `getent passwd <usuário>` | Info do usuário (funciona também com LDAP/SSSD) |
| `sudo -l` | O que seu usuário pode executar com sudo (auditoria rápida) |
| `useradd -m -s /bin/bash <usuário>` | Cria usuário com home e shell |
| `passwd -S <usuário>` / `passwd -l <usuário>` | Status da senha / bloqueia a conta |
| `last -n 20` | Últimos 20 logins (detecta acessos estranhos) |
| `ls -lZ <arquivo>` | Contexto SELinux (se der `Permission denied` sem motivo aparente) |

🔧 **Diagnóstico 1 — "Permission denied" mesmo com a permissão aparentemente certa:**
```bash
namei -l /opt/app/data/arquivo.log
# Mostra as permissões de CADA componente do caminho. Em 90% dos casos
# o problema está num diretório pai sem permissão x.
```

🔧 **Diagnóstico 2 — A aplicação não consegue escrever o log após o deploy:**
```bash
ls -ld /var/log/app && id app
# Compare o dono do diretório com o usuário do processo. Corrija com
# chown app:app /var/log/app ou ajuste o User= do serviço systemd.
```

---

## 4. Processos

| Comando | Para que serve |
|---|---|
| `ps aux --sort=-%mem \| head -15` | Top 15 processos por consumo de memória |
| `ps -ef | grep <nome>` | Procura um processo pelo nome (melhor: `pgrep -af`) |
| `pgrep -af python` | PIDs e linha completa dos processos python |
| `top` / `htop` | Monitor interativo (htop: mais visual, mata com F9) |
| `kill <PID>` | SIGTERM: pede ao processo para terminar de forma limpa |
| `kill -9 <PID>` | SIGKILL: mata sem avisar (último recurso, pode deixar locks) |
| `pkill -f "gunicorn.*worker"` | Mata por padrão da linha de comando |
| `lsof -i :8080` | Qual processo está ouvindo na porta 8080 |
| `lsof -p <PID>` | Arquivos e sockets abertos por um processo |
| `nice -n 10 <cmd>` / `renice -n 5 -p <PID>` | Diminui/aumenta a prioridade de CPU |
| `nohup <cmd> &` | Executa imune ao fechamento do terminal |
| `strace -p <PID> -e trace=open,read` | Vê as syscalls de um processo ao vivo (diagnóstico profundo) |

🔧 **Diagnóstico 1 — Uma porta "já está em uso" ao subir o serviço:**
```bash
ss -tlnp | grep 8080
# Mostra o PID exato. Decida: é um processo zumbi de um deploy
# anterior? Mate com kill <PID>, evitando o -9 se possível.
```

🔧 **Diagnóstico 2 — A máquina está lenta e você não sabe qual processo é:**
```bash
ps -eo pid,ppid,%cpu,%mem,cmd --sort=-%cpu | head -10
dmesg -T | grep -i "oom\|killed process" | tail -5
# Se o OOM killer agiu, você verá qual processo o kernel sacrificou.
```

---

## 5. systemd: serviços

| Comando | Para que serve |
|---|---|
| `systemctl status <serviço>` | Estado, últimos logs e PID do serviço |
| `systemctl restart <serviço>` | Reinicia o serviço |
| `systemctl enable --now <serviço>` | Habilita no boot E já inicia agora |
| `systemctl is-enabled <serviço>` | Inicia sozinho no boot? |
| `systemctl daemon-reload` | Recarrega a configuração após editar um unit |
| `systemctl list-units --failed` | Serviços que falharam (seu primeiro olhar após um reboot) |
| `systemctl list-timers --all` | Timers agendados (o "cron" do systemd) |
| `systemd-analyze blame | head` | O que está deixando o boot lento |
| `journalctl -u <serviço> -n 100 --no-pager` | Últimas 100 linhas do log do serviço |
| `journalctl -u <serviço> -f` | Acompanha o log ao vivo |
| `journalctl --since "30 min ago" -p err` | Erros de todo o sistema na última meia hora |

🔧 **Diagnóstico 1 — O serviço sobe e morre na hora:**
```bash
systemctl status minha-app -l --no-pager
journalctl -u minha-app -n 50 --no-pager
# Em 80% dos casos: caminho errado no ExecStart, permissão negada no
# WorkingDirectory, ou falta uma variável no EnvironmentFile.
```

🔧 **Diagnóstico 2 — Editou o .service e a mudança não pegou:**
```bash
systemctl daemon-reload && systemctl restart minha-app
# Sem daemon-reload, o systemd continua usando a definição antiga em memória.
```

---

## 6. Rede: diagnóstico

| Comando | Para que serve |
|---|---|
| `ss -tlnp` | Portas TCP em escuta com processo (o `netstat` moderno) |
| `ss -tupn` | TCP + UDP com processos |
| `ip -brief addr` | IPs de cada interface, uma linha por interface |
| `ip route` | Tabela de rotas (por onde sai o tráfego?) |
| `ping -c 4 <host>` | Conectividade básica (4 pacotes) |
| `mtr --report <host>` | Traceroute + perda por salto (ideal para abrir ticket com a operadora) |
| `curl -sSI https://<host>` | Só os cabeçalhos HTTP (responde? qual código?) |
| `curl -w "dns:%{time_namelookup} conn:%{time_connect} tls:%{time_appconnect} total:%{time_total}\n" -o /dev/null -s https://<host>` | Tempos detalhados: DNS, conexão, TLS, total |
| `dig +short <domínio>` | Resolução DNS rápida |
| `dig +trace <domínio>` | Traço completo desde a raiz (detecta delegações quebradas) |
| `host <domínio> 8.8.8.8` | Pergunta a um DNS específico (o problema é no meu DNS ou é geral?) |
| `nc -zv <host> <porta>` | Essa porta TCP está aberta? |
| `tcpdump -i eth0 -n port 443 -c 20` | Captura 20 pacotes da porta 443 (sem resolver nomes) |
| `getent hosts <domínio>` | Como o sistema resolve (respeita /etc/hosts e nsswitch) |

🔧 **Diagnóstico 1 — "A aplicação não alcança o banco de dados":**
```bash
nc -zv db-interna 5432        # porta aberta?
getent hosts db-interna       # o nome resolve?
ip route get <ip-do-banco>    # por qual interface/gateway sai?
# Se a porta está fechada mas o host responde: firewall ou serviço caído.
```

🔧 **Diagnóstico 2 — HTTPS lento, onde o tempo está se perdendo?**
```bash
curl -w "dns:%{time_namelookup} tcp:%{time_connect} tls:%{time_appconnect} srv:%{time_starttransfer}\n" -o /dev/null -s https://api.exemplo.com
# tls alto = handshake pesado ou rede com latência; srv alto = backend lento.
```

---

## 7. Disco e filesystems

| Comando | Para que serve |
|---|---|
| `df -hT` | Espaço por filesystem, com o tipo (ext4, xfs…) |
| `df -i` | Inodes livres (disco com espaço mas "cheio" = sem inodes) |
| `du -sh <dir>` | Tamanho total de um diretório |
| `lsblk -f` | Discos, partições e filesystems |
| `mount \| column -t` | O que está montado e com quais opções |
| `findmnt` | Árvore de montagens (mais legível que mount) |
| `dumpe2fs -h /dev/sda1 \| grep -i "mount count\|state"` | Estado do filesystem ext4 |
| `tune2fs -l /dev/sda1 \| grep -i reserved` | Blocos reservados para o root (5% por padrão) |
| `fstrim -av` | Libera blocos não usados em SSD (discard manual) |

🔧 **Diagnóstico 1 — O `df` marca 100% mas você apagou arquivos e continua cheio:**
```bash
lsof +L1 | grep deleted
# Arquivos apagados com descritores abertos. Trunque via /proc ou
# reinicie o serviço dono. O df só baixa quando eles fecham.
```

🔧 **Diagnóstico 2 — "No space left on device" com espaço livre:**
```bash
df -i /
# Sem inodes livres: milhões de arquivinhos (caches, sessões, filas).
# Ache o culpado: find /var/spool -xdev | head
```

---

## 8. Logs

| Comando | Para que serve |
|---|---|
| `tail -f /var/log/app.log` | Acompanha o log ao vivo |
| `tail -n 200 /var/log/app.log` | Últimas 200 linhas |
| `journalctl -f` | Acompanha o journal do sistema ao vivo |
| `journalctl --since today -p warning` | Avisos e erros de hoje |
| `journalctl -b -1 -p err` | Erros do boot anterior (após um travamento) |
| `grep -c "ERROR" app.log` | Conta erros (tendência rápida) |
| `awk '/ERROR/ {print $1, $2}' app.log | uniq -c` | Erros agrupados por hora |
| `logrotate --debug /etc/logrotate.conf` | Simula a rotação sem executar |
| `dmesg -T | tail -30` | Últimas mensagens do kernel com data legível |

🔧 **Diagnóstico 1 — O servidor reiniciou sozinho de madrugada:**
```bash
journalctl -b -1 -p err --no-pager | tail -30
last -x reboot shutdown | head
# Cruze as infos: houve OOM? kernel panic? alguém reiniciou (o last mostra o usuário)?
```

🔧 **Diagnóstico 2 — Precisa dos logs de ontem de um serviço:**
```bash
journalctl -u nginx --since yesterday --until today --no-pager | grep " 5[0-9][0-9] "
# --since/--until aceitam "yesterday", "-2h", "2026-10-04 08:00".
```

---

## 9. SSH, SCP e rsync

| Comando | Para que serve |
|---|---|
| `ssh -i ~/.ssh/chave.pem usuario@host` | Conecta com uma chave específica |
| `ssh -J bastion usuario@interna` | Pula pelo bastion (ProxyJump, sem túneis manuais) |
| `ssh -L 8080:localhost:8080 usuario@host` | Túnel: sua :8080 → :8080 remota |
| `ssh -N -f -L 5432:db:5432 usuario@host` | Túnel em background (banco de dados local) |
| `ssh-copy-id -i ~/.ssh/id_rsa.pub usuario@host` | Instala sua chave pública no servidor |
| `ssh-keygen -t ed25519 -C "meu-notebook"` | Gera chave moderna (ed25519, não RSA-1024) |
| `scp -r ./app usuario@host:/opt/` | Cópia recursiva via SSH |
| `rsync -avz --delete origem/ usuario@host:/destino/` | Sincroniza (comprime, verboso, apaga o que sobra no destino) |
| `rsync -avz --progress arquivo.iso usuario@host:/tmp/` | Cópia grande com barra de progresso |
| `rsync -avzn origem/ destino/` | Dry-run: mostra o que faria sem fazer |

🔧 **Diagnóstico 1 — O SSH pede senha mesmo com a chave configurada:**
```bash
ssh -v usuario@host   # veja quais métodos o servidor oferece
# Causas típicas: ~/.ssh com permissão 777 (deve ser 700, authorized_keys 600),
# ou o home do usuário gravável por outros (o StrictModes rejeita).
```

🔧 **Diagnóstico 2 — rsync entre servidores lentíssimo:**
```bash
rsync -av --progress origem/ usuario@host:/destino/
# Teste SEM -z: se a rede é rápida e os dados não comprimem (vídeos, zips),
# a compressão só adiciona CPU. E use --partial para retomar cortes.
```

---

## 10. Gerenciamento de pacotes

| Comando (Debian/Ubuntu) | Comando (RHEL/Rocky) | Para que serve |
|---|---|---|
| `apt update && apt upgrade -y` | `dnf upgrade -y` | Atualiza o sistema |
| `apt list --upgradable` | `dnf check-update` | O que pode ser atualizado |
| `apt install -y <pacote>` | `dnf install -y <pacote>` | Instala |
| `apt remove <pacote>` / `apt purge <pacote>` | `dnf remove <pacote>` | Desinstala (purge apaga configs) |
| `dpkg -l \| grep <pacote>` | `rpm -qa \| grep <pacote>` | Está instalado? Qual versão? |
| `dpkg -S /usr/bin/<bin>` | `rpm -qf /usr/bin/<bin>` | A qual pacote pertence este binário? |
| `dpkg -L <pacote>` | `rpm -ql <pacote>` | Quais arquivos o pacote instalou |
| `apt-cache policy <pacote>` | `dnf info <pacote>` | Versão instalada vs disponível, de qual repo vem |

🔧 **Diagnóstico 1 — Um binário se comportou estranho após atualizar:**
```bash
dpkg -S $(which nginx) && dpkg -l | grep nginx
# Confirma qual pacote o fornece e a versão. Se houver dois binários
# (ex. /usr/bin e /usr/local/bin), o which diz qual está rodando.
```

🔧 **Diagnóstico 2 — `apt update` falha com erros de GPG:**
```bash
apt update 2>&1 | grep -i "expired\|NO_PUBKEY"
# Chave expirada ou repo sem assinatura. Não use --allow-unauthenticated
# em produção: reimporte a chave oficial do fornecedor.
```

---

## 11. Docker: o essencial

| Comando | Para que serve |
|---|---|
| `docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"` | Containers em tabela legível |
| `docker logs -f --tail 100 <cont>` | Logs ao vivo, últimas 100 linhas |
| `docker exec -it <cont> bash` | Entra no container (sh se não houver bash) |
| `docker stats --no-stream` | CPU/RAM por container, um retrato |
| `docker inspect <cont> \| jq '.[0].NetworkSettings.IPAddress'` | IP do container |
| `docker compose up -d` / `docker compose down` | Sobe/desce a stack |
| `docker compose logs -f <svc>` | Logs de um serviço do compose |
| `docker system df` | Quanto disco usam imagens, containers e volumes |
| `docker system prune -a --volumes` | Limpeza agressiva (apaga volumes não usados!) |
| `docker cp <cont>:/app/log.txt ./` | Tira um arquivo de dentro do container |

🔧 **Diagnóstico 1 — O container sobe e apaga (Restarting):**
```bash
docker logs <cont> --tail 50
docker inspect <cont> --format '{{.State.ExitCode}} {{.Config.Cmd}}'
# ExitCode 1 + log de "connection refused" = subiu antes do banco.
# Solução de verdade: depends_on com healthcheck, não restart: always às cegas.
```

🔧 **Diagnóstico 2 — "No space left" e o culpado é o Docker:**
```bash
docker system df
# Quase sempre: camadas de builds antigos. Limpe com
# docker builder prune e docker image prune -a (revise antes o que apaga).
```

---

## 12. Performance: CPU, memória, I/O

| Comando | Para que serve |
|---|---|
| `uptime` | Carga média 1/5/15 min (compare com o nº de CPUs) |
| `nproc` | Nº de CPUs disponíveis |
| `vmstat 1 5` | Retrato de CPU, memória, swap e I/O a cada segundo |
| `free -h` | Memória legível (olhe "available", não "free") |
| `iostat -x 1 3` | Por dispositivo: %util, await, r/s w/s |
| `iotop -o` | Quais processos estão fazendo I/O agora |
| `sar -u 1 3` / `sar -r 1 3` | Histórico de CPU / memória (se o sysstat guarda dados) |
| `dmesg -T \| grep -i "error\|fail" \| tail` | Erros de hardware/kernel recentes |
| `cat /proc/loadavg` | Carga + processos rodando/na fila |
| `perf top` | Quais funções consomem CPU (quando o top não basta) |

🔧 **Diagnóstico 1 — Load average nas alturas mas CPU livre:**
```bash
vmstat 1 5
# Se a coluna 'b' (bloqueados) está alta e 'wa' (iowait) também: o gargalo
# é DISCO, não CPU. Continue com iostat -x para ver qual dispositivo satura.
```

🔧 **Diagnóstico 2 — É falta de RAM ou é normal?**
```bash
free -h
# No Linux, "buff/cache" alto é SAUDÁVEL (cache de disco). Preocupe-se se
# "available" cair abaixo de ~10% ou se "Swap used" crescer sem parar.
```

---

## 13. Segurança: o que você nunca deve fazer

| Regra | Por quê |
|---|---|
| ❌ `rm -rf /` ou `rm -rf $VAR/` com VAR vazia | Apaga o sistema. Use caminhos absolutos e `-i` |
| ❌ `chmod -R 777` | Dá escrita+execução para todo mundo |
| ❌ `curl http://... \| sudo bash` | Você executa código sem ler. Baixe, leia, depois execute |
| ❌ Senhas na linha de comando | Ficam no `history` e visíveis no `ps`. Use variáveis de ambiente ou arquivos com permissão 600 |
| ❌ SSH como root com senha | Só chave + `PermitRootLogin prohibit-password` |
| ✅ `sudo -i` e revise `/var/log/auth.log` | Auditoria de quem fez sudo e quando |
| ✅ `ufw allow 22/tcp && ufw enable` | Firewall mínimo viável num VPS novo |
| ✅ `fail2ban-client status sshd` | Está banindo IPs que forçam o SSH? |
| ✅ Atualize: `dnf upgrade -y` / `apt upgrade -y` | A maioria das invasões entra por software sem patch |

🔧 **Diagnóstico 1 — Suspeita de invasão:**
```bash
last -n 30                  # logins em horários estranhos ou de IPs esquisitas?
ss -tlnp | grep -v -E ":(22|80|443) "   # portas ouvindo que você não reconhece?
grep -i "failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head
# Ranking de IPs com falhas de SSH. Se uma tiver milhares: o fail2ban devia tê-la banido.
```

---

## Minicola de emergência 🚨

```bash
# A máquina não responde: o que está acontecendo?
uptime; free -h; df -h /; ss -tlnp | head; systemctl list-units --failed

# Um serviço caiu: o ritual
systemctl status <svc>; journalctl -u <svc> -n 50 --no-pager; ss -tlnp | grep <porta>

# Rede quebrada: o ritual
ping -c2 8.8.8.8; ip route; getent hosts <domínio>; curl -sSI https://<domínio> | head -3

# Disco cheio: o ritual
df -h; df -i; du -x --max-depth=2 / 2>/dev/null | sort -rn | head; lsof +L1 | grep deleted
```

---

*Conteúdo 100% original, escrito do zero pelo autor. Sem cópias de outros guias.*
