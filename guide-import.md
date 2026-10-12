# Workflow n8n importable (étapes 2 et 3)

Fichiers de ce dossier :

| Fichier | Rôle |
|---|---|
| `workflow-principal.json` | Gmail Trigger → Préparer le mail → Claude → Valider la réponse → Google Sheets |
| `workflow-alerte.json` | Error Trigger → envoi d'un mail d'alerte |
| `prompt-claude.md` | Version lisible du prompt et du schéma JSON envoyés à Claude |

Les deux workflows sont importés **désactivés** et **sans identifiants** : tu sélectionnes tes credentials dans chaque nœud.

---

## Ce que fait chaque nœud

### 1. Gmail Trigger
Interroge Gmail toutes les heures (à hh:07, hors de l'heure pile où les quotas Google sont les plus sollicités) et sort un item par nouveau mail.
- Recherche : `in:inbox -category:promotions -category:social`
- `Simplify` désactivé : on récupère le texte complet (`text`, `html`), pas seulement l'extrait.
- `Read Status = both` : un mail déjà lu sur ton téléphone avant la vérification est quand même traité.
- Pièces jointes non téléchargées.

### 2. Préparer le mail (Edit Fields)
Ne garde que 7 champs : `message_id`, `thread_id`, `date`, `from_name`, `from_email`, `subject`, `body`.
`body` = texte brut du mail ; si le mail n'a que du HTML, les balises (et les blocs `<style>`) sont retirées. Les espaces multiples sont compactés, puis le texte est coupé à 4 000 caractères.

### 3. Claude (classer et résumer) — nœud HTTP Request
Appelle directement `POST https://api.anthropic.com/v1/messages` avec le credential **Anthropic** de n8n (en-tête `x-api-key` ajouté automatiquement).
- Modèle `claude-haiku-5-5`, effort `low`, `max_tokens` 1024.
- **Sorties structurées** (`output_config.format` + JSON Schema) : l'API garantit un JSON valide avec une catégorie et une priorité prises dans les listes.
- Le corps du mail est placé entre `<email>…</email>` et le prompt dit d'ignorer toute consigne écrite dedans (protection contre l'injection de prompt).
- 3 tentatives, 5 s d'écart, en cas d'erreur réseau ou de surcharge de l'API.

**Écart par rapport à la conception :** j'ai utilisé un nœud HTTP Request plutôt que le nœud « Anthropic ». Le nœud Anthropic ne permet pas forcément de passer `output_config` (sorties structurées), et ses paramètres changent selon la version de n8n. Le HTTP Request est stable et l'appel est visible en entier. La conception prévoyait aussi une température basse : sur Haiku 5.5, toute température autre que 1 renvoie une erreur 400, c'est le schéma JSON qui garantit la stabilité.

### 4. Valider la réponse (Code)
Pour chaque mail :
1. cherche le bloc `text` de la réponse (Haiku 5.5 peut renvoyer un bloc `thinking` avant) et le parse ;
2. si le JSON est illisible, ou si Claude a refusé / a été coupé → `Catégorie = À vérifier`, `Priorité = moyenne`, `Résumé = objet du mail` ;
3. catégorie hors liste → `À vérifier` ; priorité hors liste → `moyenne` ; résumé et action coupés à 200 caractères ;
4. formate les dates en `yyyy-MM-dd HH:mm` (fuseau Europe/Paris) et construit le lien Gmail.

Il renvoie un objet dont les clés sont exactement les en-têtes de colonnes du Google Sheet.

### 5. Google Sheets (Append or Update Row)
- Colonne de correspondance : `Message ID`. Un mail déjà présent est mis à jour au lieu d'être dupliqué.
- Mapping automatique : chaque clé de l'item va dans la colonne du même nom. La colonne `Statut` n'est jamais écrite par n8n, donc ce que tu y notes est conservé.
- `Cell Format = RAW` : les valeurs sont écrites telles quelles. Sans ça, un objet de mail commençant par `=` serait interprété comme une formule par Sheets.
- `Retry On Fail` (5 essais, 5 s d'écart), aussi sur `Lignes en erreur` : l'API Google Sheets renvoie parfois « The service is receiving too many requests from you » (quota par minute dépassé). Une nouvelle tentative quelques secondes plus tard passe, au lieu de faire échouer toute l'exécution.

### Workflow d'alerte
`Error Trigger` reçoit l'échec du workflow principal, puis `Envoyer l'alerte` t'envoie un mail avec le nom du nœud en erreur, le message d'erreur et le lien vers l'exécution.

---

## Mise en place, dans l'ordre

1. **Créer le Google Sheet**, avec un onglet (par ex. `Mails`) et cette ligne 1, exactement :
   `Date | Expéditeur | Email | Objet | Catégorie | Priorité | Résumé | Action | Lien | Message ID | Traité le | Statut`
2. **Créer les credentials dans n8n** (Credentials → Add) : `Gmail OAuth2`, `Google Sheets OAuth2`, `Anthropic` (clé API).
3. **Importer `workflow-alerte.json`** (Workflows → Add → Import from File). Dans `Envoyer l'alerte` : choisis le credential Gmail et remplace `REMPLACER_PAR_TON_ADRESSE@gmail.com`. Enregistre, puis clique **Publish** : depuis n8n 2.x, un workflow d'erreur non publié ne peut pas être choisi comme Error workflow (option grisée avec un triangle).
4. **Importer `workflow-principal-v2.json`** (version durcie, à préférer à `workflow-principal.json`), puis dans chaque nœud :
   - `Gmail Trigger` et `Relire le mail` : credential Gmail ;
   - `Claude (classer et résumer)` : Authentication = Predefined Credential Type, type Anthropic, ton credential ;
   - `Google Sheets` : credential, puis choisis le document et l'onglet dans les listes. **`Column to match on` est réinitialisé à vide après le choix de l'onglet** (constaté le 2026-10-08) : remets `Message ID`, sinon l'exécution échoue avec « The 'Column to Match On' parameter is required ».
   - `Lignes en erreur` (v2) : même credential, même document, même onglet.
5. **Relier l'alerte** : dans le workflow principal, Settings → Error workflow → « Alerte erreurs (Gmail → Claude → Sheets) ».
6. **Premier test manuel** : dans `Gmail Trigger`, clique « Fetch Test Event », puis « Test workflow ». Vérifie la ligne ajoutée dans le Sheet.
7. **Publier** le workflow principal (bouton Publish) seulement après ce test. Dans le bouton Execute, la flèche ⌄ permet de choisir le déclencheur à tester (Gmail Trigger ou Reprise des erreurs).

## Étape 3 : résultats des tests (2026-10-08)

| Test | Résultat |
|---|---|
| Champs du Gmail Trigger (`id`, `threadId`, `date`, `subject`, `from.value[0]`, `text`, `html`) | OK, rien à corriger |
| Exécution complète sur un vrai mail | OK, ligne correcte (date Paris, catégorie, résumé < 200 car.) |
| Doublons (même mail relancé) | OK, une seule ligne, mise à jour via `Message ID` |
| Branche erreur (URL Claude cassée exprès) | OK, ligne `Erreur API` avec la cause dans `Action`, le lot continue |
| Reprise des erreurs | OK, la ligne `Erreur API` est remplacée par le bon classement ; `Relire le mail` sort les mêmes champs que le trigger |
| Coût | 2 950 tokens entrée + 457 sortie (dont 307 de thinking) ≈ 0,0005 $ par mail, ≈ 1,30 $/mois à 80 mails/jour |

Changements de la v2 (`workflow-principal-v2.json`) :
- nœud Claude en « Continue (using error output) » : un échec après 3 essais n'arrête plus le lot ;
- `Marquer en erreur` écrit une ligne `Erreur API` ;
- déclencheur `Reprise des erreurs` toutes les 6 h (à hh:37) qui relit ces mails dans Gmail et les retraite. Chaque passage compte comme une exécution (≈ 120/mois), d'où 6 h et non 1 h.

Limite connue : un échec de Claude ne déclenche plus le mail d'alerte (l'exécution réussit) ; il est visible dans le Sheet. La priorité peut varier (`haute`/`moyenne`) sur les cas limites.

---

## Hypothèses à vérifier

Les noms et versions de nœuds dépendent de la version de n8n. J'ai supposé une n8n Cloud 1.x récente :

| Nœud | Type | Version |
|---|---|---|
| Gmail Trigger | `n8n-nodes-base.gmailTrigger` | 1.2 |
| Edit Fields | `n8n-nodes-base.set` | 3.4 |
| HTTP Request | `n8n-nodes-base.httpRequest` | 4.2 |
| Code | `n8n-nodes-base.code` | 2 |
| Google Sheets | `n8n-nodes-base.googleSheets` | 4.5 |
| Error Trigger | `n8n-nodes-base.errorTrigger` | 1 |
| Gmail | `n8n-nodes-base.gmail` | 2.1 |

Si un nœud s'affiche avec « ? » ou « Install this node » après l'import, dis-moi ta version de n8n (menu Help → About n8n) et j'adapte.

Autres points non vérifiés sur un vrai n8n :
- **Champs du Gmail Trigger** avec Simplify désactivé : j'ai supposé `id`, `threadId`, `date`, `subject`, `from.value[0].name/address`, `text`, `html`. À contrôler sur le premier « Fetch Test Event ».
- **Modèle et tarif** : `claude-haiku-5-5`, 0,10 $ / 0,50 $ par million de tokens (entrée / sortie). Un mail de 4 000 caractères fait environ 1 500 à 2 000 tokens en entrée, donc de l'ordre de 0,0002 $ par mail. À confirmer sur la page de prix Anthropic et à mesurer à l'étape 3.
- **Limite connue, à traiter à l'étape 3** : si l'appel à Claude échoue pour un mail après les 3 tentatives, toute l'exécution s'arrête et les mails de cette heure ne sont pas repris automatiquement (l'alerte te prévient).
