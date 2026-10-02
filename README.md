# Personnalisation d'une VM Ubuntu Server

Procédure complète pour transformer une installation Ubuntu Server 24.04 LTS toute neuve en poste d'exploitation confortable : **dashboard de connexion SSH**, **fastfetch**, **alias et fonctions documentés**, **prompt amélioré** et **bases de sécurité**.


---

## Sommaire

1. [Résultat attendu](#1-résultat-attendu)
2. [Prérequis](#2-prérequis)
3. [Étape 0 : préparation du système](#3-étape-0--préparation-du-système)
4. [Étape 1 : installer fastfetch](#4-étape-1--installer-fastfetch)
5. [Étape 2 : alias et fonctions](#5-étape-2--alias-et-fonctions)
6. [Étape 3 : la commande `aliases`](#6-étape-3--la-commande-aliases)
7. [Étape 4 : prompt, historique et `.bashrc`](#7-étape-4--prompt-historique-et-bashrc)
8. [Étape 5 : configuration fastfetch](#8-étape-5--configuration-fastfetch)
9. [Étape 6 : dashboard SSH (MOTD)](#9-étape-6--dashboard-ssh-motd)
10. [Étape 7 : sécurité de base](#10-étape-7--sécurité-de-base)
11. [Utilisation au quotidien](#11-utilisation-au-quotidien)
12. [Dépannage](#12-dépannage)
13. [Annuler la personnalisation](#13-annuler-la-personnalisation)

---

## 1. Résultat attendu

| Contexte | Ce qui s'affiche |
|---|---|
| Connexion **SSH** | Dashboard : système, réseau, sécurité, conteneurs, maintenance, certificats, raccourcis |
| Terminal **local** (console VM) | fastfetch compact + liste des raccourcis favoris |
| Ligne de commande | Prompt `heure user@hôte dossier (branche git) ❯`, flèche rouge si la dernière commande a échoué |
| Raccourcis | `off`, `rb`, `maj`, `ports`, `board`... documentés et listés par la commande `aliases` |

Le dashboard signale en couleur (vert / jaune / rouge) : charge, RAM, swap, disque, température, pare-feu, tentatives SSH échouées, mises à jour, redémarrage requis, services systemd en échec, conteneurs Docker, âge du dernier backup, expiration des certificats Let's Encrypt.

---

## 2. Prérequis

- Ubuntu Server 24.04 LTS installé, avec un utilisateur disposant de `sudo`.
- Accès SSH ou console à la VM.
- Accès Internet.
- Un terminal en UTF-8 (nécessaire pour les barres et symboles).

>  **Ne collez jamais un mot de passe ou une clé Wi-Fi/SSH** dans un ticket, un chat ou ce fichier. Masquez-les avant de partager une configuration.

Toutes les commandes se lancent avec l'utilisateur normal (pas en `root`), sauf mention contraire.

---

## 3. Étape 0 : préparation du système

### 3.1 Mise à jour et paquets de base

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl wget git nano htop tree unzip p7zip-full \
                    net-tools iw dnsutils ncdu jq bash-completion
```

### 3.2 Hostname, fuseau horaire, langue

```bash
sudo hostnamectl set-hostname devops          # adaptez le nom
sudo timedatectl set-timezone Europe/Paris
sudo locale-gen fr_FR.UTF-8 && sudo update-locale LANG=fr_FR.UTF-8
```

### 3.3 Invité de VM (selon l'hyperviseur)

Permet l'arrêt propre depuis l'hyperviseur et une meilleure intégration.

```bash
# Proxmox / KVM / QEMU
sudo apt install -y qemu-guest-agent && sudo systemctl enable --now qemu-guest-agent

# VirtualBox : installer les Additions invité depuis le menu Périphériques
# VMware : sudo apt install -y open-vm-tools
```

### 3.4 Sauvegarde des fichiers existants

```bash
cp ~/.bashrc ~/.bashrc.bak
[ -f ~/.bash_aliases ] && cp ~/.bash_aliases ~/.bash_aliases.bak
echo "Sauvegardes OK"
```

---

## 4. Étape 1 : installer fastfetch

`fastfetch` n'est pas dans les dépôts d'Ubuntu 24.04. Deux méthodes :

**Méthode A : PPA (mises à jour automatiques)**

```bash
sudo add-apt-repository -y ppa:zhangsongcui3371/fastfetch
sudo apt update && sudo apt install -y fastfetch
```

**Méthode B : paquet `.deb` depuis GitHub** (si le PPA est indisponible)

```bash
cd /tmp
wget https://github.com/fastfetch-cli/fastfetch/releases/latest/download/fastfetch-linux-amd64.deb
sudo apt install -y ./fastfetch-linux-amd64.deb
```

> Pour une VM ARM, remplacez `amd64` par `aarch64`.

Vérification :

```bash
fastfetch --version
```

---

## 5. Étape 2 : alias et fonctions

Tout est rangé dans `~/.bash_aliases` (chargé automatiquement par Ubuntu). **Chaque alias porte sa description après le `#`**, ce qui permet à la commande `aliases` de les lister automatiquement.

- Alias : `alias nom='commande'   # Description`
- Fonction : une ligne `#: nom | Description` juste au-dessus

```bash
cat > ~/.bash_aliases << 'EOF'
# Format alias    : alias nom='commande'   # Description (sans apostrophe)
# Format fonction : ligne  #: nom | Description  au-dessus de la fonction

# - Système -
alias sudo='sudo '                                            # Alias utilisables apres sudo
alias off='sudo shutdown -h now'                              # Éteindre la VM
alias rb='sudo reboot'                                        # Redémarrer la VM
alias maj='sudo apt update && sudo apt full-upgrade -y'       # Mettre à jour le système
alias clean='sudo apt autoremove -y && sudo apt autoclean'    # Nettoyer les paquets inutiles
alias board='sudo /etc/update-motd.d/99-devops-dashboard'     # Afficher le dashboard
alias fail='systemctl --failed'                               # Services en échec
alias logs='sudo journalctl -f'                               # Logs système en direct
alias errs='sudo journalctl -p err -b --no-pager | tail -30'  # Erreurs depuis le boot
alias cls='clear'                                             # Effacer l écran
alias reload='source ~/.bashrc'                               # Recharger le shell

# - Ressources -
alias mem='free -h'                                           # Mémoire RAM et swap
alias dfh='df -h /'                                           # Espace disque
alias duh='du -h --max-depth=1 | sort -hr | head -15'         # Plus gros dossiers
alias top10='ps aux --sort=-%mem | head -11'                  # Top 10 processus gourmands
alias temp='cat /sys/class/thermal/thermal_zone0/temp | awk "{print \$1/1000\"°C\"}"'  # Température CPU

# - Réseau -
alias ipa='ip -br a'                                          # Interfaces et adresses IP
alias myip='curl -s ifconfig.me; echo'                        # IP publique
alias ports='sudo ss -tulpn'                                  # Ports en écoute
alias pingg='ping -c 4 1.1.1.1'                               # Tester la connexion internet
alias fw='sudo ufw status verbose'                            # État du pare-feu
alias wifi='iw dev wlo1 link'                                 # Qualité du Wi-Fi (adapter wlo1)

# - Docker & Git -
alias dps='docker ps'                                         # Conteneurs actifs
alias dpa='docker ps -a'                                      # Tous les conteneurs
alias dcu='docker compose up -d'                              # Compose : démarrer
alias dcd='docker compose down'                               # Compose : arrêter
alias dcl='docker compose logs -f --tail 50'                  # Compose : logs
alias dprune='docker system prune'                            # Nettoyer Docker
alias gst='git status -sb'                                    # Git : statut
alias gl='git log --oneline --graph -15'                      # Git : historique
alias gp='git pull'                                           # Git : récupérer

# - Navigation -
alias ll='ls -lah --color=auto'                               # Liste détaillée
alias ..='cd ..'                                              # Dossier parent
alias ...='cd ../..'                                          # Remonter de 2 niveaux
alias grep='grep --color=auto'                                # Grep en couleur

# - Fonctions -
#: ff | Fastfetch + raccourcis
ff() { fastfetch; echo; aliases -f; echo; }

#: mkcd | Créer un dossier et y entrer : mkcd nom
mkcd() { mkdir -p "$1" && cd "$1"; }

#: extract | Décompresser une archive : extract fichier
extract() {
  [ -f "$1" ] || { echo "Fichier introuvable"; return 1; }
  case "$1" in
    *.tar.gz|*.tgz) tar xzf "$1" ;;  *.tar.bz2) tar xjf "$1" ;;
    *.tar.xz) tar xJf "$1" ;;        *.tar) tar xf "$1" ;;
    *.zip) unzip "$1" ;;             *.gz) gunzip "$1" ;;
    *.7z) 7z x "$1" ;;               *) echo "Format inconnu" ;;
  esac
}

#: bak | Sauvegarder un fichier avec la date : bak fichier
bak() { cp -a "$1" "$1.bak-$(date +%Y%m%d-%H%M)" && echo "✔ $1 sauvegardé"; }

#: svc | Statut d un service : svc nginx
svc() { systemctl status "$1" --no-pager; }

#: dlogs | Logs d un conteneur : dlogs nom
dlogs() { docker logs -f --tail 100 "$1"; }

#: dsh | Shell dans un conteneur : dsh nom
dsh() { docker exec -it "$1" sh -c 'bash || sh'; }

#: hg | Chercher dans l historique : hg mot
hg() { history | grep -i "$1"; }

#: ea | Éditer les alias
ea() { ${EDITOR:-nano} ~/.bash_aliases && source ~/.bash_aliases; }

#: addalias | Ajouter un alias : addalias nom commande description
addalias() {
  [ $# -eq 3 ] || { echo "Usage : addalias nom 'commande' 'description'"; return 1; }
  printf "alias %s='%s'   # %s\n" "$1" "$2" "$3" >> ~/.bash_aliases
  source ~/.bash_aliases && echo "✔ alias $1 ajouté"
}

# - Perso -
EOF
```

**Notes**

- `board` remplace l'ancien nom `dash`, qui masquerait le vrai shell `/usr/bin/dash`. `gst` remplace `gs` (Ghostscript).
- `alias sudo='sudo '` permet d'écrire `sudo ports` ou `sudo ll` en conservant les alias.
- L'alias `wifi` suppose une interface `wlo1` : vérifiez le vôtre avec `ip -br a`.
- **Pas d'apostrophe dans les descriptions**, sinon la lecture automatique échoue.

---

## 6. Étape 3 : la commande `aliases`

Elle lit `~/.bash_aliases` et propose trois modes :

| Commande | Rôle |
|---|---|
| `aliases` | Liste complète, groupée par section |
| `aliases -f` | Favoris avec description (utilisé par `ff`) |
| `aliases -n` | Noms des favoris sur une ligne (utilisé par le dashboard) |

```bash
sudo tee /usr/local/bin/aliases > /dev/null << 'EOF'
#!/bin/bash
# aliases      : liste complète par section
# aliases -f   : favoris avec description
# aliases -n   : noms des favoris sur une ligne
F="${ALIAS_FILE:-__HOME__/.bash_aliases}"
FAV="off rb maj clean board fail ports ipa fw mem ff"
[ -r "$F" ] || exit 0
awk -v mode="$1" -v fav=" $FAV " '
function trim(s){ gsub(/^[ \t]+|[ \t]+$/,"",s); return s }
function emit(n,d){
  if (mode=="-n") { if (index(fav," " n " ")) names = names (names ? " " : "") n; return }
  if (mode=="-f") {
    if (index(fav," " n " ")==0) return
    if (!hdr) { printf "  \033[1;36mRaccourcis\033[0m\n"; hdr=1 }
  } else if (sec!=last) { printf "\n  \033[1;36m%s\033[0m\n", sec; last=sec }
  printf "    \033[1;32m%-10s\033[0m %s\n", n, d
}
/^# - /{ s=$0; sub(/^# - /,"",s); sub(/ -.*$/,"",s); sec=s; next }
/^alias /{ if (match($0,/[ ]+# /)) { d=substr($0,RSTART+RLENGTH); h=substr($0,1,RSTART-1); sub(/^alias /,"",h); sub(/=.*/,"",h); emit(h,d) } next }
/^#:/{ split(substr($0,3),p,"|"); emit(trim(p[1]),trim(p[2])) }
END{ if (mode=="-n") print names }
' "$F"
EOF
# Remplace le chemin par le dossier personnel de l'utilisateur courant
sudo sed -i "s#__HOME__#$HOME#" /usr/local/bin/aliases
sudo chmod +x /usr/local/bin/aliases
```

> Le dashboard SSH tourne en `root` : le chemin du fichier d'alias est donc **écrit en dur** dans le script (`$HOME` de l'utilisateur qui installe).
>
> Pour changer vos favoris, modifiez la ligne `FAV=` dans `/usr/local/bin/aliases`.

---

## 7. Étape 4 : prompt, historique et `.bashrc`

Le bloc est délimité par des marqueurs `# >>> perso >>>` : relançable sans doublon.

```bash
sed -i '/# >>> perso >>>/,/# <<< perso <<</d' ~/.bashrc
cat >> ~/.bashrc << 'EOF'

# >>> perso >>>
# Historique : plus long, sans doublons, partagé entre sessions
HISTSIZE=50000; HISTFILESIZE=100000
HISTCONTROL=ignoreboth:erasedups
shopt -s histappend cmdhist checkwinsize autocd cdspell

# Prompt : heure, user@hôte, dossier, branche git, flèche verte/rouge selon le dernier code retour
__git() { local b; b=$(git symbolic-ref --short HEAD 2>/dev/null) && printf ' (%s)' "$b"; }
__prompt() {
  local rc=$?; local c=32; [ $rc -ne 0 ] && c=31
  PS1="\[\e[2m\]\A\[\e[0m\] \[\e[1;36m\]\u@\h\[\e[0m\] \[\e[1;34m\]\w\[\e[33m\]$(__git)\[\e[0m\] \[\e[1;${c}m\]❯\[\e[0m\] "
}
PROMPT_COMMAND=__prompt

# Fastfetch + raccourcis à l ouverture d un terminal local (le MOTD gère le SSH)
[[ $- == *i* && -z "$SSH_CONNECTION" ]] && command -v fastfetch &>/dev/null && ff
# <<< perso <<<
EOF
source ~/.bashrc
```

---

## 8. Étape 5 : configuration fastfetch

Sans logo, couleurs cyan/blanc, cohérent avec le dashboard.

```bash
mkdir -p ~/.config/fastfetch && cat > ~/.config/fastfetch/config.jsonc << 'EOF'
{
  "$schema": "https://github.com/fastfetch-cli/fastfetch/raw/dev/doc/json_schema.json",
  "logo": { "type": "none" },
  "display": { "separator": "  ", "color": { "keys": "cyan", "title": "white" } },
  "modules": [
    { "type": "title", "format": "{host-name}" },
    "break",
    { "type": "os", "key": "OS      " },
    { "type": "kernel", "key": "Kernel  " },
    { "type": "uptime", "key": "Uptime  " },
    { "type": "cpu", "key": "CPU     " },
    { "type": "memory", "key": "RAM     ", "percent": { "type": 3 } },
    { "type": "disk", "key": "Disque  ", "folders": "/", "percent": { "type": 3 } },
    { "type": "localip", "key": "IP      " },
    { "type": "packages", "key": "Paquets " }
  ]
}
EOF
ff
```

Pour garder un logo réduit : remplacez `"type": "none"` par `"type": "small"` dans la section `logo`.

---

## 9. Étape 6 : dashboard SSH (MOTD)

### 9.1 Désactiver les messages Ubuntu par défaut

Évite les doublons (aide, Landscape, actualités) :

```bash
sudo chmod -x /etc/update-motd.d/{10-help-text,50-landscape-sysinfo,50-motd-news}
```

> Si un de ces fichiers n'existe pas, `chmod` affiche une erreur sans gravité.

### 9.2 Installer le dashboard

Variable à adapter en haut du script : `BACKUP_DIR` (dossier de sauvegardes, laissez vide pour masquer la ligne).

```bash
cat << 'EOF' | sudo tee /etc/update-motd.d/99-devops-dashboard > /dev/null && sudo chmod +x /etc/update-motd.d/99-devops-dashboard
#!/bin/bash
export LC_ALL=C.UTF-8

# - Config ------------------------
BACKUP_DIR=""            # ex: /var/backups/restic (vide = masqué)
W=72

R=$'\033[0m'; B=$'\033[1m'; DIM=$'\033[2m'
CY=$'\033[36m'; GR=$'\033[32m'; YE=$'\033[33m'; RD=$'\033[31m'; WH=$'\033[97m'

line()    { printf '%s' "$DIM"; for ((i=0;i<W;i++)); do printf '─'; done; printf '%s\n' "$R"; }
section() { printf '\n %s▌%s %s%s%s\n' "$CY" "$R" "$B$WH" "$1" "$R"; }
row()     { printf '   %s%-16s%s %b\n' "$DIM" "$1" "$R" "$2"; }
color()   { if (( $1 >= 90 )); then printf '%s' "$RD"; elif (( $1 >= 70 )); then printf '%s' "$YE"; else printf '%s' "$GR"; fi; }
bar() {
    local pct=$1 w=20 f i; (( pct > 100 )) && pct=100; f=$(( pct * w / 100 ))
    color "$pct"; for ((i=0;i<f;i++)); do printf '━'; done
    printf '%s' "$DIM"; for ((i=f;i<w;i++)); do printf '─'; done
    printf '%s %3d%%%s' "$R" "$pct" "$R"
}
ok()   { printf '%s●%s %s' "$GR" "$R" "$1"; }
warn() { printf '%s●%s %s' "$YE" "$R" "$1"; }
bad()  { printf '%s●%s %s' "$RD" "$R" "$1"; }
human_age() { local s=$1; if (( s < 3600 )); then echo "$((s/60)) min"; elif (( s < 86400 )); then echo "$((s/3600)) h"; else echo "$((s/86400)) j"; fi; }

# - Collecte -----------------------
. /etc/os-release
HOST=$(hostname); KERNEL=$(uname -r); NOW=$(date '+%d/%m/%Y %H:%M')
UPTIME=$(uptime -p | sed 's/^up //')
CORES=$(nproc)
CPU=$(lscpu | awk -F': +' '/Model name/ {print $2; exit}' | sed 's/([RTM]*)//g; s/  */ /g' | cut -c1-38)
read -r L1 L5 L15 _ < /proc/loadavg
LOAD_PCT=$(awk -v l="$L1" -v c="$CORES" 'BEGIN{printf "%d", l*100/c}')

MEM_USED=$(free -m | awk '/Mem:/ {print $3}'); MEM_TOTAL=$(free -m | awk '/Mem:/ {print $2}')
MEM_PCT=$(( MEM_USED * 100 / MEM_TOTAL ))
SWAP_PCT=$(free -m | awk '/Swap:/ {if ($2>0) print int($3*100/$2); else print 0}')
DISK_PCT=$(df -P / | awk 'NR==2 {gsub("%","",$5); print $5}')
DISK_INFO=$(df -h / | awk 'NR==2 {print $3" / "$2}')
TEMP=""
[ -r /sys/class/thermal/thermal_zone0/temp ] && TEMP=$(( $(cat /sys/class/thermal/thermal_zone0/temp) / 1000 ))

IP_LAN=$(hostname -I | awk '{print $1}')
IP_TS=$(ip -4 addr show tailscale0 2>/dev/null | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
GW=$(ip route | awk '/default/ {print $3; exit}')
PORTS=$(ss -tlnH 2>/dev/null | awk '$4 !~ /^(127\.|\[::1\])/ {n=split($4,a,":"); print a[n]}' | sort -nu | paste -sd' ')

UPD=$(timeout 3 /usr/lib/update-notifier/apt-check 2>&1 | tr ';' ' ')
UPD_ALL=${UPD%% *}; UPD_SEC=${UPD##* }
FAILED_LIST=$(systemctl --failed --no-legend --plain 2>/dev/null | awk '{print $1}' | paste -sd' ')
FAILED=$(systemctl --failed --no-legend --plain 2>/dev/null | wc -l)

# - Affichage ----------------------─
echo; line
printf ' %s%s%s  %s%s%s\n' "$B$WH" "${HOST^^}" "$R" "$DIM" "$NOW" "$R"
printf ' %s%s · kernel %s%s\n' "$DIM" "$PRETTY_NAME" "$KERNEL" "$R"
line

section "SYSTÈME"
row "Uptime"   "$UPTIME"
row "CPU"      "${CPU} ${DIM}(${CORES}c)${R}"
row "Charge"   "$(bar "$LOAD_PCT")  ${DIM}${L1} ${L5} ${L15}${R}"
row "Mémoire"  "$(bar "$MEM_PCT")  ${DIM}${MEM_USED} / ${MEM_TOTAL} Mo${R}"
row "Swap"     "$(bar "$SWAP_PCT")"
row "Disque /" "$(bar "$DISK_PCT")  ${DIM}${DISK_INFO}${R}"
if [ -n "$TEMP" ]; then
    if (( TEMP >= 80 )); then row "Température" "$(bad "${TEMP}°C")"
    elif (( TEMP >= 65 )); then row "Température" "$(warn "${TEMP}°C")"
    else row "Température" "$(ok "${TEMP}°C")"; fi
fi

section "RÉSEAU"
row "IP LAN"    "${IP_LAN} ${DIM}→ gw ${GW}${R}"
if [ -n "$IP_TS" ]; then row "Tailscale" "$(ok "$IP_TS")"
elif command -v tailscale &>/dev/null; then row "Tailscale" "$(bad "non connecté")"
else row "Tailscale" "${DIM}non installé${R}"; fi
row "Ports ouverts" "${YE}${PORTS:-aucun}${R}"

section "SÉCURITÉ"
if command -v ufw &>/dev/null; then
    if ufw status 2>/dev/null | grep -q "Status: active"; then
        row "Pare-feu" "$(ok "UFW actif") ${DIM}($(ufw status | grep -c ALLOW) règles allow)${R}"
    else row "Pare-feu" "$(bad "UFW inactif") ${DIM}→ ufw enable${R}"; fi
else row "Pare-feu" "$(bad "UFW absent")"; fi
if systemctl is-active --quiet fail2ban 2>/dev/null; then
    BANNED=$(fail2ban-client status sshd 2>/dev/null | awk -F'\t' '/Currently banned/ {print $NF}')
    row "Fail2ban" "$(ok "actif") ${DIM}· ${BANNED:-0} IP bannie(s) sur sshd${R}"
else row "Fail2ban" "$(warn "non installé")"; fi
FAILS=$(timeout 3 journalctl -u ssh --since "24 hours ago" -q --no-pager 2>/dev/null | grep -c "Failed password")
if (( FAILS > 50 )); then row "SSH échecs 24h" "$(bad "$FAILS")"
elif (( FAILS > 10 )); then row "SSH échecs 24h" "$(warn "$FAILS")"
else row "SSH échecs 24h" "$(ok "$FAILS")"; fi
SESS=$(who | awk '{ip=$NF; gsub(/[()]/,"",ip); if (ip==$1 || ip ~ /^tty|^:/) ip="local"; print $1"@"ip}' | paste -sd' ')
row "Sessions" "$(who | wc -l) ${DIM}· ${SESS}${R}"

if command -v docker &>/dev/null && timeout 3 docker info &>/dev/null; then
    section "CONTENEURS"
    D_RUN=$(docker ps -q | wc -l); D_ALL=$(docker ps -aq | wc -l)
    D_BAD=$(docker ps -q --filter health=unhealthy | wc -l)
    D_EXIT=$(docker ps -aq --filter status=exited | wc -l)
    D_RESTART=$(docker ps -q --filter status=restarting | wc -l)
    row "Docker" "$(ok "${D_RUN} actifs") ${DIM}/ ${D_ALL} total · ${D_EXIT} arrêtés${R}"
    (( D_BAD > 0 ))     && row "Unhealthy"  "$(bad "${D_BAD} conteneur(s)")"
    (( D_RESTART > 0 )) && row "Restarting" "$(warn "${D_RESTART} conteneur(s)")"
    NAMES=$(docker ps --format '{{.Names}}' | head -8 | paste -sd' ')
    [ -n "$NAMES" ] && row "En cours" "${DIM}${NAMES}${R}"
    IMG=$(docker images -f dangling=true -q | wc -l)
    (( IMG > 0 )) && row "Images orphelines" "$(warn "$IMG") ${DIM}→ docker image prune${R}"
fi

section "MAINTENANCE"
if [ "${UPD_ALL:-0}" -gt 0 ] 2>/dev/null; then
    if [ "${UPD_SEC:-0}" -gt 0 ]; then row "Mises à jour" "$(bad "${UPD_ALL} en attente") ${DIM}(dont ${UPD_SEC} sécurité)${R}"
    else row "Mises à jour" "$(warn "${UPD_ALL} en attente")"; fi
else row "Mises à jour" "$(ok "à jour")"; fi
if [ -f /var/run/reboot-required ]; then row "Redémarrage" "$(bad "requis")"; else row "Redémarrage" "$(ok "non requis")"; fi
if (( FAILED > 0 )); then row "Services" "$(bad "${FAILED} en échec") ${DIM}${FAILED_LIST}${R}"; else row "Services" "$(ok "tous OK")"; fi

if [ -n "$BACKUP_DIR" ] && [ -d "$BACKUP_DIR" ]; then
    LAST=$(find "$BACKUP_DIR" -type f -printf '%T@\n' 2>/dev/null | sort -n | tail -1 | cut -d. -f1)
    if [ -n "$LAST" ]; then
        AGE=$(( $(date +%s) - LAST ))
        if (( AGE > 172800 )); then row "Dernier backup" "$(bad "il y a $(human_age $AGE)")"
        elif (( AGE > 90000 )); then row "Dernier backup" "$(warn "il y a $(human_age $AGE)")"
        else row "Dernier backup" "$(ok "il y a $(human_age $AGE)")"; fi
    fi
fi

CERTS=(/etc/letsencrypt/live/*/cert.pem)
if [ -e "${CERTS[0]}" ]; then
    section "CERTIFICATS"
    for c in "${CERTS[@]}"; do
        d=$(basename "$(dirname "$c")")
        end=$(openssl x509 -enddate -noout -in "$c" 2>/dev/null | cut -d= -f2)
        days=$(( ( $(date -d "$end" +%s) - $(date +%s) ) / 86400 ))
        if (( days < 7 )); then row "$d" "$(bad "${days} j restants")"
        elif (( days < 21 )); then row "$d" "$(warn "${days} j restants")"
        else row "$d" "$(ok "${days} j restants")"; fi
    done
fi

section "OUTILS"
TOOLS=""
for t in docker git kubectl helm ansible terraform tailscale; do command -v "$t" &>/dev/null && TOOLS+="$t "; done
row "Détectés" "${GR}${TOOLS:-aucun}${R}"
[ -x /usr/local/bin/aliases ] && row "Raccourcis" "${CY}$(/usr/local/bin/aliases -n)${R} ${DIM}→ aliases${R}"
echo; line; echo
EOF
```

### 9.3 Tester

```bash
sudo /etc/update-motd.d/99-devops-dashboard     # ou simplement : board
```

Puis ouvrez une **nouvelle session SSH** pour voir le rendu réel à la connexion.

### 9.4 Option : masquer la ligne « Last login » de sshd

```bash
sudo sed -i 's/^#\?PrintLastLog.*/PrintLastLog no/' /etc/ssh/sshd_config
sudo sshd -t && sudo systemctl reload ssh
```

> Testez avec une deuxième connexion avant de fermer la session en cours.

### 9.5 Seuils d'alerte

| Indicateur | Jaune | Rouge |
|---|---|---|
| Barres (charge, RAM, swap, disque) | ≥ 70 % | ≥ 90 % |
| Température | ≥ 65 °C | ≥ 80 °C |
| Échecs SSH sur 24 h | > 10 | > 50 |
| Certificat Let's Encrypt | < 21 jours | < 7 jours |
| Dernier backup | > 25 h | > 48 h |

Modifiables directement dans le script.

---

## 10. Étape 7 : sécurité de base

Le dashboard affiche en rouge un pare-feu inactif. À traiter dès l'installation.

### 10.1 Pare-feu UFW

**Autorisez SSH avant d'activer**, sinon vous perdez l'accès :

```bash
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status verbose
```

Ouvrir d'autres services selon les besoins : `sudo ufw allow 80,443/tcp`.

### 10.2 Fail2ban (recommandé si SSH est exposé)

```bash
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

### 10.3 Mises à jour de sécurité automatiques

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

### 10.4 Authentification SSH par clé (recommandé)

Depuis **votre poste** :

```bash
ssh-keygen -t ed25519 -C "mon-poste"
ssh-copy-id utilisateur@IP_DE_LA_VM
```

Quand la connexion par clé fonctionne, désactivez le mot de passe :

```bash
sudo tee /etc/ssh/sshd_config.d/99-durcissement.conf > /dev/null << 'EOF'
PasswordAuthentication no
PermitRootLogin no
EOF
sudo sshd -t && sudo systemctl reload ssh
```

>  Gardez une session ouverte et testez dans un second terminal avant de la fermer.

---

## 11. Utilisation au quotidien

| Commande | Effet |
|---|---|
| `aliases` | Liste complète des alias et fonctions, par section |
| `ff` | fastfetch + raccourcis favoris |
| `board` | Réaffiche le dashboard |
| `off` / `rb` | Éteindre / redémarrer proprement |
| `maj` | Mettre à jour tout le système |
| `ports` | Ports en écoute |
| `fail` | Services systemd en échec |
| `ea` | Éditer le fichier d'alias |
| `reload` | Recharger le shell |
| `addalias nom 'cmd' 'desc'` | Ajouter un alias visible partout |

Exemple :

```bash
addalias nginxr 'sudo systemctl restart nginx' 'Redémarrer Nginx'
```

Il apparaît immédiatement dans `aliases`. Pour le faire figurer dans les favoris (`ff` et dashboard), ajoutez son nom à la ligne `FAV=` de `/usr/local/bin/aliases`.

---

## 12. Dépannage

| Symptôme | Cause probable | Solution |
|---|---|---|
| Colonnes décalées dans le dashboard | Locale non UTF-8 | Vérifier `export LC_ALL=C.UTF-8` en haut du script |
| Rien ne s'affiche à la connexion SSH | Script non exécutable | `sudo chmod +x /etc/update-motd.d/99-devops-dashboard` |
| Doublons à la connexion | Anciens MOTD actifs | Étape 9.1 |
| `aliases` n'affiche rien | Mauvais chemin dans le script | `grep 'F=' /usr/local/bin/aliases` et corriger |
| `ff: command not found` | Alias non chargés | `source ~/.bashrc` |
| `fastfetch: command not found` | Non installé | Étape 4 |
| Alias cassé après ajout | Apostrophe dans la description | Éditer avec `ea` |
| Service `systemd-networkd-wait-online` en échec | Interface non branchée (ex. Ethernet absent) | Voir ci-dessous |
| Température absente | Pas de capteur exposé par la VM | Normal en VM, la ligne est masquée |

### Service `systemd-networkd-wait-online` en échec

Fréquent quand une interface déclarée n'est pas connectée. Dans votre fichier Netplan (`/etc/netplan/*.yaml`), marquez l'interface comme optionnelle :

```yaml
network:
  version: 2
  ethernets:
    eno1:
      dhcp4: true
      optional: true
```

Puis :

```bash
sudo chmod 600 /etc/netplan/*.yaml
sudo netplan try          # annulation auto après 120 s si la connexion est perdue
sudo systemctl reset-failed systemd-networkd-wait-online.service
```

---

## 13. Annuler la personnalisation

```bash
# Dashboard et commande aliases
sudo rm -f /etc/update-motd.d/99-devops-dashboard /usr/local/bin/aliases

# Restaurer les MOTD Ubuntu par défaut
sudo chmod +x /etc/update-motd.d/{10-help-text,50-landscape-sysinfo,50-motd-news}

# Retirer le bloc du .bashrc
sed -i '/# >>> perso >>>/,/# <<< perso <<</d' ~/.bashrc

# Alias et config fastfetch
rm -f ~/.bash_aliases
rm -rf ~/.config/fastfetch

# (ou restaurer les sauvegardes de l'étape 3.4)
# cp ~/.bashrc.bak ~/.bashrc
```

---

## Récapitulatif des fichiers créés

| Fichier | Rôle |
|---|---|
| `~/.bash_aliases` | Alias et fonctions documentés |
| `~/.bashrc` (bloc `perso`) | Historique, prompt, fastfetch en local |
| `~/.config/fastfetch/config.jsonc` | Apparence de fastfetch |
| `/usr/local/bin/aliases` | Liste les alias à partir de leurs commentaires |
| `/etc/update-motd.d/99-devops-dashboard` | Dashboard affiché à la connexion SSH |

---

*Testé sur Ubuntu Server 24.04 LTS.*
