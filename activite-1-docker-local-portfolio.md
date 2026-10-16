# Activité 1 — Déployer localement un portfolio conteneurisé

**BTS SIO2 — Bloc 1 : Support et mise à disposition de services informatiques**  
**Durée : 2 heures**  
**Modalité : activité Docker sur Windows ou Linux**

## Compétences visées du bloc 1

Cette activité mobilise principalement les compétences suivantes du bloc 1 du BTS SIO : **mettre à disposition des utilisateurs un service informatique**, **répondre aux incidents et aux demandes d’assistance et d’évolution**, **gérer le patrimoine informatique**, **travailler en mode projet** et **organiser son développement professionnel**.

## Finalité

Dans cette première activité, vous déployez votre portfolio sur votre propre machine à l’aide de Docker. L’objectif est de comprendre les bases : image, conteneur, port publié, test fonctionnel, diagnostic simple et documentation technique.

---

## Objectifs pédagogiques

À l’issue de l’activité, vous serez capable de :

- identifier les fichiers nécessaires à un site statique ;
- construire une image Docker simple à partir d’un serveur Web existant ;
- lancer un conteneur et publier un port ;
- tester l’accès au site depuis un navigateur ;
- diagnostiquer quelques incidents fréquents ;
- documenter la procédure de déploiement.

---

## Prérequis techniques

- Une VM Debian 13 ou un poste avec Docker fonctionnel.
- Un portfolio déjà rédigé en HTML/CSS/JavaScript.
- Un éditeur de texte, par exemple VSCode.
- Un navigateur Web.

Pour Docker, la documentation officielle distingue la construction d’image, l’exécution d’un conteneur et le partage éventuel de l’image ; cette activité se concentre d’abord sur la construction puis l’exécution locale.

---

## Arborescence attendue

Créez sur votre poste un dossier `portfolio-docker-local`, par exemple dans le dossier Documents.

L'arborescence attendue en fin d'activité 1 sera la suivante :

```text
portfolio-docker-local/
├── Dockerfile
├── README.md
├── lancement.bat
└── portfolio/
    ├── index.html
    ├── style.css
    ├── script.js
    └── ressources/
```

---

## Partie 1 — Préparer Docker

**Durée indicative : 20 minutes**

Docker Desktop est la solution officielle pour Windows avec interface graphique.

**Activer WSL2 - Sous-système Windows pour Linux**

WSL2 permet d'exécuter un véritable environnement Linux directement sur Windows 10 ou 11 sans utiliser de machine virtuelle lourde.

Exécutez la commande ci-dessous dans une console PowerShell :

```powershell
wsl --install
```

**Télécharger Docker Desktop**

Téléchargez l'installateur depuis [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/)

**Installez Docker Desktop**
- Exécutez l'installateur
- Cochez "Use WSL 2 instead of Hyper-V" (recommandé)
- Suivez l'assistant

**Configurer Docker Desktop**

Au premier lancement :
- Acceptez les conditions d'utilisation
- Connectez-vous (optionnel pour usage personnel)
- Dans Settings > Resources > WSL Integration, activez l'intégration avec vos distributions.

**Vérifier l'installation**

- Lancez l'application Docker Desktop
- Ouvrez un terminal PowerShell :

```powershell
docker run hello-world
```
- Vérifiez la réponse en console :
```powershell
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

**En cas de problème, vous pouvez utiliser la VM Débian13-docker, mise à disposition par l'enseignant. Sur cette VM, Docker Engine a été installé.**

**Tester le lancement d'une image Docker**

Une image est le modèle immuable à partir duquel Docker créera les conteneurs
Téléchargez l’image Ubuntu 24.04 depuis le registre Docker :
```powershell
docker pull ubuntu:24.04
```
Exécutez maintenant :

```powershell
docker run -it --name ubuntu-test ubuntu:24.04 bash
```
- docker run crée puis démarre un conteneur.
- -it relie le clavier et ouvre un terminal interactif.
- --name ubuntu-test donne un nom facile à réutiliser.
- bash est le programme principal : vous arrivez sur une invite Linux semblable à root@<id>:/#.

Quittez maintenant Bash proprement :
```powershell
exit
```
Vous revenez dans PowerShell : le conteneur est arrêté, car son processus principal (bash) se termine, mais il ne sera pas supprimé.

Relancez le conteneur :
```powershell
docker start -ai ubuntu-test
```
- start redémarre un conteneur arrêté.
- -a rattache la sortie du conteneur au terminal.
- -i transmet votre saisie clavier au conteneur.

Arrêtez de nouveau le conteneur.
Pour afficher la liste des conteneurs, y compris ceux qui sont arrêtés, utilisez la commande :
```powershell
docker ps -a
```
Pour supprimer un conteneur précis, identifié par son nom ou son identifiant :
```powershell
docker rm nom_du_conteneur
```

S’il est encore en cours d’exécution, arrêtez-le puis supprimez-le :
```powershell
docker stop nom_du_conteneur
docker rm nom_du_conteneur
```
Ou forcez directement la suppression :

```powershell
docker rm -f nom_du_conteneur
```
Supprimez les conteneurs créés.

Remarque : vous pouvez également utilisez Docker Desktop (onglet Containers) pour démarrer, arrêter ou encore supprimer vos conteneurs.

---

## Partie 2 — Préparer le portfolio

**Durée indicative : 10 minutes**

Vérifiez que votre site est autonome :

- `index.html` doit exister ;
- les chemins vers les feuilles de style, scripts et images doivent être relatifs ;
- aucun lien ne doit dépendre d’un chemin local Windows ou Linux ;
- les fichiers doivent être clairement organisés.

Exemple correct dans `index.html` :

```html
<link rel="stylesheet" href="style.css">
<script src="script.js" defer></script>
<img src="ressources/photo.jpg" alt="Photo de profil">
```

---

## Partie 3 — Construire l’image Docker du portfolio

**Durée indicative : 10 minutes**

Un Dockerfile est un fichier texte qui décrit les étapes nécessaires pour construire une image Docker : image de base, fichiers à copier, logiciels à installer et commande de démarrage.

Il rend le déploiement reproductible : la même image peut être reconstruite et exécutée de façon identique sur différentes machines.

Dans votre dossier, créez le fichier texte `Dockerfile` (attention, sans extension):

```dockerfile
FROM httpd:2.4-alpine

COPY ./portfolio/ /usr/local/apache2/htdocs/

EXPOSE 80
```

Cette construction repose sur une image existante du serveur Apache HTTP Server, puis y copie le contenu du portfolio dans le répertoire servi par le serveur.  

Construisez ensuite l’image :

```bash
docker build -t portfolio-prenom-nom:1.0 .
```
La commande construit une image Docker à partir du Dockerfile présent dans le dossier courant (.).
L’option -t lui attribue le nom portfolio-prenom-nom et la version 1.0, afin de pouvoir ensuite la lancer ou la partager facilement.


Vérifiez le résultat :

```bash
docker image ls
```

---

## Partie 4 — Lancer le conteneur

**Durée indicative : 10 minutes**

Démarrez un conteneur de test :

```bash
docker run -d --name portfolio-test -p 8081:80 portfolio-prenom-nom:1.0
```

Cette commande démarre en arrière-plan (-d) un conteneur nommé portfolio-test à partir de l’image portfolio-prenom-nom:1.0.
Le port 8081 du poste hôte est relié au port 80 du conteneur : le portfolio sera donc accessible avec http://localhost:8081


Vérifiez le status du conteneur :

```bash
docker ps
docker logs portfolio-test
```

Testez dans le navigateur de votre poste :

```text
http://localhost:8081
```
Le service devient alors accessible par publication d’un port du conteneur vers l’hôte. Cette logique de service exposé constitue une première forme de mise à disposition d’un service informatique.

---

## Partie 5 — Script de lancement du conteneur

**Durée indicative : 20 minutes**

L'objectif est de créer un script batch (fichier lancement.bat), à la racine de votre dossier de travail, qui effectuera toutes les tâches précédentes.

Le script doit construire l’image, supprimer un éventuel ancien conteneur, lancer le nouveau, puis ouvrir le navigateur.

Ecrivez ce script.

```batch
REM Script de lancement du portfolio de PRENOM NOM
echo Bonjour, le script s'exécute.
...
pause
```

---

## Partie 6 — Documenter le déploiement

**Durée indicative : 10 minutes**

Rédigez un fichier `README.md` contenant :

- l’arborescence du projet ;
- les commandes utilisées ;
- l’URL de test ;
- les incidents rencontrés et leurs corrections.

Exemple minimal :

````markdown
# Déploiement local du portfolio — Prénom NOM

## Construction
```bash
docker build -t portfolio-prenom-nom:1.0 .
```

## Lancement
```bash
docker run --rm -d --name portfolio-test -p 8081:80 portfolio-prenom-nom:1.0
```

## Test
http://localhost:8081
````

---

## Travail à rendre

Vous devez remettre au professeur une archive nommée **nom-prenom.zip** contenant le dossier complet du projet avec :
1. le `Dockerfile` ;
2. le `README.md` ;
3. le script de lancement `lancement.bat` ;
4. les fichiers de votre portfolio.

**Le professeur doit pouvoir dézipper votre fichier, exécuter le script (lancement.bat) et voir le site s'afficher dans un navigateur.**

---

## Partie 7 — Annexe - Diagnostiquer et corriger

**Durée indicative : 10 minutes**

Incidents possibles :

- le conteneur ne démarre pas ;
- le port 8081 est déjà utilisé ;
- les images du portfolio ne s’affichent pas ;
- le site affiché n’est pas celui attendu.

Commandes utiles :

```bash
docker ps -a
docker logs portfolio-test
docker image ls
ss -lntp
```

Cette phase travaille directement la compétence liée au traitement d’incidents et à l’assistance autour d’un service informatique. 

---

## Transition vers l’activité 2

Dans l’activité 2, le même portfolio ne sera plus seulement lancé localement. Il sera déployé sur une **VM Docker mutualisée**, puis publié sur **Render**, afin d’introduire la mise en ligne et la présence en ligne prévues dans le bloc 1. Render sait déployer un service à partir d’un `Dockerfile` ou d’une image Docker existante.
