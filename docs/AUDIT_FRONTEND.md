# Audit front-end — AcadémiePlus

**Date :** 31 juillet 2026
**Périmètre :** application web `src/` (Vite + React 18 + TypeScript + Tailwind + shadcn/ui + Supabase), configuration du build, hygiène du dépôt. Les Edge Functions Supabase (`supabase/functions/`) ne sont mentionnées que lorsqu'elles impactent le front.

---

## 1. Résumé exécutif

| Domaine | État | Commentaire |
|---|---|---|
| Architecture & routage | 🟢 Bon | Lazy loading systématique, routes protégées par rôle, ErrorBoundary |
| Sécurité front | 🟡 Correct | Sanitisation centralisée (DOMPurify), CSP en place ; dépendances vulnérables à corriger |
| Qualité de code / TypeScript | 🔴 Faible | `strict: false`, 761 `any` explicites, 6 erreurs `tsc` réelles, fichiers monolithiques |
| Tests | 🔴 Quasi inexistants | 1 seul test placeholder ; E2E non exécutable hors sandbox Lovable |
| Performance | 🟡 Moyen | Bon code-splitting par route, mais bundle d'entrée de 787 kB et chunk ParentDashboard de 620 kB |
| i18n / RTL | 🟡 Partiel | Infra i18next fr/ar solide, mais ~85 % des pages ont des textes en dur |
| Accessibilité | 🟡 Moyen | Bonnes bases (Radix, `alt`, `aria-*`), mais zoom bloqué dans le viewport |
| Hygiène du dépôt | 🔴 À corriger | APK de 9,3 Mo committé, lockfile pointant un registre privé, double dossier mobile |

Le build de production **fonctionne** (36 s) et la suite de tests **passe** (1 test). Les trois chantiers prioritaires sont : (1) la dette TypeScript/tests, (2) le poids du bundle initial, (3) l'hygiène du dépôt qui casse `npm install` hors de l'environnement Lovable.

---

## 2. Vérifications exécutées

| Commande | Résultat |
|---|---|
| `git fetch origin` | ✅ Connectivité au dépôt GitHub confirmée |
| `npm install` | ❌ Échec initial (403 sur registre privé Lovable — voir §8.1), ✅ après réécriture locale des URLs vers `registry.npmjs.org` |
| `npx tsc -b` | ❌ 6 erreurs de type réelles (voir §4.2) |
| `npm run lint` | ❌ 829 problèmes (782 erreurs, 47 avertissements) — détail §4.3 |
| `npm test` (Vitest) | ✅ 1/1 test (placeholder uniquement) |
| `npm run build` (Vite) | ✅ Build OK en 36 s, avec avertissement « chunks > 500 kB » |
| `npm audit` | ❌ 6 vulnérabilités (3 high, 3 moderate) |

---

## 3. Architecture — points forts

- **Code-splitting exemplaire** : 51 routes en `React.lazy()` dans `src/App.tsx`, seule la page d'accueil est chargée en eager, avec un `Suspense` fallback global.
- **Contrôle d'accès par rôle bien structuré** : `ProtectedRoute` (`src/components/ProtectedRoute.tsx`) gère 6 rôles, redirige chaque utilisateur non autorisé vers *son* espace (`ROLE_HOME`), et absorbe les déconnexions transitoires du SDK Supabase (fenêtre de grâce de 1,5 s).
- **Cache des rôles en mémoire** dans `AuthContext` (une seule requête par session au lieu de 4-7 par navigation) — optimisation documentée en commentaire.
- **Sanitisation HTML centralisée** : `src/lib/sanitizeHtml.ts` (DOMPurify) est utilisé par tous les rendus de contenu de leçon (`HtmlWithMath`, `LessonMarkdown`, `PDFContent`, `InlineLessonEditor`), complété par `rehype-sanitize` côté Markdown.
- **`OptimizedImage`** avec IntersectionObserver pour le lazy loading des images.
- **Aucun `console.log`** dans `src/` ; commentaires de code soignés qui expliquent le *pourquoi*.
- **PKCE** pour le flux OAuth (nécessaire au flux natif Android, documenté dans `src/integrations/supabase/client.ts`).

---

## 4. Qualité de code & TypeScript

### 4.1 Configuration TypeScript trop permissive — 🔴 priorité haute

`tsconfig.app.json` : `strict: false`, `noImplicitAny: false`, `strictNullChecks: false`, `noUnusedLocals/Parameters: false`. ESLint désactive aussi `no-unused-vars`. Conséquences mesurées :

- 334 occurrences de `as any` dans `src/`
- 761 erreurs `@typescript-eslint/no-explicit-any` au lint

Sans `strictNullChecks`, toute la catégorie « undefined is not a function / cannot read property of null » n'est détectée qu'à l'exécution. **Recommandation :** activer `strictNullChecks` d'abord (le moins coûteux/le plus rentable), puis `noImplicitAny`, en corrigeant module par module.

### 4.2 Erreurs de type réelles masquées par le build — 🔴

Vite/SWC ne fait pas de type-checking : le build passe alors que `npx tsc -b` échoue avec 6 erreurs, dont :

- `src/components/admin/AddUserDialog.tsx:79` et `EditUserDialog.tsx:103` — `.finally()` appelé sur un `PromiseLike` (thenable Supabase) qui ne l'expose pas.
- `src/components/course/QuizExerciseCRUD.tsx:554` — `ItemType` incluant `"chapter"` assigné à un type qui ne l'accepte pas.
- `src/components/course/RichContentField.tsx:43` — prop obligatoire `resetKey` manquante sur `InlineLessonEditor`.
- `src/components/admin/AdminAssistantPanel.tsx:51` et `src/lib/documentExtraction.ts:43` — import d'URL `https://esm.sh/mammoth@1.12.0` non résolu par TS (import dynamique runtime ; à typer ou à remplacer par la dépendance npm).

**Recommandation :** ajouter un script `typecheck` (`tsc -b --noEmit`) et l'exécuter en CI pour empêcher la régression.

### 4.3 Lint — ⚠️

Répartition des 829 problèmes : 761 `no-explicit-any`, **37 `react-hooks/exhaustive-deps`**, 10 `react-refresh/only-export-components`, divers. Les 37 avertissements de dépendances de hooks (dans `Cours.tsx`, `Dashboard.tsx`, `Account.tsx`, `LessonActivityTabs.tsx`, etc.) sont les plus risqués : ce sont des bugs potentiels de synchronisation d'état (données périmées après changement de paramètre de route, par ex.).

### 4.4 Fichiers monolithiques — ⚠️

8 fichiers dépassent 800 lignes : `LessonActivityTabs.tsx` (1 587), `Cours.tsx` (1 226), `AdminAssistantPanel.tsx` (1 170), `Auth.tsx` (1 169), `StudentDashboardContent.tsx` (1 082), `Admin.tsx` (1 076), `InlineLessonEditor.tsx` (978), `ExamList.tsx` (927). `Cours.tsx` cumule ~9 `useEffect` et l'état d'un chat IA, de quiz, d'exercices et de navigation. À découper en sous-composants/hooks dédiés.

### 4.5 Couche data — ℹ️

Pas de bibliothèque de cache serveur (TanStack Query absent) : 270 `useEffect` dans pages/composants font du fetching manuel. Conséquences : pas de cache entre navigations, requêtes dupliquées, gestion manuelle des races/annulations. Une migration progressive vers TanStack Query (déjà standard dans les templates Lovable) réduirait fortement le code et les bugs de synchro — et traiterait mécaniquement une partie des 37 `exhaustive-deps`.

---

## 5. Tests — 🔴 priorité haute

- **Unitaires :** un seul fichier, `src/test/example.test.ts`, qui vérifie `expect(true).toBe(true)`. L'infrastructure (Vitest + Testing Library + jsdom) est installée et fonctionnelle mais inutilisée.
- **E2E :** `playwright.config.ts` importe `lovable-agent-playwright-config`, paquet absent de `package.json` — les E2E ne sont pas exécutables hors de la sandbox Lovable, et aucun fichier `*.spec.ts` n'existe.

**Recommandation :** commencer par des tests sur la logique pure et critique déjà isolée : `src/lib/sanitizeHtml.ts` (sécurité), `lessonBlocks.ts`, `levelEngine.ts`, `validatePassword` (à extraire de `Auth.tsx`), puis `ProtectedRoute` (matrice rôles × routes) avec Testing Library.

---

## 6. Performance

### 6.1 Bundle d'entrée trop lourd — 🔴 priorité haute

Sortie du build :

| Chunk | Taille | Gzip |
|---|---|---|
| `index-C_7Zk_CC.js` (entrée) | 787 kB | 244 kB |
| `ParentDashboard` | 620 kB | 185 kB |
| `index-mkHTU-B-.js` | 429 kB | 130 kB |
| `generateCategoricalChart` (recharts) | 349 kB | 98 kB |
| `HtmlWithMath` (KaTeX) | 267 kB | 81 kB |

Causes identifiées et remèdes :

1. **`jspdf` + `html2canvas` importés statiquement** dans `src/lib/parentReportPdf.ts` (importé par `ParentDashboard.tsx`) → passer à `await import("jspdf")` dans le handler de téléchargement ; le chunk ParentDashboard devrait fondre de ~450 kB.
2. **`framer-motion` dans le chemin eager** (via `Hero.tsx`/`MobileHome.tsx` importés par `Index`) → contribue au chunk d'entrée pour de simples animations d'apparition ; remplaçable par des animations CSS sur la landing, ou différable.
3. **Pas de `manualChunks`** dans `vite.config.ts` → configurer un découpage vendor (react, radix, supabase) pour améliorer le cache navigateur entre déploiements.

### 6.2 Divers — ⚠️

- 47 `key={index}` dans des listes : acceptable pour des listes statiques, source de bugs pour des listes réordonnées/filtrées (quiz, messagerie) — à vérifier au cas par cas.
- Aucun `React.memo` dans le projet ; `useMemo`/`useCallback` présents (133) mais non systématiques sur les gros arbres (dashboards).
- Google Fonts (Cairo, Tajawal) chargées de façon bloquante dans `index.html` (les `preconnect` sont bien présents). Option : auto-héberger via `@fontsource` — supprimerait aussi la dépendance CSP à `fonts.googleapis.com`.
- `AuthContext` fait les redirections post-connexion en `window.location.href` (rechargement complet, ~25 occurrences dans `src/`). Si c'est voulu (remise à zéro de l'état), le documenter ; sinon préférer `navigate()`.

---

## 7. Sécurité front

### 7.1 Points positifs — ✅

- Tous les `dangerouslySetInnerHTML` de contenu passent par `sanitizeLessonHtml` (DOMPurify) ; `chart.tsx` (shadcn) n'injecte que du CSS généré ; aucun secret en dur dans `src/` (la clé Supabase committée est la clé *anon/publishable*, prévue pour être publique — la sécurité réelle repose sur les RLS).
- CSP définie à la fois en `<meta>` (`index.html`) et en en-têtes serveur (`public/_headers`), avec `connect-src` limité au projet Supabase, `object-src 'none'`, `frame-ancestors 'self'` — et le commentaire dans `index.html` documente correctement les limites des `<meta>`.
- HSTS, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy` dans `public/_headers` (⚠️ à condition que l'hébergeur les serve réellement — à vérifier en production avec `curl -I`).

### 7.2 À corriger / surveiller

- 🔴 **`npm audit` : 6 vulnérabilités (3 high, 3 moderate)** — `react-router` (via react-router-dom), `ws` (uninitialized memory disclosure + DoS), `form-data` (CRLF injection), `js-yaml`, `brace-expansion`, `esbuild` (dev). La plupart se corrigent avec `npm audit fix` ; planifier la montée de `react-router-dom`.
- ⚠️ **`script-src 'unsafe-eval'`** dans la CSP : affaiblit la protection XSS. Identifier la bibliothèque qui l'exige et vérifier si elle est toujours nécessaire.
- ⚠️ **Session Supabase en `localStorage`** : choix par défaut du SDK, mais tout XSS résiduel donne accès aux tokens. C'est précisément pour cela que la CSP et la sanitisation doivent rester strictes (tests sur `sanitizeLessonHtml` recommandés — §5).
- ⚠️ **`Excellence.tsx:47`** injecte une chaîne de traduction en HTML sans sanitisation (`dangerouslySetInnerHTML={{ __html: t("excellence.intro1") }}`). Risque faible (fichiers de traduction contrôlés) mais c'est le seul point d'injection non sanitisé — utiliser le composant `<Trans>` de react-i18next.
- ℹ️ Le fichier `.env` est committé ; il ne contient que des valeurs publiables, mais l'usage recommandé est `.env.example` + `.env` ignoré, pour éviter qu'un vrai secret n'y atterrisse un jour par habitude.

---

## 8. Hygiène du dépôt

### 8.1 Lockfile lié au registre privé Lovable — 🔴 (bloquant pour la CI)

`package-lock.json` contient 15 URLs `resolved` pointant vers `europe-west1-npm.pkg.dev/lovable-core-prod/sandbox-npm-cache/...`. Hors de la sandbox Lovable, ce registre renvoie **403** et **`npm install` échoue** (reproduit dans cet audit ; contournement local : réécriture des URLs vers `registry.npmjs.org`, non committée). **Recommandation :** régénérer le lockfile contre le registre public (`rm -rf node_modules && npm install --registry=https://registry.npmjs.org`) et le committer, sinon aucune CI externe ne pourra builder le projet.

### 8.2 Artefacts binaires committés — 🔴

- `application-mobile/APKs/academieplus-debug.apk` : **9,3 Mo** (86 % du poids du dépôt, 10,85 Mo au total). Un APK — de surcroît *debug* — n'a pas sa place dans git : à distribuer via GitHub Releases et à retirer de l'historique (`git filter-repo` ou BFG).
- `overleaf/main.pdf` (160 kB) : artefact généré depuis `main.tex`, à ignorer.

### 8.3 Divers — ⚠️

- **Deux dossiers mobiles quasi homonymes** : `application-mobile/` (APKs) et `application_mobile/` (config Capacitor + README), plus un `capacitor.config.ts` à la racine ET dans `application_mobile/`. À fusionner pour lever l'ambiguïté (les scripts npm `cap:*` pointent vers `application_mobile`).
- **Deux lockfiles** : `bun.lock` et `package-lock.json` coexistent. Choisir un gestionnaire et supprimer l'autre lockfile.
- **Bug fonctionnel** : `src/components/ReferralShareDialog.tsx:71` — le partage Facebook utilise `app_id=YOUR_APP_ID` (placeholder) : la fonctionnalité est cassée en production.

---

## 9. i18n & RTL — 🟡

L'infrastructure est bonne : i18next avec détection de langue, `applyDirection()` qui bascule `dir`/`lang` sur `<html>`, arabe par défaut avec fallback `ar`, fichiers `fr.json`/`ar.json`.

Mais la couverture est très partielle :

- Seules **7 pages sur 47** utilisent `useTranslation` (essentiellement le marketing : Hero, Pricing, FAQ…).
- **154 fichiers** de `src/` contiennent du texte arabe en dur, et d'autres du français en dur (`ProtectedRoute` : « Vérification des autorisations… », `Auth.tsx` : messages de validation). Le sélecteur de langue ne s'applique donc réellement qu'à la vitrine ; l'application interne est figée dans un mélange arabe/français.
- `index.html` : `lang="ar" dir="rtl"` statique alors que `<title>` et `meta description` sont en français — incohérent pour le SEO et les lecteurs d'écran avant hydratation.

**Recommandation :** décider du périmètre réel du bilinguisme. Si l'app interne doit être bilingue, migrer progressivement les chaînes vers les fichiers de locale (commencer par les composants partagés : `ProtectedRoute`, `AppHeader`, toasts). Sinon, retirer le sélecteur de langue des écrans internes pour ne pas promettre ce qui n'existe pas.

---

## 10. Accessibilité — 🟡

- ✅ Bonnes bases : primitives Radix (focus management, rôles), 74 attributs `aria-*`, attributs `alt` présents sur les `<img>` vérifiées (y compris `alt_text` dynamique côté éditorial).
- 🔴 **`user-scalable=no, maximum-scale=1.0`** dans le viewport (`index.html:5`) : bloque le zoom, violation directe de WCAG 1.4.4 (Resize Text) et pénalisant pour un public scolaire. Si c'était pour éviter le zoom auto sur focus des inputs iOS, `font-size >= 16px` sur les champs suffit.
- ⚠️ Pas de tests a11y automatisés (`axe-core`/`jest-axe` ou le plugin ESLint `jsx-a11y`) — le plugin ESLint serait le gain le plus rapide.
- ⚠️ Points non vérifiables statiquement (contrastes réels, ordre de focus, annonces de toasts) : à passer au travers d'un audit Lighthouse/axe sur l'app déployée.

---

## 11. Plan d'action priorisé

| # | Action | Effort | Impact |
|---|---|---|---|
| 1 | Régénérer `package-lock.json` sur le registre public (§8.1) | XS | Débloque toute CI/CD externe |
| 2 | `npm audit fix` + montée `react-router-dom` (§7.2) | S | Corrige 3 vulnérabilités high |
| 3 | Import dynamique de `jspdf`/`html2canvas` (§6.1) | XS | −450 kB sur le chunk ParentDashboard |
| 4 | Retirer l'APK et le PDF du dépôt (§8.2) | S | −9,5 Mo, dépôt sain |
| 5 | Corriger les 6 erreurs `tsc` + script `typecheck` en CI (§4.2) | S | Stoppe la régression de types |
| 6 | Supprimer `user-scalable=no` (§10) | XS | Conformité WCAG 1.4.4 |
| 7 | Corriger le placeholder `YOUR_APP_ID` Facebook (§8.3) | XS | Répare le partage parrainage |
| 8 | Traiter les 37 `react-hooks/exhaustive-deps` (§4.3) | M | Élimine des bugs de synchro latents |
| 9 | `manualChunks` + audit `framer-motion` sur l'entrée (§6.1) | M | Réduction du bundle initial |
| 10 | Tests unitaires sur `sanitizeHtml`, `levelEngine`, `ProtectedRoute` (§5) | M | Filet de sécurité minimal |
| 11 | Activer `strictNullChecks` puis `noImplicitAny`, module par module (§4.1) | L | Fiabilité de fond |
| 12 | Étendre l'i18n aux écrans internes ou réduire le périmètre affiché (§9) | L | Cohérence produit |

---

*Audit réalisé sur le commit `d127d87` (branche `claude/academieplus-frontend-audit-7nbgx0`).*
