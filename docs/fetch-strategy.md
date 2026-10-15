# Stratégie de fetch et de cascade — AutoLoc

> Atelier 2 — Associations JPA : Cascade et Fetch
> Auteur : Founoun Dhhibi

Ce document justifie, pour chaque association du modèle AutoLoc, le choix du côté propriétaire,
du `FetchType`, du `CascadeType` et de `orphanRemoval`.

## 1. Principes retenus

- **Côté propriétaire** : c'est le côté qui porte la clé étrangère en base, toujours le côté
  `@ManyToOne` d'une relation un-à-plusieurs. Le côté inverse (`@OneToMany`) utilise
  `mappedBy` et ne crée aucune colonne.
- **Fetch** : `LAZY` partout. Les valeurs par défaut de `@ManyToOne` et `@OneToOne` sont `EAGER` ;
  les forcer à `LAZY` évite de charger toute une chaîne d'objets liés (réservation → client →
  véhicule → agence…) à chaque lecture. Les collections (`@OneToMany`, `@ManyToMany`) sont déjà
  `LAZY` par défaut, on le déclare explicitement pour que le choix soit visible.
- **Cascade** : uniquement quand l'enfant **ne peut pas vivre sans son parent** (composition).
  Par défaut, aucune opération n'est cascadée.
- **Lombok** : `@Getter` / `@Setter` ciblés, jamais `@Data`, car son `toString()` / `hashCode()`
  parcourrait les deux côtés d'une association bidirectionnelle (`StackOverflowError`).

## 2. Tableau récapitulatif

| Association | Annotations | Côté propriétaire (clé étrangère) | Fetch | Cascade | Justification |
|---|---|---|---|---|---|
| Contrat → Paiement | `@OneToMany(mappedBy)` / `@ManyToOne` | `Paiement` (`contrat_id`) | LAZY | `ALL` + `orphanRemoval = true` | Un paiement n'existe que rattaché à son contrat (composition) : supprimer le contrat supprime ses paiements, et retirer un paiement de la liste le supprime en base. LAZY : charger un contrat ne doit pas charger tout son historique de paiements. |
| Agence → Véhicule | `@OneToMany(mappedBy)` / `@ManyToOne` | `Vehicule` (`agence_id`) | LAZY | Aucune | Un véhicule survit à la suppression de son agence (il peut être réaffecté) : supprimer l'agence ne doit pas détruire le parc. LAZY : une agence peut avoir des centaines de véhicules. |
| Agence → Employé | `@OneToMany(mappedBy)` / `@ManyToOne` | `Employe` (`agence_id`) | LAZY | Aucune | Un employé existe indépendamment de son agence (mutation possible). Pas de cascade pour ne pas perdre de personnel par erreur. |
| Véhicule ↔ Équipement | `@ManyToMany` + `@JoinTable` | `Vehicule` (table `vehicule_equipement`) | LAZY | Aucune | Les équipements (GPS, siège bébé…) sont **partagés** entre plusieurs véhicules : supprimer un véhicule ne doit pas supprimer un équipement utilisé ailleurs. `Set` pour éviter les doublons. |
| Client → Réservation | `@OneToMany(mappedBy)` / `@ManyToOne` | `Reservation` (`client_id`) | LAZY | `PERSIST` | Enregistrer un nouveau client avec ses réservations les enregistre aussi. Pas de `REMOVE` : l'historique des réservations ne doit pas disparaître avec le compte client. |
| Réservation → Véhicule | `@ManyToOne` (unidirectionnelle) | `Reservation` (`vehicule_id`) | LAZY | Aucune | Une réservation référence un véhicule qui existe indépendamment d'elle. Pas besoin de naviguer du véhicule vers ses réservations, donc pas de côté inverse. |
| Réservation ↔ Contrat | `@OneToOne(mappedBy)` / `@OneToOne` | `Contrat` (`reservation_id`, unique) | LAZY | `ALL` (côté `Reservation`) | Un contrat n'a pas de sens sans sa réservation : enregistrer ou supprimer la réservation se propage au contrat. |
| Véhicule → Maintenance | `@OneToMany(mappedBy)` / `@ManyToOne` | `Maintenance` (`vehicule_id`) | LAZY | `PERSIST` | Une maintenance est créée avec le véhicule concerné. Pas de `REMOVE` : l'historique de maintenance doit être conservé même si le véhicule est retiré. |

## 3. Points d'attention

- **`@OneToOne` et LAZY** : côté propriétaire (`Contrat.reservation`), `LAZY` fonctionne bien.
  Côté inverse (`Reservation.contrat`, avec `mappedBy`), Hibernate doit interroger la table
  `contrat` pour savoir si un contrat existe, ce qui rend le chargement paresseux inefficace sans
  instrumentation du bytecode. On le déclare tout de même `LAZY` pour rester cohérent.
- **`LazyInitializationException`** : accéder à une collection `LAZY` en dehors d'une transaction
  lève cette exception. On la traite avec une requête dédiée (`JOIN FETCH`) plutôt qu'en passant
  la collection en `EAGER`.
- **Synchroniser les deux côtés** : pour une association bidirectionnelle, positionner le côté
  propriétaire (`paiement.setContrat(contrat)`) **et** ajouter à la collection
  (`contrat.getPaiements().add(paiement)`). La clé étrangère est écrite à partir du côté propriétaire.
- **`cascade` vs `orphanRemoval`** : `REMOVE` supprime les enfants quand on supprime le parent ;
  `orphanRemoval` supprime un enfant retiré de la collection du parent, même si le parent
  n'est pas supprimé. Les deux sont complémentaires (utilisés ensemble sur Contrat → Paiement).

## 4. Vérification

Le schéma généré par Hibernate (`ddl-auto=update`) a été vérifié sur une base vierge :

- une clé étrangère par association propriétaire : `paiement.contrat_id`, `vehicule.agence_id`,
  `employe.agence_id`, `reservation.client_id`, `reservation.vehicule_id`, `contrat.reservation_id`
  (unique), `maintenance.vehicule_id` ;
- une seule table de jointure, `vehicule_equipement` ;
- aucune table ni colonne parasite sur les côtés inverses (`mappedBy`).

Comportement observé (logs `show-sql`) sur Contrat → Paiement :

- `persist(contrat)` seul insère aussi les paiements (cascade `PERSIST`) ;
- charger un contrat n'exécute que `select … from contrat` ; les paiements ne sont chargés
  qu'au premier appel de `getPaiements()` (`LAZY`) ;
- retirer un paiement de la liste exécute `delete from paiement` (`orphanRemoval`) ;
- supprimer le contrat supprime ses paiements restants (cascade `REMOVE`).
