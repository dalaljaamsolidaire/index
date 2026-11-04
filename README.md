# DALAL JAAM SOLIDAIRE — Plateforme web

Site institutionnel, collecte de dons (Wave, Orange Money) et back-office de l'association.

**Stack** : Next.js 15 (App Router, Server Actions) · React 19 · Prisma 6 · SQLite (dév.) / PostgreSQL (prod.) · Tailwind CSS 4 · Manrope.

---

## 1. Démarrage

```bash
npm install
cp .env.example .env        # puis compléter les valeurs
npx prisma db push           # crée la base
npm run db:seed              # contenus provisoires + compte super administrateur
npm run dev                  # http://localhost:3000
```

- Back-office : `http://localhost:3000/admin` (identifiants `SEED_ADMIN_EMAIL` / `SEED_ADMIN_PASSWORD` du `.env`).
  **Changez le mot de passe dès la première connexion** (menu *Mon compte*).
- Production : `npm run build && npm start`.

## 2. Contenus provisoires — à remplacer

Conformément au cahier des charges, **aucune statistique, aucun partenaire, membre d'équipe, réalisation, témoignage ou information légale n'a été inventé**.

| Élément | État livré | Où le compléter |
|---|---|---|
| Présentation, histoire, vision, mission, valeurs | Textes provisoires (badge « Contenu provisoire ») | *Textes des pages* |
| Programmes (4 axes) | Axes génériques provisoires | *Programmes* |
| Équipe | 4 fonctions, noms « à compléter » | *Équipe* |
| Partenaires, réalisations, témoignages | Vides (état d'attente élégant) | menus correspondants |
| Mentions légales, NINEA, récépissé, siège | « à compléter » | *Textes des pages* + *Paramètres* |
| Photos (bannière, présentation), logo | Emplacements identifiés « Photo officielle à venir » ; monogramme provisoire | *Médiathèque* puis *Paramètres* |
| Coordonnées, WhatsApp, carte | Vides | *Paramètres* |

Le badge provisoire disparaît dès que la case « Contenu provisoire » est décochée.

## 3. Paiements — Wave et Orange Money

**Principe non négociable : un don ne passe à « Confirmé » (et n'obtient un reçu) que via `applyVerification()`, appelée uniquement après une vérification auprès du prestataire.** La redirection de retour du navigateur n'est jamais considérée comme une preuve de paiement. Sans clés configurées, le formulaire affiche « Paiement en ligne bientôt disponible » et aucune transaction n'est créée.

### Wave Business (Checkout API)
1. Dans le portail Wave Business : créer une clé API *Checkout* et un webhook.
2. URL du webhook : `https://<domaine>/api/payments/wave/webhook` — événements `checkout.session.completed` et `checkout.session.payment_failed`.
3. Renseigner `WAVE_API_KEY` et `WAVE_WEBHOOK_SECRET`.

Sécurité : signature HMAC `Wave-Signature` vérifiée (anti-rejeu 5 min), puis **relecture de la session via `GET /v1/checkout/sessions/:id`**, contrôle du montant, traitement idempotent.

### Orange Money Sénégal (API Sonatel)
1. Obtenir auprès de Sonatel : `client_id`, `client_secret`, code marchand.
2. Déclarer l'URL de callback marchand `https://<domaine>/api/payments/orange/webhook` avec `ORANGE_WEBHOOK_SECRET` comme clé d'API (en-tête `x-api-key`).
3. Renseigner les variables `ORANGE_*`.

> ⚠️ **À valider en recette avec Sonatel avant mise en ligne.** Les chemins d'API (`ORANGE_TOKEN_PATH`, `ORANGE_QRCODE_PATH`, `ORANGE_STATUS_PATH`) reprennent la documentation Sonatel connue et sont surchargeables par variable d'environnement. Par défaut (`ORANGE_VERIFY_MODE=api`), chaque notification est re-vérifiée via l'endpoint de statut ; `callback` ne doit être utilisé que si votre contrat ne fournit pas cet endpoint.

### Dons mensuels
Wave et Orange Money ne proposent pas de prélèvement automatique pour ce type d'usage. Un don mensuel crée donc un **engagement** : chaque mois, le donateur reçoit un lien personnel (`/engagement/<jeton>`) et valide lui-même le paiement. Le back-office (*Engagements mensuels*) liste les échéances à relancer avec des boutons WhatsApp / e-mail pré-remplis.

### Tâche planifiée (toutes les heures)
```bash
curl -H "Authorization: Bearer $CRON_SECRET" https://<domaine>/api/cron/recurring
# ou : npm run recurring:reminders
```
Re-vérifie les dons en attente, passe en *Expiré* ceux non confirmés après 48 h (jamais en *Confirmé*), marque les engagements à relancer.

## 4. Fonctionnalités

**Public** : Accueil · À propos · Nos actions (+ fiches) · Réalisations (galerie photo/vidéo, lightbox, filtres) · Actualités (pagination, partage) · Faire un don · Suivi du don · Reçu imprimable / PDF · Espace donateur (historique, totaux, reçus, engagements) · Devenir bénévole · Partenaires · Contact (formulaire, téléphone, WhatsApp, carte) · Mentions légales · Données personnelles.

**Back-office** : tableau de bord (collecte confirmée, 30 jours, répartition), dons (filtres, détail, journal prestataire, revérification, export CSV), engagements, campagnes (objectif + jauge basée sur les dons confirmés), textes des pages, actualités (brouillon / publication / programmation), programmes, réalisations (+ galerie), témoignages, partenaires, équipe, médiathèque, bénévoles (suivi, notes, suppression), messages, paramètres, utilisateurs & rôles, journal d'activité.

**Rôles**

| Rôle | Accès |
|---|---|
| Super administrateur | Tout, dont comptes utilisateurs |
| Administrateur | Tout sauf comptes |
| Éditeur de contenu | Contenus, textes, médiathèque |
| Trésorerie / Finances | Dons, engagements, campagnes, exports |

## 5. Sécurité et données personnelles

- Mots de passe bcrypt (coût 12), session JWT signée en cookie `httpOnly`/`SameSite=Lax`/`Secure` (prod.), 8 h.
- Contrôle d'accès côté serveur sur chaque page **et** chaque action (`requireUser(permission)`), middleware en première barrière.
- Limitation de débit (connexion, formulaires, dons, statut), piège à robots, validation Zod, protection CSRF native des Server Actions.
- Téléversements : liste blanche de types, vérification de la signature binaire, 8 Mo max., noms aléatoires.
- En-têtes de sécurité (HSTS, nosniff, frame-options, referrer-policy, permissions-policy), `noindex` sur l'admin et les pages privées.
- Export CSV protégé contre l'injection de formules ; journal d'audit des actions sensibles.
- Consentement explicite sur tous les formulaires ; page de politique de confidentialité (loi sénégalaise n° 2008-12, à finaliser).
- Le limiteur de débit est en mémoire : pour plusieurs instances, le remplacer par Redis.

## 6. Sauvegardes

```bash
npm run backup
```
Crée `backups/<horodatage>/` (base SQLite via `VACUUM INTO`, export JSON des dons, fichiers téléversés) et conserve les 30 dernières. À programmer quotidiennement et à copier hors du serveur.
En PostgreSQL : `pg_dump "$DATABASE_URL" > backups/db-$(date +%F).sql` en complément.

## 7. Mise en production (recommandations)

1. PostgreSQL : `provider = "postgresql"` dans `prisma/schema.prisma`, `DATABASE_URL` adaptée, `npx prisma migrate deploy` (ou `db push`).
2. `APP_URL` = URL publique en HTTPS (utilisée pour les retours de paiement, reçus, sitemap).
3. Stockage des fichiers : `public/uploads` doit être persistant (volume) — ou adapter `src/lib/uploads.ts` vers un stockage objet.
4. Configurer la tâche planifiée et la sauvegarde quotidienne.
5. Tester un paiement réel de faible montant avec chaque prestataire avant l'ouverture publique.

## 8. Structure

```
prisma/            schéma + données initiales
scripts/           sauvegarde, tâche planifiée
src/app/(site)/    pages publiques
src/app/admin/     back-office
src/app/api/       webhooks paiement, statut, cron, export
src/lib/payments/  intégrations Wave et Orange Money
src/lib/           session, permissions, contenus, utilitaires
```
