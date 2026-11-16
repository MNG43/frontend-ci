# SamaFile - frontend-ci

Interface web React de SamaFile, une application de gestion des files d’attente pour les services publics. Le frontend utilise Vite et React Router ; les appels API passent par `/api` et `/health`.

| Élément | Valeur |
| --- | --- |
| Runtime | Node.js 22, React 19, Vite |
| Port de développement | `5173` |
| Image Docker | `mng2021/samafile-frontend` |
| Dépôt | [MNG43/frontend-ci](https://github.com/MNG43/frontend-ci) |
| Docker Hub | [mng2021/samafile-frontend](https://hub.docker.com/r/mng2021/samafile-frontend) |

## Fonctionnalités

- Espace usager : consulter les services, prendre et suivre un ticket, consulter son historique.
- Espace agent : consulter sa file, appeler le suivant et mettre à jour le traitement d’un ticket.
- Espace administrateur : gérer les services, guichets et comptes selon le rôle.
- Le suivi actualise l’état de la file par requêtes périodiques à l’API.

## Développement local

Prérequis : Node.js 22 et npm. Depuis ce dossier :

```powershell
npm ci
Copy-Item .env.example .env
npm run dev
```

Vite démarre sur `http://localhost:5173`. Par défaut, `VITE_API_URL` est vide et le proxy de développement transmet `/api` et `/health` à `http://localhost:8000`. Démarrez donc le backend localement ou utilisez la stack décrite dans [`deployment-cd/README.md`](../deployment-cd/README.md).

Pour créer et prévisualiser le build de production :

```powershell
npm run lint
npm run build
npm run preview
```

`VITE_API_URL` peut définir une URL API absolue au moment de la compilation. Laisser la variable vide permet d’utiliser le proxy Nginx de la stack.

## Docker

Le `Dockerfile` construit l’application puis sert les fichiers statiques avec Nginx non privilégié sur le port `8080`. La stack de déploiement utilise le port interne du conteneur et expose l’interface via le proxy sur `http://localhost:8888`.

Pour construire l’image depuis ce dépôt :

```powershell
docker build -t samafile-frontend:local .
docker run --rm -p 8080:8080 samafile-frontend:local
```

## CI et publication

Le `Jenkinsfile` lance oxlint, npm audit, Hadolint et Trivy, puis construit l’image Docker. La publication sur Docker Hub est conditionnée par le paramètre Jenkins `PUBLISH` (désactivé par défaut). Quand il est activé, Jenkins pousse le numéro du build et le SHA du commit ; `latest` est publié sur les branches principales configurées. Les rapports de scan sont archivés dans Jenkins.

### Exécuter le pipeline CI dans Jenkins

Configurez `samafile-frontend-ci` en **Pipeline script from SCM**, avec ce dépôt, la branche
`mng` et le chemin `Jenkinsfile`. Lancez **Build with Parameters** avec `PUBLISH` décoché pour
valider le frontend sans publication. Après un build réussi, cochez `PUBLISH` pour envoyer
l'image sur Docker Hub.

Le job exécute Oxlint, `npm audit` (seuil HIGH), Hadolint, le build Docker et Trivy sur l'image
et les sources. Les rapports `reports/npm-audit.json`, `reports/trivy-image.txt` et
`reports/trivy-fs.txt` sont archivés dans Jenkins. Les avertissements Oxlint ne sont pas des
erreurs de lint; vérifiez toutefois qu'aucune erreur ne bloque le job.

La publication crée des tags au numéro du build et au SHA court du commit. Le tag `latest` est
réservé aux branches `main`, `master` et `develop`; sur `mng`, utilisez le tag effectivement
publié et visible sur Docker Hub pour le déploiement CD.

Pour déclencher automatiquement le job lors d'un changement, configurez **Configurer > Triggers >
Scrutation de l'outil de gestion de version** avec `H/2 * * * *`. Le polling surveille la branche
SCM configurée; poussez un commit sur `mng` et vérifiez qu'un build démarre. Laissez `PUBLISH`
décoché pour les validations ordinaires.

## Configuration

`.env.example` documente `VITE_API_URL`. N’y placez aucun secret : les valeurs Vite sont intégrées au bundle et visibles dans le navigateur.
