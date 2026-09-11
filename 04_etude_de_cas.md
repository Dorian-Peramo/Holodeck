# Étude de cas — Projet Holodeck

> Document destiné à préparer la présentation orale : il explique le
> **fond** (pourquoi ces choix techniques) et la **forme** (comment le
> travail a été mené) du projet, indépendamment du détail des commandes
> déjà couvert dans le journal technique.

---

## 1. Présentation du cas

### Le scénario du sujet

Le sujet, habillé sous la forme d'un scénario Star Trek, demande de
construire pour la Fédération des Planètes Unies une infrastructure web
destinée aux ingénieurs du "Holodeck" à bord de l'USS Enterprise-D.
Derrière l'habillage fictif, c'est un cas très classique d'infrastructure
d'entreprise : un serveur qui héberge plusieurs services web, avec des
contraintes de sécurité, de compatibilité applicative et de gestion
réseau — le genre de situation qu'on retrouve dans n'importe quelle PME
qui héberge ses propres services.

### La problématique de fond

Comment construire, sur une seule machine, une plateforme web qui :
- héberge plusieurs applications avec des besoins techniques différents
  (deux versions de PHP incompatibles entre elles) ;
- reste administrable sans compte à privilèges élargis ;
- protège les échanges (web et transfert de fichiers) par chiffrement ;
- authentifie ses utilisateurs de façon centralisée ;
- n'expose que le strict nécessaire vers l'extérieur.

C'est une problématique d'**équilibre** : chaque contrainte de sécurité
ajoutée complique l'administration, et chaque service ajouté élargit la
surface d'attaque. Le projet consiste à démontrer qu'on peut concilier
les deux.

---

## 2. Le fond — analyse des besoins et choix techniques

### 2.1 Traduire un cahier des charges en architecture

Le sujet donne des contraintes fonctionnelles (ce que le système doit
faire) et des contraintes de sécurité (comment il doit le faire). Le
premier travail a été de les traduire en une architecture concrète :
deux VM, deux réseaux (WAN/LAN), un domaine interne dédié
(`starfleet.lan`), et une répartition claire des services sur le
serveur.

Ce découpage réseau (LAN isolé pour les services internes, WAN pour les
mises à jour) n'est pas explicitement demandé mot pour mot dans le
sujet, mais il découle directement de la contrainte "pare-feu strict" :
sans séparer les deux réseaux, on ne peut pas distinguer ce qui doit être
exposé de ce qui doit rester interne.

### 2.2 Les arbitrages techniques et leur justification

Plusieurs choix ont été faits en pesant le coût (temps, complexité,
maintenabilité) face au bénéfice (conformité au sujet, robustesse,
valeur pédagogique) :

| Choix | Alternative écartée | Raison |
|---|---|---|
| PHP via dépôt tiers (sury.org) | Compilation depuis les sources | Le sujet exige une version récente, pas une compilation maison — le dépôt tiers répond à l'exigence sans la lourdeur (dépendances, absence d'intégration systemd) de la compilation |
| Autorité de certification maison | Certificat auto-signé simple | Une CA, même petite, reproduit le vrai fonctionnement d'une PKI d'entreprise et est plus démonstrative à l'oral qu'un certificat isolé |
| nftables | iptables | nftables est le remplaçant moderne recommandé sur Debian actuel, avec une syntaxe plus lisible pour une politique par défaut restrictive |
| Cockpit | Webmin | Les deux étaient proposés par le sujet ; Cockpit offre une interface plus moderne et un support natif du reverse proxy, cohérent avec le reste de l'architecture en HTTPS |
| Administration en root direct | Création d'un compte avec droits élargis limités | Le sujet interdit explicitement tout compte à privilèges étendus — la seule option cohérente est l'administration directe en root, documentée comme telle |

### 2.3 La sécurité comme fil conducteur, pas comme étape finale

Un point de fond important : la sécurité n'a pas été traitée comme une
case à cocher en fin de projet, mais infusée dans chaque brique au fur
et à mesure :
- le chiffrement (HTTPS, FTP en TLS) a été mis en place dès que chaque
  service devenait fonctionnel, pas après coup ;
- le chroot FTP a été **vérifié activement** (tentative de sortie du
  cloisonnement), pas seulement configuré puis supposé fonctionnel ;
- le pare-feu a été conçu en listant d'abord précisément tous les flux
  réellement nécessaires, plutôt qu'en ouvrant largement puis en
  restreignant après coup.

Cette approche — sécuriser au fur et à mesure plutôt qu'en bloc à la fin
— est directement liée aux compétences visées par le sujet
("participer à l'élaboration et à la mise en œuvre de la politique de
sécurité").

---

## 3. La forme — méthodologie et organisation du travail

### 3.1 Une progression brique par brique

Le projet a été mené de façon **incrémentale** : chaque service a été
installé, testé et validé isolément avant de passer au suivant, plutôt
que de tout configurer d'un bloc puis de déboguer l'ensemble à la fin.
Ordre suivi : réseau (DHCP/DNS) → serveur applicatif (PHP) → serveur web
(Nginx) → chiffrement (certificat) → base de données → annuaire → FTP →
administration → pare-feu.

Cet ordre n'est pas arbitraire : chaque brique dépend techniquement de
la précédente (les vhosts ont besoin de PHP-FPM déjà actif ; le FTP
réutilise le certificat déjà généré ; le pare-feu ne peut être finalisé
qu'une fois tous les ports réellement utilisés connus).

### 3.2 Une méthode de résolution de problèmes reproductible

Le projet a rencontré plusieurs blocages techniques réels (détaillés
dans le journal technique). La méthode appliquée à chacun a été
constante :

1. **Isoler le symptôme** (quelle commande échoue, avec quel message
   exact)
2. **Consulter les logs du service concerné** avant toute hypothèse
   (Nginx, PHP-FPM, Cockpit, vsftpd, systemd)
3. **Tester une hypothèse à la fois**, jamais plusieurs corrections en
   même temps
4. **Valider explicitement** la correction (nouveau test, pas une
   simple absence d'erreur)

Cette méthode a permis de distinguer, par exemple, un vrai problème de
configuration (permissions de socket PHP-FPM) d'un faux problème lié à
une erreur de syntaxe d'une commande de test (le cas du mot de passe FTP
vide) — la différence entre les deux n'était pas évidente au premier
abord.

### 3.3 Une documentation pensée comme un livrable à part entière

Le sujet demande explicitement une procédure d'export et une notice
utilisateur. Au-delà du minimum demandé, la documentation produite a
été structurée en plusieurs niveaux, chacun avec un public différent :

- un **README** pour une vue d'ensemble rapide (architecture, services,
  sécurité)
- une **notice d'installation/utilisation** pour un utilisateur final
  qui n'a pas suivi la construction du projet
- une **procédure d'export** technique et reproductible
- un **journal technique complet**, qui trace non seulement les
  commandes mais le raisonnement et les erreurs rencontrées — pensé pour
  rester exploitable dans la durée, au-delà de la seule évaluation
  immédiate

Cette stratification répond à une logique de forme : documenter, ce
n'est pas décrire ce qu'on a tapé, c'est permettre à quelqu'un d'autre
(ou à soi-même, plus tard) de comprendre et de reproduire le travail.

---

## 4. Résultats et validation

Chaque brique a été validée par un test fonctionnel réel, pas par une
simple vérification que le service "tourne" :

| Brique | Preuve de fonctionnement |
|---|---|
| DHCP/DNS | Résolution croisée serveur ↔ client, `dig` en interrogation directe |
| PHP 7/8 | Version affichée séparément par chaque vhost |
| HTTPS | Certificat accepté sans avertissement une fois la CA importée |
| LDAP | Authentification réussie depuis un vrai formulaire web (`ldap_bind`) |
| FTP | Connexion chiffrée réussie et tentative de sortie du chroot bloquée |
| Cockpit | Tableau de bord accessible après connexion root |
| Pare-feu | Tous les services encore accessibles après application des règles |

---

## 5. Difficultés et posture professionnelle

Le projet n'a pas suivi une ligne droite. Plusieurs blocages ont
nécessité des heures de diagnostic (en particulier la réécriture
répétée de `/etc/resolv.conf` par le client DHCP, et le refus de
connexion WebSocket de Cockpit). Ce qui est présenté ici comme une
suite logique d'étapes est en réalité le résultat d'itérations, de
fausses pistes écartées, et de corrections successives.

C'est un point à assumer à l'oral plutôt qu'à masquer : la valeur
démontrée par ces épisodes n'est pas d'avoir tout su résoudre du premier
coup, mais d'avoir su **diagnostiquer méthodiquement** des problèmes
dont la cause n'était pas évidente au premier regard — ce qui est
précisément la compétence recherchée par un métier d'administration
système et de sécurité.

---

## 6. Bilan critique et pistes d'amélioration

Points qui pourraient être présentés comme axes d'évolution si la
question est posée à l'oral :

- **Haute disponibilité** : l'infrastructure repose sur une seule VM
  serveur ; un vrai environnement de production répartirait les services
  critiques (DNS secondaire, réplication LDAP, cluster de base de
  données).
- **Supervision** : aucun outil de monitoring/alerting n'a été mis en
  place au-delà des logs bruts.
- **Gestion des secrets** : les mots de passe sont actuellement
  transmis "à la main" ; un coffre-fort de secrets (Vault, ou a minima
  des variables d'environnement chiffrées) serait plus robuste à plus
  grande échelle.
- **Automatisation** : chaque brique a été configurée manuellement ; un
  outil d'infrastructure-as-code (Ansible, par exemple) rendrait le
  déploiement reproductible en une commande plutôt qu'en suivant un
  journal de bord.

Ces limites ne remettent pas en cause la validité du projet dans le
cadre donné (un environnement pédagogique isolé), mais permettent de
montrer une prise de recul sur ce qui changerait dans un contexte de
production réel.
