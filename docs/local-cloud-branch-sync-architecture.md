# Synchronisation local-cloud par branche : comparaison Gloowflix et Noverfly POS

## Objet et périmètre

Ce document décrit l'architecture déjà observée dans les dépôts de production. Il ne modifie pas le comportement des services.

- Référence cloud : `D:\SERVEUR\GLOOWFLIX\V1`
- Référence POS local-first : `D:\Logiciel\Lagrace20260529\noverfly-pos`
- Une **branche** est un magasin/site opérationnel identifié dans le POS par `local_site_id` (Noverfly) ou `storeId` (Gloowflix).

## Conclusion

Gloowflix et Noverfly ne résolvent pas exactement le même problème :

| Sujet | Gloowflix Cloud | Noverfly POS |
|---|---|---|
| Données configurables par client | Collections JSON et scripts cloud, isolés par `tenantId` + site/projet | Collections cloud nommées + schéma local SQLite métier |
| Magasin / branche | `Store`, identifié par `storeId` et rattaché à un `tenantId` | Installation/site local, identifié par `local_site_id` |
| Source immédiate de vérité | PostgreSQL cloud | SQLite local ; la vente ne dépend pas d'Internet |
| Hors connexion | Pas de moteur de réplication locale trouvé | Oui : outbox transactionnelle persistante |
| Synchronisation cloud-local | API cloud et WebSocket temps réel, mais pas de protocole de réplication par poste | Push idempotent + pull delta + checkpoints + reprise après panne |
| Exécution de scripts client | Sandbox JavaScript, versions, publication, rollback, jobs BullMQ | Ce n'est pas la responsabilité du moteur de synchronisation POS |

Il ne faut donc pas recopier le modèle Gloowflix comme s'il était déjà une synchronisation local-first. Le bon modèle pour le POS est : **Noverfly garde son moteur local-first ; Gloowflix fournit la couche cloud programmable qui peut servir à configurer les collections et automatisations.**

## Ce que fait réellement Gloowflix

### 1. Isolation multi-tenant et par projet

Les Collections et les scripts DevAPI sont séparés par `tenantId` et `projectId`/site. Un client peut créer son propre schéma de collections et ses scripts JavaScript. Le code est stocké en base, exécuté dans une sandbox, versionné et publiable ; les tâches déclenchées ou planifiées passent par BullMQ.

Cette couche est adaptée à une configuration par client : collections sur mesure, mapping de données, workflows, API publique calculée et automatisations. Elle n'installe pas de code local sur les postes de caisse et ne synchronise pas une base SQLite.

Références :

- `GLOOWFLIX/V1/docs/noverfly-cloud-scripts.md`
- `GLOOWFLIX/V1/docs/devapi/cloud-scripts-operational-guide.md`

### 2. Magasins et appareils POS centralisés

Le module POS Gloowflix possède `Store`, `POSDevice`, stock, ventes et profils d'imprimante. Les routes utilisent `storeId` et les magasins sont rattachés à un `tenantId`. Les ventes sont enregistrées directement dans une transaction PostgreSQL puis un événement WebSocket est publié.

Ce modèle sépare correctement les magasins dans le cloud, mais il reste **cloud-centralisé** : aucune outbox locale, aucun curseur de pull, aucune file de reprise par appareil n'a été identifié dans ce module.

Références :

- `GLOOWFLIX/V1/src/modules/pos/pos.service.ts`
- `GLOOWFLIX/V1/src/modules/pos/pos.routes.ts`

## Ce que fait réellement Noverfly POS

Le flux Noverfly est local-first :

```text
Interface React
  -> API locale /api/local/*
  -> local-server
  -> SQLite (écriture métier + outbox)
  -> sync_operations
  -> SyncPusher
  -> Collections Noverfly Cloud
  -> SyncPuller / SyncDeltaApplier
  -> SQLite + WebSocket local
```

Une action de caisse est d'abord écrite dans SQLite. L'opération à envoyer est conservée dans `sync_operations` avec une clé d'idempotence. Quand le réseau revient, le serveur local pousse les éléments en attente puis récupère uniquement les changements nouveaux du cloud.

Les éléments fondamentaux sont :

- `sync_operations` : outbox durable ; états `pending`, `sent`, `acked`, `failed`, `conflict` ;
- `sync_checkpoints` : curseur par `local_site_id`, direction et type d'entité ;
- `cloud_mappings` : correspondance entre identifiants locaux et cloud ;
- `local_site_id` : identité de la branche/installation utilisée dans les documents cloud ;
- `idempotency_key` : évite une double vente lors d'un retry ;
- pull delta : la reprise ne recharge pas toute la base ;
- WebSocket cloud et scheduler : réveillent le pull sans laisser l'interface contacter le cloud directement.

Références :

- `noverfly-pos/docs/realtime/noverfly-sync-architecture.md`
- `noverfly-pos/docs/realtime/cloud-delta-sync.md`
- `noverfly-pos/docs/architecture/outbox-sync.md`
- `noverfly-pos/packages/sync-engine/src/sync-checkpoint.ts`
- `noverfly-pos/packages/sync-engine/src/sync-delta-applier.ts`

## Contrat de branche recommandé

Pour chaque document synchronisé, conserver explicitement les identités suivantes :

```json
{
  "tenant_id": "organisation-client",
  "project_id": "application-ou-site-cloud",
  "local_site_id": "branche-installation-pos",
  "device_id": "poste-ou-serveur-local",
  "entity_id": "identifiant-metier-local",
  "idempotency_key": "operation-unique",
  "updated_at": "date-iso",
  "revision": 1
}
```

Règles :

1. `tenant_id` empêche tout mélange entre clients.
2. `local_site_id` sépare les données propres à une branche, notamment ventes, stock local et imprimantes.
3. Les données partagées (catalogue, taux, politiques) doivent déclarer leur portée : `tenant`, `branch` ou `device`.
4. Un appareil ne doit jamais appliquer un document destiné à un autre `local_site_id`, sauf règle explicite de diffusion tenant.
5. Une opération est reconnue par sa clé d'idempotence avant tout nouvel envoi.
6. Un checkpoint est enregistré après l'application transactionnelle du delta, jamais avant.

## Répartition des responsabilités

```text
Gloowflix Cloud
  - authentification, tenant, projet
  - collections configurables
  - scripts sandboxés, versions, publication, jobs BullMQ
  - API cloud et événements temps réel

Noverfly local-server (par branche)
  - SQLite et continuité des ventes hors connexion
  - outbox, retry, idempotence, mapping cloud-local
  - checkpoint et application des deltas
  - impression et périphériques locaux

Interface POS
  - parle exclusivement au local-server
  - affiche le statut de synchronisation reçu en WebSocket local
```

## Ce qui est déjà en production et ne doit pas être remplacé sans migration

Le POS utilise déjà les collections cloud `la-grace-*`, les tables SQLite de synchronisation, la reprise des opérations abandonnées au démarrage, et l'identité `local_site_id`. Le remplacer brutalement par les seules routes `Store` de Gloowflix ferait perdre la garantie hors-ligne et la reprise fiable.

Une évolution doit rester compatible avec les opérations en attente, les mappings existants et les checkpoints. Elle demande une migration versionnée et testée ; ce document ne propose aucune modification de production.

## Vérification à effectuer avant toute évolution

1. Identifier la branche par `local_site_id` sur chaque poste réel.
2. Vérifier que chaque document cloud contient sa portée (`tenant_id`, `local_site_id` lorsque nécessaire).
3. Contrôler les opérations en attente et les checkpoints par branche.
4. Tester une vente sans Internet, redémarrer le serveur local, puis rétablir Internet.
5. Vérifier qu'une vente de la branche A n'est ni poussée ni appliquée sur la branche B.

