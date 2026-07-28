# GeTime — Plateforme SaaS Multi-Tenant de Gestion des Emplois du Temps Académiques

> **Statut du projet : en développement actif**
> Le code source de GeTime est hébergé sur un dépôt privé distinct. Ce dépôt public documente l'architecture, les choix techniques et l'état d'avancement du projet — l'accès au code source peut être demandé directement (voir section [Contact](#contact)).

---

## Sommaire

- [Vision du projet](#vision-du-projet)
- [Stack technique](#stack-technique)
- [Architecture](#architecture)
- [Moteur de planification et gestion des conflits](#moteur-de-planification-et-gestion-des-conflits)
- [Gestion multi-tenant](#gestion-multi-tenant)
- [Rôles, permissions et hiérarchie organisationnelle](#roles-permissions-et-hierarchie-organisationnelle)
- [Modules fonctionnels](#modules-fonctionnels)
- [État d'avancement](#etat-davancement)
- [Contact](#contact)

---

## Vision du projet

Les établissements d'enseignement supérieur africains font face à un problème récurrent : la planification manuelle des emplois du temps académiques génère des conflits (enseignants, salles, filières), des erreurs de communication et une perte de temps considérable pour les équipes administratives.

**GeTime** est une plateforme SaaS multi-tenant conçue pour automatiser cette planification, détecter les conflits en temps réel avant qu'ils ne se produisent, et fournir aux administrations académiques un outil moderne, fiable et adapté à la structure organisationnelle réelle des établissements (facultés, départements, filières, niveaux).

Ce projet est le projet phare de mon activité de développement de solutions SaaS pour le marché africain.

---

## Stack technique

| Couche | Technologie |
|---|---|
| Backend | Laravel 13 (PHP 8.3) |
| Frontend | React 19 + Vite + React Router (SPA) |
| Base de données | MySQL 8 |
| Files d'attente / jobs asynchrones | Redis |
| Autorisation | Spatie Laravel-Permission |
| Authentification API | Laravel Sanctum (tokens Bearer, stateless) |
| Styling | Tailwind CSS v4 (design tokens via `@theme`) |
| IA | Module RAG pour l'assistance à la génération de planning |

---

## Architecture

### Backend

L'architecture backend suit une approche en couches strictes, pensée pour la maintenabilité et la testabilité d'un domaine métier complexe :

```
Route → Middleware → Controller → Form Request → Service → Repository → Eloquent Model
```

- **Form Requests** : toute la validation des règles métier (quotas horaires, verrouillage de semaine, cohérence des créneaux) est isolée du contrôleur.
- **Services** : orchestrent la logique métier — détection de conflits, gestion des suppléments d'heures, escalade des rapports de cours.
- **Repositories** : encapsulent l'accès aux données derrière des interfaces, permettant de découpler la logique métier du moteur Eloquent.
- **Policies** (18 au total) : gèrent l'autorisation fine par ressource, en cohérence avec la hiérarchie organisationnelle à 8 niveaux.

### Frontend

Le frontend est une SPA React (Vite + React Router), volontairement **non** construite sur Next.js malgré une première recommandation en ce sens dans le cahier des charges initial — le choix a été révisé après analyse : GeTime est une application authentifiée, orientée dashboard, sans besoin de rendu serveur ni de SEO public, ce qui rend une SPA classique plus adaptée et plus simple à opérer.

Structure en couches simplifiée (volontairement allégée par rapport à une Clean Architecture complète, jugée sur-ingénierée pour la taille du projet) :

```
types → schemas (validation) → services (appels API) → hooks → components
```

- Authentification par token Bearer Sanctum, stocké en `sessionStorage`, injecté manuellement dans les en-têtes des requêtes (pas de cookies SPA, pas de sessions serveur).
- Thème sombre par défaut, piloté par variables CSS personnalisées.
- UI Kit interne complet : `Button`, `Card`, `Input`, `Badge`, `Spinner`, `Skeleton`, `Modal`, `Toast`, `Tooltip`, `Dropdown`, `Select` — construit avec les design tokens Tailwind v4 (`@theme`).

---

## Moteur de planification et gestion des conflits

Le cœur technique de GeTime est son **moteur anti-conflits**, qui exécute 8 vérifications distinctes avant de valider toute création ou modification de créneau (conflit enseignant, conflit salle, conflit filière, dépassement de quota horaire, etc.).

### Mécanisme de quota-supplément

Chaque enseignant possède un quota horaire par matière (et non un quota global), quel que soit son type de contrat (permanent, vacataire, visiteur). Lorsque ce quota est atteint :

- La planification sur cette matière est **bloquée** (pas seulement un avertissement).
- Le chef de département peut accorder des **suppléments d'heures**, de manière répétable, chaque octroi étant tracé individuellement avec justification obligatoire.
- Le plafond effectif est calculé dynamiquement : `heures_planifiées + somme(suppléments accordés)` — sans champ de statut "bloqué" stocké en base, pour éviter toute désynchronisation.

### Verrouillage hebdomadaire (Week-Lock)

Les semaines de programmation passent par des états contrôlés :

- **Verrouillée** : bloque la création/modification, autorise l'annulation avec motif obligatoire.
- **Clôturée** : bloque toute opération sauf une réouverture exceptionnelle, explicite et journalisée.

Le verrouillage est déclenché **automatiquement** par une tâche planifiée (configurable au niveau de l'établissement), et non par une action manuelle d'un planificateur — plusieurs planificateurs travaillant en parallèle sur la même semaine, un verrouillage manuel local aurait figé le travail des autres. C'est le moteur anti-conflits, et non le verrouillage, qui garantit la sécurité de la planification concurrente.

---

## Gestion multi-tenant

GeTime est conçu dès le départ comme une plateforme multi-tenant, avec une distinction stricte entre :

- **Platform Context** — vue globale du Super Admin sur l'ensemble des tenants.
- **Tenant Context** — le contexte d'un établissement donné.
- **Tenant Context Switcher** — mécanisme permettant au Super Admin de basculer entre tenants.

Le tenant actif est résolu **par requête**, via un middleware qui lit un identifiant `current_tenant_id` stocké directement sur le token d'accès (`personal_access_tokens`), plutôt qu'en session — cohérent avec une authentification API stateless. Le Global Scope Eloquent qui isole les données par tenant ne prévoit **aucun contournement automatique** pour le Super Admin : il se base uniquement sur le contexte actif résolu par le middleware, ce qui élimine toute ambiguïté de sécurité.

---

## Rôles, permissions et hiérarchie organisationnelle

- Hiérarchie organisationnelle à **8 niveaux**.
- **8 rôles système**, gérés via Spatie Laravel-Permission.
- **18 Policies Laravel** couvrant l'ensemble des ressources métier.
- Tableaux de bord différenciés par rôle.

La spécification complète des permissions est documentée séparément (`GeTime_Permissions_Spec_v1.md`), ainsi qu'une spécification métier pédagogique complète rédigée en français, extraite du domaine réel des établissements ciblés.

---

## Modules fonctionnels

| Module | Description | État |
|---|---|---|
| Programmation des emplois du temps | Création/modification de créneaux avec moteur anti-conflits (8 vérifications) | ✅ Backend complet et testé |
| Gestion des quotas et suppléments d'heures | Suivi par matière, octroi de suppléments tracés | ✅ Complet — 6 tests fonctionnels validés |
| Rapports de cours (signature délégué) | Statut à l'heure/en retard basé sur signature, escalade automatique J/J+1/J+2 | 🔄 Backend complet, frontend en cours |
| Verrouillage hebdomadaire | Automatisation du cycle verrouillée/clôturée | ✅ Complet |
| Contexte multi-tenant | Bascule Super Admin entre établissements | 🔄 En cours de conception/implémentation |
| Assistant IA (RAG) | Aide à la génération et à la suggestion de planning | 🔄 En cours d'exploration |
| Programmation des examens | — | ⏸ Hors périmètre pour cette version |

---

## État d'avancement

Ce projet est développé par phases (Phase 0 à Phase 6, MVP scope), avec une approche méthodique : chaque décision architecturale majeure (multi-tenant, quotas, verrouillage) fait l'objet d'une réflexion documentée avant implémentation, plutôt que d'un développement ad-hoc.

Le backend constitue aujourd'hui la partie la plus avancée du projet ; le frontend est en cours de construction en parallèle des modules backend restants.

---

## Contact

Le code source est hébergé sur un dépôt privé. Pour toute demande d'accès (revue technique, entretien, collaboration), vous pouvez me contacter directement.

**Negoue Tamo Sylvinhio**
Full Stack Software Engineer & Mobile Developer
📧 sylvinhio676@gmail.com
🔗 [github.com/sylvinhio676-ux](https://github.com/sylvinhio676-ux)
