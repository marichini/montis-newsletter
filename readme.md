# Guide Complet Newsletter Montis

**Automatisation Make.com + GitHub + Claude MCP**

Juin 2026 - Par Matteo Marochini

---

## Table des matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Architecture](#2-architecture)
3. [Les scénarios Make](#3-les-scénarios-make)
4. [Gestion des flux RSS](#4-gestion-des-flux-rss)
5. [GitHub et Cloudflare](#5-github-et-cloudflare)
6. [Utiliser Claude + Make MCP](#6-utiliser-claude--make-mcp)
7. [Visual Studio Code](#7-visual-studio-code)
8. [Bonnes pratiques](#8-bonnes-pratiques)
9. [Guide de dépannage](#9-guide-de-dépannage)
10. [Conclusion](#conclusion)

---

## 1. Vue d'ensemble

La **newsletter Montis** est un système entièrement automatisé qui génère deux newsletters hebdomadaires (Daily et Weekly) sur les sujets de DLT, tokenisation des titres, régulation MiCA, et secteur bancaire de gros. Le système fonctionne 24/7 sans intervention humaine.

### Processus automatisé

- ✅ Récupère articles depuis 8+ flux RSS
- ✅ Résume avec Llama 3.3 (OpenRouter)
- ✅ Génère HTML automatiquement
- ✅ Pousse sur GitHub
- ✅ Servit via Cloudflare Workers

### Timeline et statut

- **Créé** : Début 2026 par Matteo Marochini
- **Statut** : Opérationnel et en mode handover
- **Dernière vérification** : 17 juin 2026 (Weekly Digest confirmé à 07:30 Paris)
- **Hébergement** : eu1.make.com, GitHub, Cloudflare Workers

---

## 2. Architecture

### Vue d'ensemble technique

| Composant | Rôle | Details |
|-----------|------|---------|
| Make.com | Orchestration | 2 scénarios actifs |
| RSS Feeds | Sources | 8+ sources |
| OpenRouter | LLM API | Llama 3.3 |
| GitHub | Stockage | MMaro06/montis-newsletter |
| Cloudflare | Web server | montis-newsletter.letzgomontis.workers.dev |

### Flux de données

Chaque jour :

1. **07:30 Paris** → Make exécute Daily Digest
2. **RSS feeds** → HTTP GET (Ledger Insights, PostTrade360, etc.)
3. **OpenRouter API** → Résume les articles (Llama)
4. **GitHub API** → Push HTML généré
5. **Cloudflare Workers** → Servit le site publique

Le Weekly Digest suit le même processus mais ajoute aussi des données de marché live (CoinGecko, Yahoo Finance).

---

## 3. Les scénarios Make

### Daily Digest (ID: 6200212)

| Propriété | Valeur |
|-----------|--------|
| Horaire | Lundi-Vendredi à 07:30 Paris (UTC+2) |
| RSS sources | Ledger Insights, PostTrade360, The Block, Future of Finance, Cryptoast, crypto.news, France Post-Marché, CoinTelegraph |
| Placeholders | 7 sections : DATE, TLDR, MONTIS, DLT, REG, SUM, GEO |
| Output | daily-digest.html sur GitHub (main branch) |

### Weekly Digest (ID: 6220310)

| Propriété | Valeur |
|-----------|--------|
| Horaire | Chaque lundi à 07:30 Paris |
| Modules | 13 modules (HTTP GET, 2 RSS fetch, 6 LLM calls, SetVariable, GitHub API) |
| Données de marché | CoinGecko (cryptos), Yahoo Finance via allorigins.win (actions) |
| Temps d'exécution | ~96 secondes (confirmé 17 juin 2026) |

### Structure générale des scénarios

Chaque scénario suit ce pattern :

1. **Trigger** (horaire schedulé)
2. **HTTP GET** pour récupérer le template HTML depuis GitHub
3. **2 à 8 appels RSS** HTTP GET
4. **Appels LLM** (OpenRouter) pour résumer
5. **SetVariable2** pour construire le HTML final (chaîne de replace())
6. **GitHub API GET** pour récupérer le SHA du fichier
7. **GitHub API PUT** pour pusher le contenu

---

## 4. Gestion des flux RSS

### Comment marche un flux RSS ?

RSS = Really Simple Syndication. C'est un format XML qui liste les articles d'un site web. Chaque article a :

- Titre
- Description (résumé ou contenu partiel)
- Lien vers l'article complet
- Date de publication

### Sources RSS actuelles

#### Daily Digest

| Source | Section | Status |
|--------|---------|--------|
| Ledger Insights | DLT & Settlement | ✓ Actif |
| PostTrade360 | Post-Trade | ✓ Actif |
| The Block | Crypto & DLT | ✓ Actif |
| Future of Finance | Fintech & Banking | ✓ Actif |
| Cryptoast | Crypto EU | ✓ Actif |
| crypto.news | Digital Assets | ✓ Actif |
| France Post-Marché | Post-Trade FR | ✓ Actif |
| CoinTelegraph | Crypto & News | ✓ Actif |

#### Sources futures (En cours de discussion)

- **BIS** (Bank for International Settlements) - RSS communiqués de presse (probablement dispo)
- **ESMA** (European Securities and Markets Authority) - RSS non trouvé (Drupal CMS, pas d'URL publique)

### Ajouter une nouvelle source RSS

⚠️ **IMPORTANT** : Toujours vérifier qu'une source RSS est accessible AVANT de modifier un scénario Make.

#### Étapes

1. Cherche le flux RSS sur le site (cherche /rss, /feed, ou une icône RSS)
2. Test l'URL avec un HTTP GET sur ton navigateur ou avec `curl`
3. Vérifie que tu reçois du XML valide avec des articles
4. Si OK → Partage l'URL avec Matteo pour approbation
5. Une fois approuvé → Ajoute un module HTTP GET au scénario Make
6. Ajoute un module LLM pour résumer les articles
7. Ajoute le résumé au SetVariable2 qui construit le HTML

---

## 5. GitHub et Cloudflare

### GitHub : Le stockage des newsletters

- **Repo** : MMaro06/montis-newsletter
- **Branch** : main (seule branche utilisée)
- **Fichiers principaux** :
  - `daily-digest.html` - Généré automatiquement
  - `weekly-digest.html` - Généré automatiquement
  - `daily-digest-template.html` - Template (modifié manuellement par Matteo)
  - `weekly-digest-template.html` - Template (modifié manuellement par Matteo)
  - `README.md` - Documentation complète

### Comment Make pousse vers GitHub

#### 1. GitHub API GET (module 12 du Weekly)

- Récupère les métadonnées du fichier (surtout le SHA)

#### 2. GitHub API PUT (module 13 du Weekly)

- Pousse le contenu HTML généré
- Inclut le SHA pour valider la version (sinon erreur 422)
- Inclut un message de commit auto-généré

### Cloudflare Workers

- **URL** : montis-newsletter.letzgomontis.workers.dev
- **Rôle** : Serveur web qui affiche les newsletters
- **Intégration** : Fetch les fichiers HTML depuis GitHub (raw.githubusercontent.com)

Les données de marché live (CoinGecko, Yahoo Finance) sont chargées en client-side JavaScript.

---

## 6. Utiliser Claude + Make MCP

### Qu'est-ce que MCP ?

**MCP** = Model Context Protocol. C'est un protocole qui permet aux LLMs (comme Claude) de parler à des services externes (Make, Gmail, Slack, etc.) sans apprendre à faire des appels API manuellement.

**Exemple** : Au lieu de dire "Peux-tu faire un appel GET à l'API GitHub ?", tu dis "Modifie mon scénario Make pour inclure une nouvelle source RSS", et Claude fait directement via le connecteur Make MCP.

### Comment activer Make MCP sur Claude

1. Va sur claude.ai
2. Compte → Settings → Connectors
3. Cherche "Make.com"
4. Clique "Connect" et authentifie avec ton compte Make

### Cas d'usage : Modifier un scénario Make via Claude

#### Prompt exemple

```
Je dois ajouter BIS press releases (URL RSS: https://www.bis.org/press/rss.xml) au Daily Digest. 
Utilise Make pour :
- Ajoute un module HTTP GET qui récupère ce flux
- Ajoute un module LLM pour résumer les articles
- Mets les résumés dans une nouvelle section du HTML
```

#### Claude va

- Vérifier le scénario 6200212
- Ajouter les modules nécessaires
- Tester et valider

### Limitations & avertissements

⚠️ Ne modifie JAMAIS le scénario 6199804 (Setup/Debug) - doit rester inactif

⚠️ Toujours tester les changements en environment de test d'abord

⚠️ Les templates HTML doivent être modifiés manuellement sur GitHub, pas dans Make

⚠️ Les fichiers template ne doivent jamais être base64-encodés dans Make (limite de 50-60KB)

---

## 7. Visual Studio Code

### Pourquoi VS Code ?

VS Code t'permet de :

- Éditer les templates HTML localement
- Valider le HTML/CSS avant de pusher sur GitHub
- Utiliser Git directement depuis l'éditeur
- Faire du dev local avant de déployer sur prod

### Setup : Cloner le repo

Via VS Code :

1. `Ctrl+Shift+P` (ou `Cmd+Shift+P`) → "Clone Repository"
2. Colle : `https://github.com/MMaro06/montis-newsletter`
3. Choisis un dossier local

### Workflow : Modifier et pusher

1. Édite `daily-digest-template.html` ou `weekly-digest-template.html`
2. `Ctrl+Shift+G` (Source Control)
3. Ajoute un message de commit (ex: "Update Weekly template with new styles")
4. Clique "Commit & Push"

### Extensions recommandées

- **Live Server** - Preview HTML en live
- **Prettier** - Formatage auto du HTML/CSS
- **HTML Validator** - Vérifie le HTML

---

## 8. Bonnes pratiques

### Règles d'or (établies par Matteo)

#### 1. Vérifier les RSS avant de modifier Make

- Cherche l'URL avec web_fetch, pas juste en supposant que le site a un RSS
- Certains sites (Central Banking, Global Custodian) cachent les RSS derrière login → échoue silencieusement

#### 2. Ne jamais dépasser 50-60KB par blueprint

- Les fichiers HTML de 40-50KB → Erreur 500 sur scenarios_update
- Solution : Fetch les templates depuis GitHub en runtime, pas en base64

#### 3. Ordre des champs OpenRouter HTTP

- DOIT inclure : `shareCookies`, `parseResponse`, `allowRedirects`, `stopOnHttpError`, `requestCompressedContent`
- `jsonStringBodyContent` DOIT être en dernier
- Sinon : BundleValidationError

#### 4. SetVariable2 : pas de & ou concat() imbriqués

- Les expressions complexes → isinvalid: true
- Solution : Décompose en étapes, utilise les références de modules (ex: `10.week_date`)

#### 5. GitHub SHA : doit être 2.body.sha

- Pas 2.sha ou 2.data.sha

#### 6. Array indexing : 1-based avec [1]

- Les réponses OpenRouter : `choices[1].message.content`
- Sinon : retourne un objet qui se base64-encode mal

#### 7. toString() pour binaire

- `raw.githubusercontent.com` retourne du binaire
- Toujours : `toString()` avant `replace()`

#### 8. Pas de chevauchement RSS

- Chaque section doit avoir des sources distinctes
- Sinon : mêmes articles dans plusieurs sections

#### 9. TOUJOURS approuver avant d'implémenter

- Partage les candidats avec Matteo
- Implémente uniquement après OK

---

## 9. Guide de dépannage

### Le scénario a échoué ? Vérifier d'abord

1. Va sur Make → Executions
2. Vérifie le statut de la dernière exécution
3. Clique sur l'exécution → "Logs" pour voir les détails

### Erreurs courantes

| Erreur | Solution |
|--------|----------|
| HTTP 404 | URL cassée. Vérifie le lien RSS ou GitHub |
| HTTP 403 | Authentification manquante. Vérifie token GitHub/OpenRouter |
| HTTP 422 | Vérifie le SHA GitHub (doit être `2.body.sha`) |
| `isinvalid` | SetVariable2 invalide - simplifie les expressions |
| BundleValidationError | Champs OpenRouter mal organisés - respecte l'ordre |
| Timeout | Trop de modules. Optimise l'exécution |
| Valeurs vides | Vérifie que stopOnHttpError est false (pour skip erreurs RSS) |

### Le scénario s'arrête avec "BundleValidationError"

**Cause** : Module HTTP OpenRouter a les mauvais champs

**Solution** :

- Vérifie l'ordre des champs : `shareCookies`, `parseResponse`, `allowRedirects`, `stopOnHttpError`, `requestCompressedContent`, (puis) `jsonStringBodyContent`
- Ajoute les champs manquants

### GitHub push échoue (422 Unprocessable Entity)

**Cause** : SHA invalide ou mismatch

**Solution** :

- Vérifie que tu récupères le SHA via module 2 (GET `/repos/MMaro06/montis-newsletter/contents/...`)
- Vérifie que tu références `2.body.sha` (pas `2.sha`)
- Vérifie que le message de commit n'est pas vide

---

## Conclusion

La newsletter Montis est un système complet et automatisé. Tu as maintenant tous les outils pour :

- ✅ Comprendre comment ça marche
- ✅ Le maintenir sans casser les choses
- ✅ Améliorer avec de nouvelles sources RSS
- ✅ Utiliser Claude + Make MCP pour automatiser encore plus
- ✅ Développer localement avec VS Code

### Règles essentielles

- **Toujours vérifier** avant d'implémenter
- **Toujours approuver** avec Matteo
- **Templates sur GitHub**, pas dans Make
- **Max 50-60KB** par blueprint
- **Respecter l'ordre** des champs OpenRouter

---

**Bon dev ! 🚀**

*Document créé par Matteo Marochini - Montis Digital - Juin 2026*
