# Guide opérationnel Alfred

## Ajouter un nouvel agent
1. **Dupliquer le template** dans n8n (`templates/Agent_*`).
2. **Créer le workflow** dans le dossier `10_agents/` et nommer `Agent_<Domaine>`.
3. **Définir le contrat d'entrée** :
   ```json
   {
     "tenant_id": "string",
     "user_id": "string",
     "intent": "string",
     "text": "string",
     "slots": { "...": "..." },
     "context": {}
   }
   ```
4. **Importer les utilitaires** :
   - `HTTP Request - GetConnection` (récupération tokens Supabase)
   - `Execute Workflow - RefreshToken` (si OAuth)
   - `Set - Output` avec `{status, output_message, artifacts, metadata}`.
5. **Brancher les nœuds métiers** (Google, HubSpot, etc.).
6. **Ajouter le workflow** dans `Alfred_Orchestrateur` :
   - Mettre à jour le `Switch Intent` (valeur intent → nom workflow).
   - Ajouter le nouveau nœud `Execute Workflow` + branche `error`.
7. **Tester** avec `tenant_id` de staging + jeu de requêtes.
8. **Exporter le JSON** et commit dans `templates/`.

## Connecter un compte utilisateur
1. **Créer le tenant** dans Supabase (`tenants` + `users`).
2. **Inviter l'utilisateur** à se connecter via Supabase Auth.
3. **Depuis le portail web** :
   - L'utilisateur clique sur « Connecter {App} ».
   - OAuth redirige vers une Edge Function qui stocke `access_token`, `refresh_token`, `expires_at` dans `connections`.
4. **Tester la connexion** via workflow `Setup_CreateCredentials` :
   - Entrée `{tenant_id, app}`.
   - Le workflow crée/actualise les credentials n8n (`POST /rest/credentials`).
   - Réalise une requête test (ex. `GET /me`).
5. **Vérifier** que l'agent récupère bien la connexion (log Supabase).

## Communication via Telegram
1. **Créer le bot** via `@BotFather` et récupérer le token.
2. **Configurer le webhook** :
   - Staging : utiliser `Telegram Trigger` avec polling.
   - Prod : configurer Caddy pour relayer `https://alfred.domain/bot` → n8n webhook.
3. **Sécuriser** :
   - Ajouter `X-Signature` (HMAC du body) côté proxy.
   - Vérifier la signature dans `Alfred_Orchestrateur` avant traitement.
4. **Déployer** le workflow `Alfred_Orchestrateur`.
5. **Tester** :
   - Message texte → classification + réponse.
   - Message vocal → transcription + réponse.
   - Commandes `/help`, `/status` (nœud `Switch` spécifique).
6. **Surveiller** via Uptime Kuma et logs Supabase (`runs`).

## Gestion des erreurs et logs
- Chaque agent doit avoir une branche `error` → workflow `Agent_Error_Handler`.
- `Agent_Error_Handler` enregistre l'erreur (`runs.status = 'error'`) et répond « Un humain vous recontactera ».
- Les métadonnées LLM (tokens, modèles) sont persistées dans `runs` pour le suivi des coûts.

## Maintenance
- **Backups** :
  - `cron` appelle `docker exec n8n pg_dump` puis `rclone` vers Google Drive.
  - Workflow `Backup_Verify` teste une restauration hebdomadaire sur une base temporaire.
- **Mises à jour n8n** :
  - Staging → tests → export JSON → import prod.
  - Garder `render.yaml` et `.env` synchronisés.
- **Monitorings** :
  - Uptime Kuma : orchestrateur, Supabase, proxy.
  - Alertes Discord/Slack pour erreurs critiques.

Ce guide complète la documentation d'architecture et permet de faire évoluer Alfred rapidement et en sécurité.
