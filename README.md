# Cloud Run - Cours Complet

## Introduction à Cloud Run

### Qu'est-ce que Cloud Run ?

Cloud Run est une plateforme **serverless entièrement gérée** qui permet de déployer et d'exécuter des applications conteneurisées. Elle s'adapte automatiquement à la charge de travail et ne facture que l'utilisation réelle des ressources.

### Principales fonctionnalités :

- **Serverless** : Pas de gestion d'infrastructure requise.
- **Mise à l'échelle automatique** : Augmente ou réduit le nombre d'instances en fonction du trafic.
- **Paiement à l'usage** : Vous ne payez que les ressources réellement consommées.
- **Langage universel** : Compatible avec tout langage/framework pouvant être conteneurisé.
- **Sécurisé** : Intégré avec IAM et Secret Manager pour une gestion sécurisée des accès et des secrets.

## Intégration avec l'écosystème Google Cloud

Cloud Run s'intègre à l'écosystème plus large de Google Cloud, ce qui vous permet de créer des applications complètes et riches en fonctionnalités. Vous pouvez utiliser :

- **Les services de stockage de données** tels que **Cloud SQL, Cloud Storage, Firestore**, et d'autres.

## Avantages de Cloud Run

- **Serverless** : Cloud Run est une plateforme serverless, ce qui signifie que vous n'avez pas à vous soucier de l'approvisionnement et de la gestion de l'infrastructure.
- **Support de l'intégration et du déploiement continus (CI/CD)** : Compatible avec des référentiels de code source tels que **GitHub, Bitbucket, ou Cloud Source Repositories**.
- **Facturation à l'usage** : Vous ne payez que pour les ressources utilisées, optimisant ainsi les coûts.

## Configuration de Cloud Run

### Prérequis

Avant de déployer sur Cloud Run, assurez-vous d'avoir :

- Un projet Google Cloud ([Créer un projet](https://console.cloud.google.com/))
- Installé `gcloud` CLI ([Guide d'installation](https://cloud.google.com/sdk/docs/install))
- Installé Docker ([Télécharger Docker](https://www.docker.com/get-started))
- Activé l'API Cloud Run :
  ```sh
  gcloud services enable run.googleapis.com
  ```

### Déploiement d'une première application

1. **Créer une application de test**

   ```sh
   echo 'FROM gcr.io/distroless/base
   COPY index.html /var/www/html/index.html
   CMD ["python3", "-m", "http.server", "8080"]' > Dockerfile
   ```

2. **Construire l'image Docker**

   ```sh
   docker build -t gcr.io/YOUR_PROJECT_ID/hello-cloud-run .
   ```

3. **Pousser l'image vers Google Container Registry**

   ```sh
   gcloud auth configure-docker
   docker push gcr.io/YOUR_PROJECT_ID/hello-cloud-run
   ```

4. **Déployer sur Cloud Run**

   ```sh
   gcloud run deploy hello-cloud-run \
       --image gcr.io/YOUR_PROJECT_ID/hello-cloud-run \
       --platform managed \
       --region us-central1 \
       --allow-unauthenticated
   ```

5. **Récupérer l'URL du service**

   ```sh
   gcloud run services describe hello-cloud-run --format 'value(status.url)'
   ```

## Surveillance et journalisation

- **Cloud Logging & Monitoring** pour suivre les logs et la performance :
  ```sh
  gcloud logging logs list
  ```

## Résumé

### En résumé :

- **Cloud Run est un produit serverless géré sur Google Cloud** qui exécute et met à l'échelle automatiquement des conteneurs à la demande.
- **Vous pouvez déployer n'importe quelle application conteneurisée** qui gère des requêtes web.
- **Cloud Run prend en charge les requêtes HTTPS** et gère leur routage vers votre application.
- **Cloud Run exécute vos applications sous forme de services ou de tâches (jobs)** selon vos besoins.

## Résumé

Cloud Run est une plateforme flexible et performante pour exécuter des applications conteneurisées sans gestion d'infrastructure. Elle est idéale pour :

- Héberger des **APIs** et **microservices**.
- Gérer des **jobs en arrière-plan** et des **tâches planifiées**.
- **Intégration facile** avec les services Google Cloud.

## Prochaines étapes

- **Explorer la documentation** : [Cloud Run Docs](https://cloud.google.com/run/docs)
- **Tester un déploiement** en conditions réelles.
- **Expérimenter avec les fonctionnalités avancées** (scaling, trafic management, monitoring, etc.).

---

Ce cours sur Cloud Run est maintenant terminé ! 🚀

