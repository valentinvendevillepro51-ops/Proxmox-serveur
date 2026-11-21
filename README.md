# 🖥️ Homelab Proxmox — VLAN, Tailscale & auto-hébergement


Serveur **Proxmox VE** personnel servant de terrain d'apprentissage réseau et de plateforme d'hébergement (AdGuard Home, sites web, services divers), accessible à distance de façon sécurisée via **Tailscale**.

---

## 🎯 Objectifs

1. **Apprendre le réseau** : VLAN, routage inter-VLAN, pare-feu, DNS, DHCP, VPN, reverse proxy.
2. **Héberger mes services** : sites web, applications self-hosted, bases de données, etc.
3. **Filtrer les pubs et traqueurs** sur tout le réseau avec **AdGuard Home**.
4. **Accéder à tout depuis n'importe où** sans ouvrir de port sur la box, grâce à **Tailscale**.
5. **Expérimenter sans risque** : snapshots, sauvegardes, VM/LXC jetables.

---

## 🧱 Architecture

```
                    Internet
                        │
                  [ Box / Routeur ]
                        │
              [ Pare-feu / Routeur VLAN ]   (OPNsense / pfSense en VM, ou routeur physique)
                        │  trunk 802.1Q
                [ Switch managé VLAN ]
                        │  trunk 802.1Q
              ┌─────────┴──────────┐
              │   Proxmox VE       │
              │   vmbr0 (VLAN-aware)│
              └─┬────┬────┬────┬───┘
            VLAN10 VLAN20 VLAN30 VLAN40
              │     │      │      │
            Mgmt  Serveurs  Lab   IoT/invités
```

### Plan d'adressage VLAN (exemple)

| VLAN | Nom        | Réseau          | Usage                                          |
|------|------------|-----------------|------------------------------------------------|
| 10   | MGMT       | 10.10.10.0/24   | Interface Proxmox, switch, administration      |
| 20   | SERVERS    | 10.10.20.0/24   | AdGuard Home, sites web, services en production|
| 30   | LAB        | 10.10.30.0/24   | Tests, expérimentations, VM jetables           |
| 40   | IOT/GUEST  | 10.10.40.0/24   | Appareils connectés, invités (isolés)          |
| 1/50 | LAN        | 10.10.50.0/24   | PC, téléphones (clients de confiance)          |

> Règle de base du pare-feu : **tout est bloqué entre VLAN par défaut**, on n'autorise que le nécessaire (ex. LAN → SERVERS en 80/443, tous → AdGuard en 53, MGMT accessible uniquement depuis LAN/Tailscale).

---

## 🧰 Matériel et prérequis

- Un PC/mini-PC pour Proxmox (idéalement 2 cartes réseau, 16 Go de RAM, SSD).
- Un **switch managé** compatible VLAN 802.1Q (le port vers Proxmox en *trunk*).
- Un routeur/pare-feu capable de gérer les VLAN (OPNsense, pfSense, MikroTik, UniFi…).
- Un compte **Tailscale** (gratuit pour usage perso).
- Clé USB pour l'ISO de **Proxmox VE** : <https://www.proxmox.com/en/downloads>

---

## 🚀 Installation

### 1. Installer Proxmox VE
Installer depuis l'ISO, définir une IP statique de management (ex. `10.10.10.2/24`, passerelle `10.10.10.1`) et un mot de passe root solide.

### 2. Configurer le bridge VLAN-aware

Éditer `/etc/network/interfaces` :

```ini
auto lo
iface lo inet loopback

iface eno1 inet manual

auto vmbr0
iface vmbr0 inet manual
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094

# Interface de management de l'hôte dans le VLAN 10
auto vmbr0.10
iface vmbr0.10 inet static
    address 10.10.10.2/24
    gateway 10.10.10.1
```

Appliquer : `ifreload -a` (ou redémarrer).

Pour attacher une VM/LXC à un VLAN : dans les paramètres de la carte réseau, bridge `vmbr0` + **VLAN Tag** = `20` (par exemple).

### 3. Configurer le switch
- Port vers Proxmox : **trunk** (VLAN 10, 20, 30, 40 tagués).
- Port vers le routeur : **trunk**.
- Ports clients : **access** dans le VLAN voulu.

### 4. Pare-feu / routage inter-VLAN
Créer une interface par VLAN sur le routeur, activer un serveur DHCP par VLAN (sauf si IP statiques), puis écrire les règles de filtrage (voir plan ci-dessus).

---

## 🔐 Tailscale — accès distant sécurisé

Tailscale crée un VPN maillé (basé sur WireGuard) : pas de redirection de port, pas d'IP publique nécessaire.

### Installation sur l'hôte Proxmox (ou dans un LXC dédié)

```bash
curl -fsSL https://tailscale.com/install.sh | sh

# Autoriser le routage IP (nécessaire pour le subnet router)
echo 'net.ipv4.ip_forward = 1' | tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | tee -a /etc/sysctl.d/99-tailscale.conf
sysctl -p /etc/sysctl.d/99-tailscale.conf

# Connexion + annonce des sous-réseaux
tailscale up --advertise-routes=10.10.10.0/24,10.10.20.0/24 --accept-dns=false
```

Puis, dans la [console d'administration Tailscale](https://login.tailscale.com/admin/machines) :
1. **Approuver les routes** annoncées par la machine.
2. (Optionnel) Désactiver l'expiration de clé pour cette machine.
3. (Optionnel) Utiliser **ACL/tags** pour limiter qui accède à quoi.

> 💡 **Variante recommandée** : installer Tailscale dans un **LXC dédié** (VLAN 10) plutôt que sur l'hôte, pour garder l'hôte Proxmox propre. Il faut alors activer l'accès à `/dev/net/tun` pour le conteneur.

### DNS : AdGuard partout via Tailscale
Dans *Tailscale → DNS*, ajouter l'IP d'AdGuard Home (`10.10.20.x`) comme serveur DNS global, ou en *split DNS*. Les appareils nomades bénéficient alors du filtrage hors de la maison.

### Accès pratique
- Interface Proxmox : `https://10.10.10.2:8006` (via la route Tailscale).
- SSH : `ssh root@10.10.10.2` depuis n'importe quel appareil connecté au tailnet.

---

## 📦 Services hébergés

| Service              | Type    | VLAN | Rôle                                              |
|----------------------|---------|------|---------------------------------------------------|
| **AdGuard Home**     | LXC     | 20   | DNS + blocage pubs/traqueurs pour tout le réseau  |
| **Tailscale**        | LXC/hôte| 10   | Accès distant, subnet router                      |
| **Reverse proxy** (Nginx Proxy Manager / Caddy / Traefik) | LXC | 20 | Point d'entrée HTTP(S), certificats TLS |
| **Sites web / apps** | LXC/VM  | 20   | Sites perso, projets, applications self-hosted    |
| **Pare-feu** (OPNsense) | VM   | —    | Routage et filtrage inter-VLAN (optionnel)        |
| **Lab**              | VM/LXC  | 30   | Tests réseau, Linux, Windows Server, Docker, etc. |
| **Sauvegardes** (Proxmox Backup Server) | VM/LXC | 10 | Sauvegardes des VM/LXC (idéalement sur un autre disque/machine) |

### AdGuard Home — configuration rapide
1. Créer un LXC Debian/Ubuntu dans le VLAN 20 avec une **IP statique** (ex. `10.10.20.53`).
2. Installer AdGuard Home : <https://github.com/AdguardTeam/AdGuardHome#getting-started>
3. Interface de configuration initiale sur le port `3000`, puis `80`.
4. Distribuer cette IP comme **DNS** via le DHCP de chaque VLAN.
5. Autoriser le port 53 (TCP/UDP) vers AdGuard depuis tous les VLAN dans le pare-feu.

### Exposer un site web
- **En privé** : accessible uniquement via Tailscale (le plus sûr).
- **Au public** : via un reverse proxy + redirection 80/443, ou via un tunnel (Cloudflare Tunnel, Tailscale Funnel) pour éviter d'ouvrir des ports. Toujours placer ces services dans le VLAN SERVERS isolé.

---

## 🔒 Bonnes pratiques de sécurité

- Désactiver l'accès SSH par mot de passe, utiliser des **clés SSH**.
- Activer le **2FA** sur Proxmox et Tailscale.
- Ne jamais exposer l'interface Proxmox (8006) sur Internet.
- Isoler les VLAN IoT/invités du reste du réseau.
- Activer le **pare-feu Proxmox** (datacenter + VM) en complément.
- Mettre à jour régulièrement (`apt update && apt full-upgrade`).
- Faire des **sauvegardes régulières** et tester la restauration.
- Utiliser des **snapshots** avant chaque expérimentation risquée.

---

## 📚 Ce que j'apprends avec ce projet

- Fonctionnement des **VLAN (802.1Q)**, trunks, ports access/tagged
- **Routage inter-VLAN** et règles de pare-feu
- **DNS / DHCP**, filtrage avec AdGuard Home
- **VPN maillé** et subnet routing avec Tailscale
- **Virtualisation** (KVM) et **conteneurs** (LXC)
- **Reverse proxy**, certificats TLS (Let's Encrypt)
- Supervision, sauvegarde et restauration

---

## 🗺️ Feuille de route

- [ ] Installer Proxmox VE et configurer le bridge VLAN-aware
- [ ] Configurer le switch et le pare-feu (VLAN 10/20/30/40)
- [ ] Installer Tailscale + subnet router
- [ ] Déployer AdGuard Home
- [ ] Déployer le reverse proxy
- [ ] Héberger un premier site web
- [ ] Mettre en place les sauvegardes (PBS)
- [ ] Ajouter la supervision (Uptime Kuma, Grafana, Prometheus…)
- [ ] Documenter chaque service

---

## 🔗 Ressources utiles

- Documentation Proxmox : <https://pve.proxmox.com/wiki/Main_Page>
- Documentation Tailscale : <https://tailscale.com/kb>
- AdGuard Home : <https://github.com/AdguardTeam/AdGuardHome>
- Proxmox Helper Scripts (communauté) : <https://community-scripts.github.io/ProxmoxVE/>

---

## 📝 Licence

Projet personnel d'apprentissage — libre à adapter pour votre propre homelab.
