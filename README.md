# NFS Most Wanted (2005) em split-screen no Linux com PartyDeck

Tentativa de rodar o **Need for Speed: Most Wanted (2005, PC, patch 1.3)** em multiplayer local (duas instâncias no mesmo PC, ligadas por LAN) no **Linux**, usando o [PartyDeck](https://github.com/partydeck/partydeck), já que o [Nucleus Co-op](https://www.splitscreen.me/) só funciona no Windows.

> **Status: parcialmente funcionando.**
> O lobby LAN funciona (as duas instâncias se enxergam e entram na mesma sala). **A corrida não inicia:** o host fica em "Waiting for players to connect" e o jogador 2 fica carregando. A causa mais provável está documentada abaixo, mas **o problema ainda não foi resolvido**.

Este repositório documenta o que foi descoberto, o que funciona, o que foi descartado e onde a ajuda é bem-vinda.

---

## Ambiente de teste

| Item | Valor |
|---|---|
| Distro | CachyOS (Arch), KDE Plasma 6 (Wayland) |
| Jogo | NFS Most Wanted 2005 **Black Edition**, patch **1.3** |
| Launcher | PartyDeck v0.8.7 (tar.gz, SteamOS x86_64) |
| Runtime | GE-Proton via `umu-run` |
| GPU | AMD Radeon RX 5700 XT |

> Você precisa da **sua própria cópia legítima** do jogo, com o patch 1.3 oficial da EA. Este repositório não distribui o jogo nem arquivos de terceiros.

---

## Como o jogo faz multiplayer (o que o handler do Nucleus ensina)

O jogo de PC não tem split-screen. Ele tem **LAN** de 2 a 4 jogadores. O Nucleus Co-op funciona abrindo várias instâncias e ligando-as por LAN. Lendo o handler do Nucleus (`handler.js`, versão 10 do hub splitscreen.me), os pontos importantes são:

1. **Várias instâncias:** o mod *NFSMW Extra Options* (`NFSMWExtraOptions.asi`, carregado por um ASI loader `winmm.dll`) com `AllowMultipleInstances = 1`.
2. **Cada instância precisa do próprio `server.dll` e `server.cfg`.** O handler faz cópia real do `server.dll` (não link) e dá a cada instância uma `PORT` diferente.
3. **Modo janela** (`WindowedMode=2` no ini do Extra Options) e resolução ajustada.
4. **Fluxo manual no jogo:**
   - Instância 1: *LAN → Create Server* (nome = nome do jogador) → *Create Room*.
   - Instância 2: *LAN → Create Server* → procurar a sala e *Join* (pode precisar de mais de uma tentativa).

---

## O que funciona

### Passo 1: o jogo roda sozinho no Linux
Funciona com GE-Proton (via `umu-run`) e um prefixo Wine próprio.

### Passo 2: duas instâncias na **mesma pasta** travam
No PartyDeck, as duas instâncias usam a mesma pasta do jogo. Ao criar o servidor LAN na segunda instância, o jogo **fecha**. Com o log do Proton ligado (`PROTON_LOG=1`) aparece:

```
wine: Unhandled page fault on read access to 00000050 at address 03E39D56
```

O endereço está dentro do `server.dll` (carregado em `03E00000`). O mesmo fechamento aconteceu rodando uma instância fora do PartyDeck com outra já aberta na mesma pasta, então **a causa não é o PartyDeck, o gamescope nem os controles**.

### Passo 3: pastas separadas resolvem o lobby
Rodar cada instância em uma **pasta própria**, com `server.dll` e `server.cfg` próprios e portas diferentes, resolveu o fechamento.

Crie as pastas (use `bash`; no `fish` esses comandos dão erro). Ajuste `SRC` para a pasta do seu jogo:

```bash
SRC="$HOME/Games/Need For Speed Most Wanted Black Edition"
IP=$(hostname -I | awk '{print $1}')
for n in 1 2; do
  D="$HOME/Games/NFSMW-p$n"
  rm -rf "$D"; mkdir -p "$D"
  cp -as "$SRC"/. "$D"/                 # links simbólicos para o resto do jogo
  rm -f "$D/server.dll" "$D/server.cfg"
  cp "$SRC/server.dll" "$D/server.dll"  # cópia REAL
  grep -vE '^(PORT|LAN_THROTTLE|ADDR|TRUST_MATCH)=' "$SRC/server.cfg" > "$D/server.cfg"
  p=$((9899+n))                          # 9900 e 9901
  printf 'PORT=%s\nLAN_THROTTLE=6\nADDR=%%%%bind("%s:%s")\nTRUST_MATCH=%%%%bind("%s")\n' "$p" "$IP" "$p" "$IP" >> "$D/server.cfg"
done
```

Cada `server.cfg` termina com algo assim (a instância 2 usa a porta 9901):

```
PORT=9900
LAN_THROTTLE=6
ADDR=%%bind("192.168.x.x:9900")
TRUST_MATCH=%%bind("192.168.x.x")
```

> Se o `server.cfg` original tiver uma linha `ADDR=` ativa (padrão da EA), comente-a com `#`.

### Passo 4: o PartyDeck escolhendo a pasta certa
O handler do PartyDeck aponta para **uma** pasta de jogo (`path_gameroot`). A solução foi usar um `.bat` como executável, que escolhe `NFSMW-p1` ou `NFSMW-p2` conforme a ordem de abertura, usando uma pasta de trava (`md` só consegue criar uma vez):

`launch.bat` (na pasta original do jogo):

```bat
@echo off
set "BASE=Z:\home\SEU_USUARIO\Games"
md "%BASE%\.nfsmw-slot1" 2>nul
if not errorlevel 1 (set "SLOT=1") else (set "SLOT=2")
cd /d "%BASE%\NFSMW-p%SLOT%"
start /wait "" speed.exe
if "%SLOT%"=="1" rd "%BASE%\.nfsmw-slot1"
```

(O arquivo precisa ter finais de linha CRLF.) No PartyDeck, edite o handler e defina o **Executable file** como `launch.bat`.

Handler (`handler.json`), campos relevantes:

```json
{
  "name": "Need for Speed Most Wanted",
  "version": "1.3",
  "spec_ver": 3,
  "path_gameroot": "/home/SEU_USUARIO/Games/Need For Speed Most Wanted Black Edition",
  "exec": "launch.bat",
  "use_goldberg": false
}
```

**Resultado:** as duas instâncias abrem em split-screen e entram na mesma sala ("game room"). 

**Cuidado com a trava:** se o PartyDeck fechar de forma brusca, a pasta `.nfsmw-slot1` pode ficar para trás e as duas instâncias vão para a mesma pasta (e uma cai). Antes de abrir de novo: `rmdir ~/Games/.nfsmw-slot1`.

---

## O problema em aberto: a corrida não inicia

Com as duas na sala, ao iniciar a corrida o host continua em "Waiting for players to connect" e o jogador 2 fica carregando.

### Evidências coletadas

**1. A porta UDP 3658 é disputada.** Com `PROTON_LOG=1` e `WINEDEBUG=+winsock`, o log da instância 1 mostra:

```
bind socket 0x23c, addr { family AF_INET, address 0.0.0.0, port 3658 }
bind failed, status 0xc0000043
```

O jogo continua mesmo assim, enviando por um socket sem porta definida.

**2. A porta muda de dono.** Com `ss -tulnp`, antes de iniciar a corrida a 3658 pertencia à instância 2. Depois de iniciar, passou para a instância 1, e a instância 2 ficou sem ela.

**3. Tráfego de rede (`tcpdump`).** Durante o carregamento aparece quase só isto, em loop, na interface `lo`:

```
IP 192.168.x.x.3658 > 192.168.x.x.3658: UDP, length 21
```

Ou seja, pacotes que saem da 3658 e chegam na própria 3658. Não aparece tráfego UDP indo de uma instância para a outra. Os broadcasts de descoberta (`255.255.255.255:9999`, 384 bytes) funcionam e é por eles que o lobby se enxerga.

### Hipótese principal (não confirmada)
Durante a corrida, o jogo usa a **porta UDP 3658** e identifica os outros jogadores pelo **IP**. Como as duas instâncias dividem o mesmo IP e disputam a mesma porta, os pacotes do host sempre caem na mesma instância, e a outra nunca recebe nada.

Isso é consistente com relatos sobre o Wine: o comportamento de portas UDP compartilhadas é diferente do Windows (por exemplo, o bug 51097 do WineHQ descreve que só o último socket ligado recebe os pacotes). Também lembra um caso antigo do Warcraft III com duas cópias na mesma máquina, em que o `bind` falhava e a solução foi interceptá-lo e ativar `SO_REUSEADDR`.

### O que já foi descartado
| Hipótese | Teste | Resultado |
|---|---|---|
| Porta 9999 em conflito | análise do `ss` e do `tcpdump` | É só broadcast de descoberta, funciona |
| Firewall (UFW) | regra liberando UDP 9999 na LAN | Sem mudança |
| Servidor `demangler.ea.com` pendurando a conexão | bloquear o host em `/etc/hosts` | Sem mudança |
| Teclado/controles embaralhados | um controle por instância | Não é a causa do travamento |
| `server.dll`/`server.cfg` compartilhados | pastas separadas | Resolveu o **lobby**, não a corrida |

---

## Caminhos possíveis (ajuda bem-vinda)

1. **Dois computadores na mesma rede.** Cada um tem seu IP, a porta 3658 não conflita. Deve funcionar, pois é o que o jogo foi feito para fazer. (Ainda não testado pelo autor.)
2. **Plugin `.asi` que intercepta o `bind`.** O jogo já carrega plugins `.asi` (via ASI loader). Um plugin poderia fazer cada instância usar um endereço próprio (por exemplo `127.0.0.1` e `127.0.0.2`) ou ativar o compartilhamento de porta. Como o servidor LAN do jogo também usa endereços, isso pode exigir mexer em vários pontos. **Não testado.**
3. **Rede virtual por instância** (network namespaces). Daria a cada instância um IP próprio, mas é invasivo e pode quebrar a comunicação com o gamescope. **Não recomendado como primeira tentativa.**

Se você conhece o protocolo do Most Wanted (porta 3658, "demangler"), sabe mexer no `bind` pelo Wine/Proton, ou já fez isso funcionar no Linux, abra uma *issue* ou *pull request*.

---

## Como reproduzir os diagnósticos

Log de rede do Wine para um arquivo, rodando fora do PartyDeck (um terminal por instância):

```bash
mkdir -p ~/nfslogs/p1 ~/nfslogs/p2

# instância 1
cd ~/Games/NFSMW-p1 && env WINEPREFIX=$HOME/.local/share/partydeck/prefixes/1 \
  PROTONPATH=GE-Proton PROTON_LOG=1 PROTON_LOG_DIR=$HOME/nfslogs/p1 \
  WINEDEBUG=+winsock umu-run speed.exe

# instância 2
cd ~/Games/NFSMW-p2 && env WINEPREFIX=$HOME/.local/share/partydeck/prefixes/2 \
  PROTONPATH=GE-Proton PROTON_LOG=1 PROTON_LOG_DIR=$HOME/nfslogs/p2 \
  WINEDEBUG=+winsock umu-run speed.exe
```

Procurar falhas de porta:

```bash
grep -inE "bind|10048|ADDRINUSE|3658" ~/nfslogs/p1/steam-default.log | head
```

Portas abertas e tráfego UDP:

```bash
ss -tulnp | grep -iE 'speed|wine'
sudo tcpdump -ni any 'udp and not port 53 and not port 5353 and not port 1900 and not port 67 and not port 68' -c 60
```

---

## Observações sobre o PartyDeck

- Com `disable_mount_gamedirs: true` (ativado nesta configuração), as pastas `overlay` e os saves por perfil dentro da pasta do jogo **não** são aplicados, segundo as notas de versão do PartyDeck. Por isso foi necessário o `launch.bat`.
- Os handlers `.pd2` do PartyDeck são arquivos zip com um `handler.json` e pastas opcionais `overlay/` e `profile_copy_gamesave/`.
- O teclado e o mouse são enviados às duas instâncias ao mesmo tempo. Use **um controle por jogador**.

---

## Créditos

- **PartyDeck**: [partydeck/partydeck](https://github.com/partydeck/partydeck) (autor: wunnr).
- **Handler do Nucleus Co-op** para o Most Wanted (usado como referência): mantido por Talos91 no [Game Handlers Hub](https://hub.splitscreen.me/).
- **NFSMW Extra Options** e **WidescreenFix** (mods do jogo): ThirteenAG e colaboradores.
- **Nucleus Co-op**: Lucas Assis e a comunidade.

Este projeto é um estudo de compatibilidade e não tem relação com a EA nem com os autores acima.
