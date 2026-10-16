# Disco cheio por log nos firewalls Edge Protect

Documentação do incidente do **Município de Pirajuba** (14/10/2026), da correção aplicada à mão e do módulo
**`disco-logs`** do FT Onboard, que padronizou essa correção e a levou para os demais clientes.

| | |
|---|---|
| **Incidente** | Disco do `fw-municipiode-pirajuba` em 100%, derrubando o Wi-Fi da Prefeitura (chamado FT-2026-7103) |
| **Causa raiz** | O `logrotate` padrão do Edge Protect não rotaciona vários logs grandes; eles crescem até encher o disco |
| **Correção** | `logrotate` extra (`/etc/logrotate.d/noc-extra`) + `logrotate` do rsyslog mais curto + watchdog de disco (cron de hora em hora) |
| **Automação** | Módulo `modules/disco-logs` do FT Onboard (Verificar / Aplicar / Remover em vários firewalls) |
| **Situação** | Aplicado nos firewalls acessíveis em 15/10/2026; ver [Pendências](#7-pendências-e-próximos-passos) |

---

## 1. Resumo para a gestão

- Em 14/10/2026 o firewall do Município de Pirajuba ficou com o disco **100% cheio**. O `/var/log` tinha **180 GB**,
  principalmente `daemon.log` (51 GB) e `charon.log` (32 GB). Com o disco cheio, o firewall deixou de funcionar
  normalmente e as antenas Wi-Fi da Prefeitura e dos prédios anexos caíram.
- A causa **não foi um log isolado**: o Edge Protect só faz rotação de `syslog`, `mail.log`, `kern.log`, `auth.log`,
  `user.log` e `cron.log`, e ainda semanalmente. `daemon.log`, `charon.log`, `sudo.log`, `messages`, `iptables.log` e
  `debug` **nunca são rotacionados**. Isso é uma falha de configuração da imagem, então **não é exclusiva de Pirajuba**.
- Após a correção, o disco de Pirajuba voltou a **12%** (23 GB usados de 207 GB) e as antenas voltaram a subir.
- Para não depender de correção manual firewall a firewall, a correção virou o módulo **`disco-logs`** do FT Onboard.
  Em 15/10/2026 ele foi aplicado em **79 firewalls selecionados**: 70 foram alcançados por SSH e 9 não responderam.
- A aplicação em massa mostrou que existem **mais dois logs sem rotação** (`wf2.log`, com até 24,3 GB, e `dhcpd.log`, com
  até 6,1 GB) e que o **`sudo.log` está entre 3 e 10 GB em quase todos os firewalls**. O módulo
  foi atualizado para a **versão 2** e passou a cobrir os dois novos.
- Pontos em aberto: investigar por que o `charon.log` e o `sudo.log` crescem tanto (a rotação resolve o espaço, não a
  origem do volume) e reaplicar o módulo com o limite padrão. Detalhes na seção 7.

---

## 2. O que aconteceu em Pirajuba

**Sintoma.** O Wi-Fi da Prefeitura e dos prédios anexos parou (FT-2026-7103).

**Diagnóstico.** O disco do `fw-municipiode-pirajuba` estava em 100%. O `/var/log` ocupava 180 GB:

| Arquivo | Tamanho aproximado |
|---|---|
| `daemon.log` | 51 GB |
| `charon.log` | 32 GB |
| demais (`syslog`, `auth.log`, `sudo.log`, `messages`, `iptables.log`, `debug`) | não registrado por arquivo; somam o restante dos 180 GB |

**Causa raiz.** O arquivo `/etc/logrotate.d/rsyslog` do Edge Protect cobre apenas `syslog`, `mail.log`, `kern.log`,
`auth.log`, `user.log` e `cron.log`, com rotação **semanal** e **4 cópias**. Os demais logs não têm rotação nenhuma e
crescem sem limite. Mesmo os que têm rotação semanal podem chegar a dezenas de GB antes de a semana acabar.

**Causa do volume do `charon.log` (hipótese, ainda não confirmada).** No `ipsec status` e no `tail` do log apareceram
dezenas de IKE_SAs estabelecidas "há 40–50 minutos", uma atrás da outra, nos túneis `ipsec-matriz-para-cemeix1` e
`ipsec-matriz-para-cemeix2`. Isso indica reautenticação frequente (a cada ~45 min), cada uma gerando uma rajada de
linhas. É compatível com `ikelifetime` ou `keylife` curtos nessas conexões. Ainda precisa ser conferido no
`/etc/ipsec.conf` (ver seção 7). Esse ponto é **separado** do problema do túnel do Lazaro (FT-2026-6887), que é tratado
por outro watchdog existente apenas nesse firewall.

**Resolução no dia.** Limpeza dos logs, correção definitiva do `logrotate` (cobrindo os arquivos que não eram
rotacionados) e um watchdog de disco para evitar recorrência. O disco voltou a 12% e as antenas subiram.

---

## 3. Por que o Edge Protect enche o disco

| Log | Rotacionado pelo padrão do Edge Protect? |
|---|---|
| `syslog`, `mail.log`, `kern.log`, `auth.log`, `user.log`, `cron.log` | Sim, mas só **semanal**, com 4 cópias e **sem limite de tamanho** |
| `daemon.log` | Não |
| `charon.log` | Não |
| `sudo.log` | Não |
| `messages` | Não |
| `iptables.log` | Não |
| `debug` | Não |
| `wf2.log` (filtro de web) | Não (descoberto no rollout) |
| `dhcpd.log` | Não (descoberto no rollout) |

Qualquer firewall com muito tráfego, túnel instável (muita linha de IKE) ou muitos acessos `sudo` pode encher o disco
mais cedo ou mais tarde. O que muda de cliente para cliente é só **quanto tempo demora**.

---

## 4. A correção manual feita em Pirajuba

Esta é a correção como foi aplicada no dia, mantida aqui como registro histórico. **Para os demais firewalls use o
módulo da seção 5**, que corrige os pontos fracos listados mais abaixo.

**Passo 1: logrotate para os logs que faltavam.** `copytruncate` é essencial: o `charon` e o syslog mantêm o arquivo
aberto, e sem isso eles continuam escrevendo no arquivo antigo e o espaço nunca é liberado.

```bash
cat > /etc/logrotate.d/noc-extra << 'EOF'
/var/log/daemon.log
/var/log/charon.log
/var/log/sudo.log
/var/log/messages
/var/log/iptables.log
/var/log/debug
{
        daily
        rotate 5
        size 200M
        missingok
        notifempty
        compress
        delaycompress
        copytruncate
}
EOF
```

**Passo 2: rsyslog mais curto** (de semanal com 4 cópias para diário com 5 cópias e limite de tamanho).

```bash
sed -i 's/weekly/daily/; s/rotate 4/rotate 5/' /etc/logrotate.d/rsyslog
sed -i '/rotate 5/a\        size 200M' /etc/logrotate.d/rsyslog
logrotate -f /etc/logrotate.d/noc-extra
logrotate -f /etc/logrotate.d/rsyslog
```

**Passo 3: watchdog de disco** (rede de segurança entre uma rotação e outra).

```bash
cat > /usr/local/bin/disk-watchdog.sh << 'EOF'
#!/bin/bash
LIMIT=85
USO=$(df / --output=pcent | tail -1 | tr -dc '0-9')

if [ "$USO" -ge "$LIMIT" ]; then
    for LOG in /var/log/daemon.log /var/log/charon.log /var/log/syslog /var/log/auth.log /var/log/sudo.log /var/log/messages /var/log/iptables.log /var/log/debug; do
        if [ -f "$LOG" ] && [ "$(stat -c%s "$LOG")" -gt 524288000 ]; then
            : > "$LOG"
            logger -t disk-watchdog "Truncado $LOG por disco em ${USO}%"
        fi
    done
    journalctl --vacuum-size=200M
fi
EOF
chmod +x /usr/local/bin/disk-watchdog.sh

cat > /etc/cron.d/disk-watchdog << 'EOF'
0 * * * * root /usr/local/bin/disk-watchdog.sh
EOF
```

O watchdog só age se o disco passar de 85% **e** o arquivo passar de 500 MB, para não truncar log à toa.

---

## 5. O módulo `disco-logs` do FT Onboard

Local no repositório: `modules/disco-logs/` (`module.json` e `script.sh`). Versão atual: **2**.
No app: **Executar → Disco cheio por log (logrotate + watchdog)**.

### 5.1 O que ele faz

| Ação | O que acontece |
|---|---|
| **Verificar** | Só lê o estado, não altera nada. Mostra uso do disco, logs com mais de 1 GB, quais logs estão sem rotação, estado do `logrotate` do rsyslog e do watchdog (e se o serviço `cron` está rodando). |
| **Aplicar** | Cria `/etc/logrotate.d/noc-extra` só para os logs que **nenhum outro arquivo do logrotate já cobre**; aperta `/etc/logrotate.d/rsyslog` (opcional, guarda a original); instala o watchdog (`/usr/local/bin/disk-watchdog.sh` + `/etc/cron.d/disk-watchdog`); confere o resultado no fim. Pode ser repetido sem duplicar nada. |
| **Remover** | Tira o watchdog, o cron e o `noc-extra` e devolve o rsyslog original. **Não mexe em nenhum log.** |

Logs cobertos (versão 2): `daemon.log`, `charon.log`, `sudo.log`, `messages`, `iptables.log`, `debug`, `wf2.log`,
`dhcpd.log`.

### 5.2 Parâmetros

| Parâmetro | Padrão | Para que serve |
|---|---|---|
| Limite do disco (`LIMIT`) | `85` | O watchdog só zera logs grandes quando o `/` passa desse percentual. O Verificar usa o mesmo valor para marcar falha. |
| Tamanho máximo por log (`SIZE`) | `200M` | Rotaciona o log ao passar disso, além da rotação diária. |
| Apertar o logrotate do rsyslog (`TIGHTEN_RSYSLOG`) | sim | Troca `weekly` por `daily`, `rotate 4` por `rotate 5` e limita o tamanho. Original em `/var/backups/ft-onboard/`. |

### 5.3 Como usar

1. Abra **Executar** e escolha **Disco cheio por log**.
2. Marque **Verificar**, selecione os firewalls e rode. Não altera nada.
3. Nos que ficarem com aviso, rode **Aplicar** com o limite **85** (padrão).
4. Para desfazer, rode **Remover**.

### 5.4 O que o Verificar reporta

| Verificação | Resultado |
|---|---|
| Uso do disco `/` | Falha se o uso for maior ou igual ao limite informado |
| Logs com mais de 1 GB | Aviso, lista os maiores (inclusive cópias já rotacionadas, como `sudo.log.1`) |
| Logs extras com rotação | Aviso se algum log conhecido está sem rotação |
| Logrotate do rsyslog | Aviso se ainda é semanal e sem limite de tamanho |
| Watchdog de disco | Aviso se não está instalado, está incompleto ou o `cron` não roda |

### 5.5 Diferenças em relação ao script manual de Pirajuba

| Ponto | Manual (Pirajuba) | Módulo |
|---|---|---|
| Limite de tamanho | `size 200M`, que faz o logrotate **ignorar o `daily`** e rotacionar só por tamanho | `maxsize 200M`, que mantém o diário **e** rotaciona se passar do limite |
| Logs que já têm rotação em outro arquivo | Duplicaria a configuração (o logrotate dá erro) | Detecta e **não duplica** |
| Validação | Nenhuma | Testa com `logrotate -d` antes de gravar; se recusar, não instala |
| Rotação forçada (`-f`) | Usada | **Não usa.** Um log de 32 GB com `copytruncate` precisa de espaço livre igual para a cópia e poderia piorar um disco cheio. O watchdog zera os logs acima de 500 MB e só depois roda o logrotate por tamanho |
| rsyslog | Alterado sem backup | Guarda a original e permite voltar com **Remover** |
| Watchdog | Trunca os logs grandes e enxuga o journal | Faz o mesmo e, **em seguida**, roda o `noc-extra` por tamanho (o logrotate padrão roda só uma vez por dia) |
| Cron | Sem proteção | `flock` evita execuções sobrepostas; `PATH` definido (o cron não enxerga `/usr/sbin`) |
| Linha `su` do logrotate | Não tratada | O `noc-extra` herda a linha `su` global |
| Logs cobertos | 6 | 8 (inclui `wf2.log` e `dhcpd.log`) |

### 5.6 Cuidados

- O watchdog **apaga o conteúdo** (zera) dos logs com mais de 500 MB quando o disco passa do limite. É uma rede de
  segurança para não derrubar o firewall, mas significa perder esse histórico. Por isso o limite padrão é **85%**.
  Um limite muito baixo (por exemplo 50%) faz o watchdog zerar logs em firewalls que estão saudáveis.
- O `copytruncate` pode, em teoria, perder algumas linhas escritas entre a cópia e o truncamento. É o preço de não
  precisar reiniciar o serviço que escreve o log.
- O onboarding configura a leitura do `sudo.log` pelo agente Zabbix. Depois de aplicar o módulo, vale confirmar em
  um firewall que o item do `sudo.log` continua recebendo dados após a rotação.
- A rotação resolve o **espaço**, não a **origem do volume**. Log crescendo demais é sintoma de algo (túnel
  reautenticando, comando `sudo` em laço, filtro web muito verboso) e deve ser investigado.

---

## 6. Aplicação nos demais clientes (15/10/2026)

O módulo foi executado em **79 firewalls** pelo FT Onboard (tela Executar, ação Aplicar).

| Resultado | Quantidade | Significado |
|---|---|---|
| Certo | 13 | Instalado e sem log grande |
| Aviso | 51 | Instalado, mas ainda existem logs **que já estavam gigantes** antes da aplicação |
| Falhou | 6 | Disco acima do limite informado na execução (ver observação abaixo) |
| Erro | 9 | SSH sem resposta (não é falha do módulo) |

**Por que tantos avisos.** O Aplicar não força a rotação (seção 5.5). Os logs que já estavam grandes continuam até a
rotação por tamanho do cron horário e do logrotate diário. As cópias `.1` só são compactadas na rotação seguinte
(`delaycompress`). Rodar **Verificar** no dia seguinte deve mostrar a melhora.

**Por que 6 falhas.** Essa primeira execução usou o limite **50%** em vez do padrão 85%. Firewalls com disco entre 51% e
69% foram marcados como falha. Não eram erros reais. Isso também deixou o watchdog instalado com limite 50% nos
firewalls em que foi aplicado, o que é agressivo demais (ver seção 5.6). **Reaplicar com 85%** corrige.

**Achados da execução em massa**

| Achado | Detalhe | Ação |
|---|---|---|
| `sudo.log` muito grande | Entre 3 e 10 GB em quase todos os firewalls | Investigar quem gera tantas linhas de `sudo` (seção 7) |
| `wf2.log` sem rotação | Até 24,3 GB (maior caso: Santa Vitória) | Incluído no módulo v2 |
| `dhcpd.log` sem rotação | Até 6,1 GB | Incluído no módulo v2 |
| `syslog.1` e `auth.log.1` com ~9 GB | Sobras da rotação antiga do rsyslog (Agroverde Matriz e Pirajuba) | Compactadas na próxima rotação |
| 9 firewalls sem SSH | Agroleste, Nova Ponte, Classe A, Lazaro, FAMA, Luz do Mundo, Milenium, Miro, Transnorato | Tratar o acesso SSH separadamente |

---

## 7. Pendências e próximos passos

1. **Reaplicar o módulo v2 com limite 85%** nos firewalls que receberam a primeira aplicação com 50% (Executar →
   Aplicar → limite `85`).
2. **Rodar Verificar de novo** (1 a 2 horas ou no dia seguinte) e conferir que os logs gigantes foram rotacionados.
3. **Descobrir a origem do `sudo.log` grande.** Num firewall afetado:
   ```bash
   tail -3 /var/log/sudo.log
   tail -2000 /var/log/sudo.log | grep -oE "USER=[a-z0-9_-]+|COMMAND=[^ ]+" | sort | uniq -c | sort -rn | head
   ```
   Se for algum usuário de serviço (por exemplo o agente de monitoramento) executando `sudo` o tempo todo, dá para
   reduzir o registro na origem. Conferir antes de alterar qualquer coisa.
4. **Confirmar a causa do `charon.log` em Pirajuba.** Se `ikelifetime` ou `keylife` estiverem curtos nos túneis dos
   Cemei, aumentá-los reduz a reautenticação e o volume de log:
   ```bash
   grep -A 15 "conn ipsec-matriz-para-cemei" /etc/ipsec.conf
   ```
5. **Tratar os 9 firewalls sem SSH** e aplicar o módulo neles depois.
6. **Pirajuba:** os arquivos criados à mão não tinham a marca do módulo. O **Aplicar** do app os substitui pela versão
   padronizada; conferir com **Verificar** que o estado ficou como "certo".

---

## 8. Como conferir direto no firewall (sem o app)

```bash
df -h /                                                  # uso do disco
du -ah /var/log 2>/dev/null | sort -rh | head -10        # maiores arquivos
cat /etc/logrotate.d/noc-extra                           # logs cobertos pela rotação extra
logrotate -d /etc/logrotate.d/noc-extra                  # simulação, não altera nada
cat /etc/cron.d/disk-watchdog                            # watchdog agendado
grep -m1 '^LIMIT=' /usr/local/bin/disk-watchdog.sh       # limite instalado
journalctl -t disk-watchdog --no-pager | tail            # quando o watchdog agiu
systemctl is-active cron                                 # o cron precisa estar ativo
```

---

## 9. Referências

- Chamado **FT-2026-7103**: Wi-Fi da Prefeitura de Pirajuba fora por disco cheio (fechado).
- Chamado **FT-2026-6887**: queda do IPsec do Lazaro em Pirajuba (assunto separado, tratado por outro watchdog).
- Código: `modules/disco-logs/module.json` e `modules/disco-logs/script.sh` (FT Onboard, repositório `Laurentx1/fw-onboard`).
- O módulo foi testado com o `logrotate` real (check, apply, remove, repetição sem duplicar, rejeição de configuração
  inválida e restauração do rsyslog) antes de ir para os firewalls.
