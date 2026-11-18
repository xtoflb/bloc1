# Activité 2 — Déployer un portfolio sur une VM Docker mutualisée puis sur Internet

**BTS SIO2 — Bloc 1 : Support et mise à disposition de services informatiques**  
**Durée : 1h30**  
**Modalité : activité Docker sur Windows ou Linux**

## Compétences visées du bloc 1

Cette activité mobilise principalement les compétences suivantes du bloc 1 du BTS SIO : **mettre à disposition des utilisateurs un service informatique**, **développer la présence en ligne de l’organisation**, **répondre aux incidents et aux demandes d’assistance et d’évolution**, **travailler en mode projet**, **gérer le patrimoine informatique** et **organiser son développement professionnel**.

## Finalité

Cette deuxième activité prolonge directement la première : le portfolio conteneurisé n’est plus seulement exécuté sur votre machine. Il est d’abord déployé sur une **VM Docker mutualisée** préparée par l’enseignant, puis publié sur **Render** afin d’obtenir une URL publique accessible sur Internet.

La difficulté augmente progressivement : vous devez désormais travailler avec un environnement partagé, respecter une convention de déploiement, gérer un dépôt Git et comprendre la différence entre validation interne et publication externe.

Cette activité se réalise donc en deux tâches :

- une **préproduction** sur VM Docker mutualisée ;
- une **publication Internet** sur une plateforme cloud (PaaS).

---

## Objectifs pédagogiques

À l’issue de l’activité, vous serez capable de :

- réutiliser un `Dockerfile` déjà fonctionnel ;
- déployer un conteneur sur une VM Docker commune ;
- respecter une convention de ports, de noms ou de chemins ;
- versionner le projet avec Git ;
- publier le même projet sur Render ;
- comparer environnement interne et mise en ligne publique.

---

## Prérequis techniques

- Activité 1 validée.
- Docker fonctionnel sur le poste étudiant ou la VM étudiante.
- Un compte GitHub.
- Un compte Render.
- Une VM Docker mutualisée préparée par l’enseignant.

Render prend en charge le déploiement Docker depuis un dépôt Git contenant un `Dockerfile`, ou à partir d’une image Docker déjà construite.

---

## Arborescence attendue

```text
portfolio-docker-public/
├── Dockerfile
├── compose.yaml
├── README.md
└── portfolio/
    ├── index.html
    ├── style.css
    ├── script.js
    └── ressources/
```

---

## Partie 1 — Réutiliser l’image du portfolio

**Durée indicative : 10 minutes**

Reprenez le projet de l’activité 1 et vérifiez qu’il fonctionne toujours localement :

```bash
docker build -t portfolio-prenom-nom:1.0 .
docker run --rm -d --name portfolio-test -p 8081:80 portfolio-prenom-nom:1.0
```

Vérifiez dans le navigateur :

```text
http://localhost:8081
```

Cette étape évite de déployer sur un environnement partagé un projet qui ne fonctionne pas déjà localement.

---

## Partie 2 — Déployer sur la VM Docker mutualisée

**Durée indicative : 30 minutes**

Une VM Debian 13 avec Docker Engine est installée et accessible à l'adresse 172.17.2.230. Vous disposez d'un compte sur cet VM, avec pour login votre identifiant de SIO et le mot de passe Btssio2017 (voir aussi en annexe).

- Copiez votre projet, via SSH ou SCP, portfolio-docker-public/ sur la VM mutualisée, dans votre dossier /home/login puis placez-vous dans le dossier du projet.

**Docker Compose** permet de décrire et de lancer plusieurs conteneurs Docker à partir d’un fichier de configuration YAML, généralement nommé compose.yaml.
Il centralise la définition des services, ports, réseaux, volumes et variables nécessaires à une application.

- Créez, dans le dossier portfolio-docker-public/ un fichier `compose.yaml` adapté à la convention fournie par l’enseignant. Vous utiliserez un port fourni par l'enseignant, en annexe. Docker Compose permet de définir vos besoins, en suivant l'exemple ci-dessous :

```yaml
services:
  portfolio:
    build: .
    container_name: portfolio-prenom-nom
    ports:
      - "8105:80"
    restart: unless-stopped
```

- Lancez le service :

```bash
docker compose up -d --build
```

Vérifiez :

```bash
docker compose ps
docker compose logs
```

- Testez depuis votre poste :

```text
http://172.17.2.230:8105
```

---

## Partie 3 — Versionner le projet dans GitHub

**Durée indicative : 20 minutes**

Initialisez ou mettez à jour le dépôt Git :

```bash
git init
git add .
git commit -m "Version initiale du portfolio public"
git branch -M main
git remote add origin URL_DU_DEPOT
git push -u origin main
```

Le dépôt devient ici à la fois un support de documentation, un historique de versions et une source de déploiement pour la plateforme cloud.

---

## Partie 4 — Publier sur Render

**Durée indicative : 30 minutes**

Render est une plateforme cloud (PaaS Platform as a Service) qui permet de déployer et d’héberger des applications Web sans administrer directement un serveur.
Elle peut construire et exécuter une application à partir d’un dépôt GitHub et d’un Dockerfile.
Chaque service reçoit une URL publique en HTTPS et peut être redéployé automatiquement après un git push.

- Allez sur la page de Render : https://render.com/
- Créez un **Web Service** dans Render :

  1. Créez-vous un compte utilisant votre compte GitHub.
  2. Choisissez le runtime **Docker**.
  3. Choisissez un hébergement en Europe (Francfort).
  4. Nommez le web service portfolio-prenom-nom
  5. Vérifiez le chemin du `Dockerfile`.
  6. Lancez le déploiement.

Render indique que, pour un projet Docker, il construit l’image à partir du `Dockerfile` du dépôt ou exécute une image existante.

Une fois l’URL publique fournie, du type https://portfolio-prenom-nom.onrender.com/, testez le site :

- affichage général ;
- chargement CSS ;
- images ;
- HTTPS ;
- accessibilité depuis un autre poste.

Votre service Render est connecté à votre dépôt GitHub, il suffit de modifier le portfolio, puis d’envoyer la modification sur la branche surveillée — généralement main. Render détecte le nouveau commit, reconstruit l’image avec le Dockerfile, puis redéploie automatiquement le service.

- Faîtes un test de modification et vérifiez sa publication sur le site public.


---

## Travail à rendre

Vous devez envoyer un e-mail à votre professeur, indiquant :

1. l’URL de votre portfolio sur la VM mutualisée (172.17.2.230) ;
2. l'adresse du dépôt GitHub ;
3. l'URL publique de votre portfolio.

---

## Transition vers l’activité 3

Dans l’activité 3, le déploiement ne sera plus seulement “réalisé avec Docker Compose” : il sera **pensé comme une activité IaC**. Vous devrez décrire de manière plus rigoureuse l’infrastructure applicative attendue — services, réseaux, volumes, proxy et variables — afin de pouvoir la détruire puis la recréer à l’identique sur une nouvelle machine. Docker Compose est justement conçu pour définir services, réseaux et volumes dans un fichier déclaratif unique.

## ANNEXE : login et ports sur la VM mutualisée
| Nom | Prénom | Login | Port |
|---|---|---|---:|
| ALLOCA | Alexian | aalloca | 8081 |
| BADOUARD | Léo | lbadouard | 8082 |
| BUSSON | Manoa | mbusson | 8083 |
| CHALONY | Eliaz | echalony | 8084 |
| CREACH | Janis | jcreach | 8085 |
| EDEL | Tom | tedel | 8086 |
| FLASON | Ruth | rflason | 8087 |
| GUEI | Auxile | aguei | 8088 |
| HEITZ | Youwen | yheitz | 8089 |
| LE BIVIC | Emma | elebivic | 8090 |
| LE CAER | Yaël | ylecaer | 8091 |
| LE REST | Dimitry | dlerest | 8092 |
| LENAIN | Thomas | tlenain | 8093 |
| MALBROUQUI | Marie | mmalbrouqui | 8094 |
| PAREIGE | Alexandre | apareige | 8095 |
| PELLEN | Timothé | tpellen | 8096 |
| PILORGET | Ivan | ipilorget | 8097 |
| PRIGENT | Kamy | kprigent | 8098 |
| REBOUX | Titouan | treboux | 8099 |
| STENTZEL | Ewenn | estentzel | 8100 |
| TARTIVEL | Maxime | mtartivel | 8101 |
| TONNERRE | Aurélien | atonnerre | 8102 |
| VINCENT | Océane | ovincent | 8103 |
| VU | Théo | tvu | 8104 |
