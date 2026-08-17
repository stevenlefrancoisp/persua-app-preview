# Plan du build réel : parcours Persua sur auth + Supabase

## ⚠️ MISE À JOUR (correction majeure après recon de l'espace membre)

Le backend Supabase **existe déjà** dans `~/Desktop/persua-espace-membre` et couvre ~90% du plan. On ne crée donc PAS un nouveau Supabase (ce serait la fragmentation à éviter). On **branche la preview sur l'existant** et on ajoute le peu qui manque.

Déjà en place (tables `supabase/schema.sql`) :
- **Auth** (auth.users) + **`profiles`** (first_name, email, `role` élève/coach, `streak`, `positioning` jsonb = le diagnostic, `positioning_submitted_at`).
- **`progress`** (user_id, kind `exercise`/`quiz`, ref) = progression par user, déjà là.
- **`checkins`**, **`journal_entries`**, **`member_documents`**, **`coach_analyses`** + fonction `public.is_coach()` pour la vue coach (RLS).

Ce qui manque (delta) → migration écrite : `supabase/bilans_migration.sql`
- **`bilans`** (série hebdo des 4 axes) = le dashboard/la notation dans le temps. Ajoutée + RLS `owner or is_coach()`.
- **Communauté** (`posts`/`post_claps`) : optionnelle, commentée dans la migration.

Map preview → existant :
| Preview | Backend espace membre |
|---|---|
| Diagnostic 33 questions | `profiles.positioning` (jsonb) + `positioning_submitted_at` |
| Vidéos vues / exercices / quiz | `progress` (kind exercise/quiz, ref) |
| Série de jours | `profiles.streak` |
| Bilan hebdo 4 axes + dashboard | **`bilans`** (nouveau) |
| Communauté | `posts` (nouveau, optionnel) |
| Vue coach | `role='coach'` + `is_coach()` (déjà là) |

Décisions à trancher (voir fin de session) : (1) on **fond la preview dans l'app React de l'espace membre** (recommandé, une base une identité) ; (2) il me faut le vrai `VITE_SUPABASE_URL` + `VITE_SUPABASE_ANON_KEY` (ou tu lances le SQL toi-même) ; (3) la nouvelle structure de formation de la preview **remplace** l'actuelle ou on **ajoute juste** bilan hebdo + dashboard.

---

Objectif : passer de la preview (localStorage, par appareil) à un vrai produit où la data d'un élève le suit sur tous ses appareils, chaque élève isolé, une seule base. Le design est déjà spécifié par la preview, on ne réinvente pas l'UX, on branche l'identité et la persistance.

## Principe d'architecture
- **Une seule base Supabase, une seule identité par personne.** Marc se connecte sur son tel, son ordi, sa tablette : même compte, mêmes données, lues dans Supabase par `user_id`. Luc a sa propre ligne. Isolation garantie par RLS (row level security). C'est l'inverse de la fragmentation de Louis.
- **Le frontend reste la preview qu'on a construite**, on lui greffe le client Supabase + une porte d'auth. Pas de réécriture React pour la v1 (on réutilise tout le travail). On pourra fondre dans l'espace membre React plus tard si on veut un seul codebase.

## Décisions à trancher (bloquantes)
1. **Projet Supabase** : réutiliser celui de l'espace membre (recommandé, une seule identité Persua partout) ou un projet dédié au parcours ? À vérifier : le ref du projet espace membre et si son auth sert déjà les membres.
2. **Auth** : email + mot de passe (choisi : RGPD + chacun a SON espace personnel). Lien magique écarté.
3. **Frontend** : garder l'app single-file actuelle + Supabase (rapide, recommandé v1) ou fondre dans l'app React de l'espace membre (plus propre, plus long) ?
4. **Domaine** : sous-domaine dédié (ex. `parcours.persua-neurovente.com`) ou intégré à `app.persua-neurovente.com` ?

## Schéma de base (SQL)

```sql
-- Identité (1:1 avec auth.users)
create table profiles (
  id uuid primary key references auth.users on delete cascade,
  email text, first_name text,
  role text default 'student',        -- 'student' | 'coach'
  created_at timestamptz default now()
);

-- Progression contenu (vidéos vues, fiches lues)
create table content_progress (
  user_id uuid references auth.users on delete cascade,
  content_id text,                     -- 'm1v1', 'm1f1', ...
  updated_at timestamptz default now(),
  primary key (user_id, content_id)
);

-- Validation des quiz de module
create table module_progress (
  user_id uuid references auth.users on delete cascade,
  module_id text,                      -- 'm1'..'m7'
  quiz_passed boolean default false,
  quiz_best int,
  updated_at timestamptz default now(),
  primary key (user_id, module_id)
);

-- Bilans des 3 semaines (série temporelle = coeur du dashboard)
create table bilans (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users on delete cascade,
  taken_at timestamptz default now(),
  decouverte int, objections int, closing int, calme int
);

-- Gestes quotidiens
create table gestes_log (
  user_id uuid references auth.users on delete cascade,
  geste_id text, done_date date default current_date,
  primary key (user_id, geste_id, done_date)
);

-- Communauté
create table posts (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users on delete cascade,
  body text not null, tag text, created_at timestamptz default now()
);
create table post_claps (
  post_id uuid references posts on delete cascade,
  user_id uuid references auth.users on delete cascade,
  primary key (post_id, user_id)
);
```

## Isolation (RLS) : la brique qui garantit "Marc voit sa data, pas celle de Luc"

```sql
-- Chaque table user : on ne voit et n'écrit que ses propres lignes
alter table content_progress enable row level security;
create policy own on content_progress using (user_id = auth.uid()) with check (user_id = auth.uid());
-- idem module_progress, bilans, gestes_log

-- Communauté : lecture par tous les connectés, écriture = les siennes
alter table posts enable row level security;
create policy read_all  on posts for select to authenticated using (true);
create policy write_own on posts for insert to authenticated with check (user_id = auth.uid());
create policy edit_own  on posts for update using (user_id = auth.uid());

-- Vue coach (Steven voit l'évolution de tous ses élèves)
create policy coach_reads_all on bilans for select
  using (exists (select 1 from profiles p where p.id = auth.uid() and p.role = 'coach'));
```

Création auto du profil à l'inscription :
```sql
create function handle_new_user() returns trigger language plpgsql security definer as $$
begin insert into profiles (id, email) values (new.id, new.email); return new; end $$;
create trigger on_auth_user_created after insert on auth.users
  for each row execute function handle_new_user();
```

## Flux auth (email + mot de passe, RGPD)
1. Inscription : email + mot de passe → `supabase.auth.signUp({ email, password })`, confirmation par email.
2. Connexion : `supabase.auth.signInWithPassword({ email, password })`. Session persistée sur chaque appareil (Marc reconnecté partout, c'est SON espace).
3. Premier login : trigger crée `profiles`, on demande le prénom une fois. Mot de passe oublié : `resetPasswordForEmail`.
4. RGPD : case de consentement à l'inscription, lien vers la politique de confidentialité, droit à l'export et à la suppression du compte (`on delete cascade` déjà en place efface toute la data de l'utilisateur).

## Couche de données (remplace ST + localStorage)
- Au login : charger en mémoire la progression + les bilans de l'utilisateur (2-3 requêtes).
- À chaque action (vidéo vue, quiz validé, bilan soumis, post) : write-through Supabase (`upsert`).
- localStorage devient un simple cache offline (PWA), la vérité est dans Supabase.
- Dashboard : `select * from bilans where user_id = auth.uid() order by taken_at` → les 4 axes + l'évolution, exactement comme la preview mais réel et multi-appareils.

## Migration localStorage -> Supabase (une fois)
Au premier login, si le localStorage local contient une progression et que la ligne Supabase est vide : proposer d'importer (upsert). Sinon Supabase prime. Évite de perdre ce que l'utilisateur a fait en preview.

## Bonus offert par cette archi (impossible en localStorage)
- **Vue coach pour toi** : une requête te donne l'évolution des bilans de toute ta communauté, qui décroche, qui progresse. Le vrai avantage sur le setup fragmenté de Louis.
- **Contenu éditable sans redeploy** (v2) : mettre modules/vidéos/quiz dans des tables `modules`/`contents`/`quiz_questions`. Corrige aussi le trou des URLs Skool du M6 (édition en base). En v1 on garde le contenu statique dans l'app (rapide).

## Phasage
- **P0** Décisions ci-dessus + vérif du projet Supabase espace membre.
- **P1** Schéma + RLS + auth lien magique + trigger profil. (fondations)
- **P2** Greffe du client Supabase dans l'app + porte d'auth (écran login).
- **P3** Remplacement de la persistance ST par Supabase (progression, quiz) + migration localStorage.
- **P4** Bilans + dashboard depuis la table `bilans` ; gestes ; communauté (option Realtime pour le live).
- **P5** Vue coach + finitions PWA (offline, cache versionné) + deploy sur le sous-domaine.
- **P6** Import localStorage, invitation de la communauté.

## Standard de build
Tout le front passe la checklist de `persua-frontend-uxui` (mobile safe, a11y, perf, PWA). L'écrit d'interface passe l'anti-slop de `persua-brand-content`. Clé anon Supabase publique côté client (RLS protège), jamais la service key dans le front.
```
