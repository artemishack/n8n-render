# Alfred Multi-Agent Architecture

## 1. Vision générale
Alfred est un assistant IA orchestré dans n8n qui s'expose via Telegram. L'orchestrateur central « Alfred_Orchestrateur » normalise toutes les requêtes entrantes, détecte l'intention et délègue l'exécution à des agents spécialisés. Chaque agent est un workflow n8n autonome qui s'appuie sur des connecteurs natifs et sur des appels LLM (OpenAI ou Claude) pour raisonner et manipuler les applications tierces (Gmail, HubSpot, etc.).

Les objectifs clés sont :
- **Modularité** : chaque domaine fonctionnel = workflow indépendant.
- **Scalabilité horizontale** : ajout/suppression d'agents sans impacter le noyau.
- **Onboarding simple** : l'utilisateur fournit un token/API OAuth une seule fois.
- **Fiabilité** : isolation des erreurs, observabilité, reprise sur incident.

## 2. Topologie n8n

```
┌───────────────────────────────────────────────────────────────────┐
│                       Alfred_Orchestrateur                        │
│                                                                   │
│  Telegram Trigger / Webhook → Normalisation → Intent LLM → Switch │
│                                                                   │
│         ├── Execute Workflow → Agent_Mail                         │
│         ├── Execute Workflow → Agent_Calendar                     │
│         ├── Execute Workflow → Agent_CRM                          │
│         └── … autres agents …                                     │
│                                                                   │
│   Agrégation réponse → Logging Supabase → Telegram Send Message   │
└───────────────────────────────────────────────────────────────────┘
```

- Les **agents** résident dans `10_agents/`.
- Les **utilitaires** communs (validation, récupération de credentials, logging) vivent dans `90_shared/`.
- En production, les triggers publics passent derrière Caddy/Cloudflare pour la signature HMAC et le rate limiting.

## 3. Workflow `Alfred_Orchestrateur`

| Étape | Nœud n8n | Description |
| --- | --- | --- |
| 1 | `Telegram Trigger` (staging) / `Webhook` (prod) | Reçoit les messages Telegram (texte, audio, pièces jointes). |
| 2 | `Set - Normalize Payload` | Transforme l'entrée vers le contrat `{tenant_id, user_id, channel, input_type, text, file_url, metadata}`. |
| 3 | `IF - Audio?` + `OpenAI Whisper` | Transcrit les messages vocaux. |
| 4 | `OpenAI Chat - IntentClassifier` | Few-shot prompt retournant `{intent, confidence, suggested_agent, slots}`. |
| 5 | `Switch Intent` | Route vers l'agent (`Execute Workflow`). |
| 6 | `Wait/Respond` | Attend la sortie JSON de l'agent `{status, output_message, artifacts}`. |
| 7 | `Telegram Send Message` | Renvoie le message final, avec en option un document ou un lien. |
| 8 | `HTTP Request - LogRun` | Écrit un enregistrement dans `supabase.rest/rpc/log_run`. |
| 9 | `Error Trigger` (branch) | Capture les erreurs, notifie, loggue en échec. |

### Intent classifier
- Modèle : `gpt-4.1-mini` ou `claude-3-haiku`. Prompt few-shot avec mapping domaine → agent.
- Retourne `fallback` si `confidence < 0.5` → réponse par défaut ou escalade humaine.

### Observabilité & erreurs
- Chaque branche `Execute Workflow` a un chemin `On Error` qui déclenche `Agent_Error_Handler` (workflow partagé) pour notifier Telegram et enregistrer l'échec.
- Métriques stockées : `tenant_id`, `agent`, `duration_ms`, `tokens`, `status`, `error_message`.

## 4. Agents spécialisés
Tous les agents partagent les caractéristiques suivantes :
- **Entrée** : JSON normalisé `{tenant_id, user_id, intent, text, slots, context}`.
- **Préambule** : récupération des credentials via Supabase (`HTTP Request` sur `/rest/v1/connections?tenant_id=...&app=...`).
- **Raisonnement** : `OpenAI Chat` (ou Claude) pour planifier l'action et rédiger la réponse.
- **Actions API** : nœuds spécifiques (Google Calendar, HubSpot, etc.) ou `HTTP Request` custom.
- **Sortie** : nœud `Set - Format Output` → `{status, output_message, artifacts, metadata}`.
- **Gestion d'erreur** : nœud `Error Trigger` avec retour `{status: "error", output_message, error_code}`.

### 4.1 Agent_Mail (Gmail/Outlook)
1. `Set - Validate Input`
2. `Workflow - GetConnection` (Supabase) → tokens Gmail/Outlook
3. `OpenAI Chat - PlanAction` (détermine `action = send|reply|search`)
4. Branche selon `action` :
   - **send/reply** : `OpenAI Chat - DraftEmail` → `Gmail Send` / `Microsoft Outlook Send`
   - **search** : `Gmail Search` ou `HTTP Request` Graph API
5. `Set - Output` avec message + IDs + éventuellement extraits d'e-mails.

### 4.2 Agent_Calendar (Google/Microsoft)
1. Extraction entités (date, participants) via `OpenAI Chat`
2. Vérification conflits (`Google Calendar - Get Events`) / `Microsoft Graph`
3. Création ou mise à jour d'événement
4. Retourne confirmation + lien calendrier.

### 4.3 Agent_CRM (HubSpot / Salesforce / Pipedrive)
1. `OpenAI Chat - IntentCRM` (create lead, update deal, recherche)
2. `Switch` selon action → appels API correspondants
3. Retourne message avec URL du contact ou du deal.

### 4.4 Autres agents
Tous suivent le même squelette. Les connecteurs n8n existants sont utilisés par défaut. Pour des services non natifs, un nœud HTTP générique s'appuie sur les tokens stockés.

## 5. Gestion des identités & credentials

### 5.1 Modèle Supabase
- `tenants(id, name, plan, status, created_at)`
- `users(id, tenant_id, telegram_user_id, role)`
- `connections(id, tenant_id, app, access_token, refresh_token, expires_at, metadata)`
- `runs(id, tenant_id, agent, status, duration_ms, tokens_in, tokens_out, cost, error_message, created_at)`

### 5.2 Flux de connexion
1. L'administrateur ouvre la page `connect/{app}` sur le portail web (Next.js ou autre).
2. L'utilisateur s'authentifie via Supabase Auth → déclenche un Edge Function/OAuth proxy.
3. Après OAuth, les tokens sont stockés dans `connections` chiffrés côté serveur.
4. n8n récupère dynamiquement les tokens à chaque exécution (`GetConnection` utilitaire).
5. Rafraîchissement automatique : un workflow partagé `RefreshToken` vérifie `expires_at` et appelle l'API OAuth pour renouveler si besoin.

### 5.3 Workflow de setup
Workflow `Setup_CreateCredentials` :
- Entrée `tenant_id` + `app` + `token/secret`
- Création des credentials n8n via API (`POST /rest/credentials`) si nécessaire
- Enregistrement Supabase
- Test de connexion (ping API)
- Retourne statut au frontend.

## 6. Sécurité & fiabilité
- **Webhook HMAC** : Caddy signe la requête `X-Signature = HMAC_SHA256(body, secret)`. n8n vérifie avant traitement.
- **Idempotence** : `Idempotency-Key` stockée dans `runs` → si déjà vu, renvoie la réponse précédente.
- **Rate limiting** : géré par Caddy/Cloudflare.
- **Gestion des erreurs** : chaque agent a un `Catch` qui renvoie un message utilisateur et un log détaillé.
- **Stockage des fichiers** : tous les fichiers entrants sont téléchargés, scannés (ClamAV via HTTP), puis déposés sur Supabase Storage.
- **Monitoring** : Uptime Kuma surveille `Alfred_Orchestrateur` (staging & prod), les agents critiques, et le job de backup.

## 7. Extensibilité & message bus
- Ajout d'un agent : cloner un workflow modèle, définir `app` et configurer le mapping d'intentions.
- Les communications inter-agents passent via `Execute Workflow` ou `Webhook` interne.
- Un message bus (Redis, Supabase Realtime) peut être greffé plus tard pour des tâches longues.

## 8. Mémoire conversationnelle
- Table `conversation_threads(id, tenant_id, user_id, history_vector)` dans Supabase.
- Workflow `Agent_ContextLoader` : `Supabase Vector Search` (pgvector) → récupère les messages pertinents → injecte dans le prompt de l'agent.
- Politique de rétention courte (20–50 tours) pour limiter les coûts.

## 9. Extensions optionnelles
- **Appels vocaux Telegram** : utiliser `OpenAI Whisper` ou `Deepgram` pour transcrire les messages audio et `gpt-4o-mini-tts` pour répondre en audio.
- **Analyse de coûts** : utilitaire `estimate_cost` basé sur tokens + tarifs LLM.
- **Dashboard** : page Supabase + Grafana pour suivre les runs, erreurs, consommation.

## 10. Roadmap livrable MVP
| Semaine | Objectifs |
| --- | --- |
| S1 | Implémenter orchestrateur + agents Mail/Calendar/CRM. |
| S2 | Ajouter logging, monitoring, gestion OAuth. |
| S3 | Finaliser onboarding utilisateur, documentation, QA. |
| S4 | Beta clients pilotes + collecte feedback. |

## 11. Référentiel de déploiement
- Déploiement n8n via Docker Compose (workers + queue Redis optionnelle).
- Sauvegardes quotidiennes `docker exec n8n pg_dump` + `rclone` vers Google Drive.
- Tests de restauration mensuels (workflow `Backup_Verify`).
- Gestion de configuration via `render.yaml` ou `.env` centralisé.

Cette architecture offre un socle robuste et extensible pour Alfred, avec un focus sur la séparation des responsabilités, l'observabilité et un onboarding utilisateur fluide.
