# Déploiement — CityLoop Quest Lille

Guide généré par **CLQ App Factory** pour **Lille** (`lille`).  
Dossier de sortie : `CLQ-LILLE/clq-lille` — URL Netlify cible : **https://clq-lille.netlify.app**

---

## Vue d’ensemble

| Élément | Valeur pour cette ville |
|--------|-------------------------|
| Slug backend / DB | `lille` |
| Package npm | `clq-lille` |
| Site Netlify (suggestion) | `clq-lille` |
| URL publique | https://clq-lille.netlify.app |
| API (Render) | https://cityloopquest-api.onrender.com |
| Région | Hauts-de-France, France |

**Ordre recommandé :** Factory (contenu) → Supabase + Stripe + Render → Google Maps → Netlify → tests → `npm run deploy`.

---

## 1. Génération depuis la Factory

1. Ouvrir **CLQ App Factory** (`npm start` → http://localhost:3847).
2. Charger ou compléter le JSON ville (**Lille**).
3. Lancer **Générer l’app** → sortie dans `CLQ-LILLE/clq-lille`.
4. Vérifier dans la Factory : audios, images POI, quiz, traductions sans erreur bloquante.
5. Tester en local :
   ```bash
   cd "CLQ-LILLE/clq-lille"
   npm install
   npm run dev
   ```
   Ouvrir http://localhost:5173 — en dev, le Service Worker est désactivé automatiquement.

> **API en local :** par défaut l’app pointe vers `https://cityloopquest-api.onrender.com`. Pour forcer une API locale :  
> `?api_base=http://localhost:8081` ou `setApiBase('http://localhost:8081')` en console.

---

## 2. GitHub

1. Créer un dépôt (ex. `clq-lille` ou `CLQ-Lille`).
2. Y pousser le contenu de `CLQ-LILLE/clq-lille` (sans `node_modules/`, sans `.env`).
3. Vérifier `.gitignore` : `.env`, `dist/`, `node_modules/`, secrets.
4. Lier le dépôt à Netlify (branche `main` ou `master`).

---

## 3. Netlify

### 3.1 Création du site

- **Build command :** `npm run build`
- **Publish directory :** `dist`
- **Functions :** `netlify/functions` (voir `netlify.toml`)
- **Node :** 18 (déjà dans `netlify.toml`)

### 3.2 Variables d’environnement (build)

À définir dans **Site settings → Environment variables** (pas la liste complète d’une autre ville — uniquement ce dont **Lille** a besoin) :

| Variable | Usage |
|----------|--------|
| `GOOGLE_MAPS_API_KEY` | Injectée dans `api-key.js` au build (`build.mjs`) |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS` | Formulaire contact / POI (`mail.html`, fonctions Netlify) |
| `POI_STAFF_EMAIL` | Destinataire des propositions POI |
| `POI_APPROVE_SECRET` | Secret d’approbation POI communautaires |
| `SITE_URL` | https://clq-lille.netlify.app |
| `NETLIFY_SITE_ID` + `NETLIFY_AUTH_TOKEN` | Blobs POI (ou `NETLIFY_BLOB_READ_WRITE_TOKEN`) |

Ne **pas** y mettre `STRIPE_SECRET_KEY` (reste sur Render).

### 3.3 Déploiement

```bash
cd "CLQ-LILLE/clq-lille"
npm install
npm run deploy
```

(`npm run deploy` = build + `netlify deploy --prod --dir=dist --no-build`)

Après le premier déploiement, noter le **Site ID** Netlify (UI ou `netlify status`).

---

## 4. Google Cloud — clé Maps

Dans [Google Cloud Console](https://console.cloud.google.com/) → APIs & Services → Credentials → votre clé **Maps JavaScript API** :

**Référents HTTP autorisés :**

- `https://clq-lille.netlify.app/*`
- `http://localhost:5173/*` (dev Vite)
- `https://localhost:5173/*` (dev Vite HTTPS)

Activer : **Maps JavaScript API**, **Places API**, **Directions API** (ou Routes API selon migration future).

---

## 5. Render — API `cityloopquest-api`

Dashboard Render → service API → **Environment** :

| Variable | Exemple / valeur |
|----------|------------------|
| `CLIENT_BASE_URL_LILLE` | `https://clq-lille.netlify.app` |
| `STRIPE_LILLE_FULL_7D_PRICE_ID` | `price_xxx` (depuis Stripe) |
| `STRIPE_LILLE_LITE_7D_PRICE_ID` | si applicable |
| `STRIPE_LILLE_UPGRADE_7D_PRICE_ID` | si applicable |

Mettre à jour **CORS / ALLOWED_ORIGINS** pour inclure `https://clq-lille.netlify.app`.

Dans le code API (si nouvelle ville) :

- Ajouter `lille` aux villes supportées (checkout, claim-session, activate-code).
- Mapper les Price IDs et `CLIENT_BASE_URL_LILLE` dans la config (ex. `STRIPE_PRICE_IDS`, `FRONTEND_URLS`).

Redéployer l’API après modification.

---

## 6. Stripe

1. Créer les **produits / prix** 7 jours pour **Lille** (FULL, LITE, UPGRADE si utilisés).
2. Copier les **Price ID** (`price_…`) dans les variables Render ci-dessus.
3. **Webhook Stripe** → pointer vers **Render** (`https://cityloopquest-api.onrender.com/…`), pas vers Netlify.
4. Ne pas configurer d’URL success/cancel figées dans Stripe : elles sont construites côté API avec `CLIENT_BASE_URL_LILLE`.

Flux utilisateur après paiement : `post-checkout.html` → `claim-session` sur Render → redirection `index.html?from=checkout`.

---

## 7. Supabase

Projet partagé (recommandé) :

1. Table **`cities`** : entrée `slug = 'lille'`, nom `Lille`, métadonnées région.
2. Table **`stripe_products_map`** (ou équivalent) : lier les Price ID Stripe à `lille`.
3. Table **`licenses`** : codes d’activation `xxx-xxx-xxx` rattachés à `city = 'lille'`.

Tester un code manuel sur `activation-manual.html` et via le lien email post-achat.

---

## 8. Garde licence par ville

Le fichier `js/license-city-guard.js` associe le hostname au slug `lille`.  
Après génération Factory, vérifier que la ligne contient bien :

```js
if (host.includes('lille')) return 'lille';
```

**Piège :** `localStorage` est partagé entre tous les sites `*.netlify.app` sur un même navigateur. Une licence **Murcia** ne doit pas débloquer **Lille** — le guard compare `clq_city` / JWT au slug du site.

---

## 9. Checklist avant mise en prod

- [ ] `npm run build` sans erreur ; `dist/` contient audio, images, traductions JSON **valides**.
- [ ] Parcours petit / moyen / grand : GPS, audio, quiz.
- [ ] Paiement test Stripe → email → activation lien **et** saisie manuelle du code.
- [ ] Bouton 🏠 (reset) → `parcours.html` sans redemander le code si accès valide.
- [ ] `mail.html` mentionne **Lille**.
- [ ] PWA : `manifest.json`, icônes `cityLoopQuest_Lille_*.png`.
- [ ] Popup fin de parcours + invitation selfie.
- [ ] CORS : appels API depuis https://clq-lille.netlify.app OK.

---

## 10. Dépannage fréquent

| Symptôme | Piste |
|----------|--------|
| `Failed to fetch` / API indisponible en local | Vider `localStorage.api_base` ou `useProdApi()` ; ne pas laisser `localhost:8081` si l’API locale n’est pas lancée. |
| Activation manuelle échoue, lien email OK | Même origine API ; vérifier `api-base.js` et CORS Render. |
| Service Worker / SSL en local | Normal sur `localhost:5173` en HTTPS ; SW désactivé en dev via `js/clq-sw-register.js`. |
| Licence d’une autre ville acceptée | `license-city-guard.js`, vider `localStorage` ou tester en navigation privée. |
| Maps ne s’affiche pas | `GOOGLE_MAPS_API_KEY` sur Netlify, référents HTTP, rebuild. |
| Webhook 400 | Secret webhook Stripe = celui configuré sur Render. |

---

## 11. Mise à jour de contenu

1. Modifier le JSON dans la **Factory**.
2. Régénérer l’app (les images/audio existants sont préservés si déjà sur disque).
3. `npm run build && npm run deploy` dans `CLQ-LILLE/clq-lille`.

---

*Document généré pour CityLoop Quest Lille — slug `lille`.*
