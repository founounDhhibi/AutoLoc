# AutoLoc — Plateforme de gestion de location de véhicules multi-agences

> UP ASI — Architecture des Systèmes d'Information · Étude de cas « AutoLoc » · Atelier 0

## 1. Objectifs du projet

**AutoLoc** est une plateforme de gestion de location de véhicules destinée à un réseau
**multi-agences**. Elle centralise la gestion du parc de véhicules, des réservations, des
contrats de location et des agences, avec un contrôle d'accès différencié selon le rôle
de l'utilisateur.

## 2. Stack technique

| Catégorie | Outils / Bibliothèques |
|---|---|
| Langage / Build | Java 17, Maven |
| Framework | Spring Boot, Spring Data JPA, Spring MVC, Spring AOP |
| Base de données | MySQL 8.x (dev), H2 (tests) |
| Productivité | Lombok, SLF4J/Logback, MapStruct |
| Documentation API | springdoc-openapi (Swagger UI) |
| Tests | JUnit 5, Mockito, MockMvc, Jacoco |
| Outillage | Git/GitHub, Postman, IntelliJ IDEA Ultimate |

## 3. Acteurs identifiés (Séance 1)

| Acteur | Rôle |
|---|---|
| **Client** | Consulte les véhicules disponibles, réserve et gère son compte. |
| **Agent d'agence** | Gère les locations au comptoir : départs, retours, paiements. |
| **Responsable d'agence** | Gère le parc, le personnel et les tarifs de son agence. |
| **Administrateur** | Administre l'ensemble du réseau : agences, utilisateurs, configuration. |

## 4. Cas d'utilisation par acteur

### Client
- S'inscrire et s'authentifier.
- Rechercher un véhicule (agence, catégorie, dates).
- Consulter la disponibilité et les tarifs.
- Effectuer une réservation.
- Annuler une réservation.
- Consulter l'historique de ses locations.

### Agent d'agence
- S'authentifier.
- Consulter et traiter les réservations de son agence.
- Enregistrer un départ de véhicule (création du contrat de location).
- Enregistrer un retour de véhicule (clôture du contrat).
- Encaisser un paiement.

### Responsable d'agence
- Gérer le parc de véhicules de l'agence (ajout, retrait, mise en maintenance).
- Gérer les agents de son agence.
- Définir les tarifs de location.
- Consulter les statistiques et rapports de l'agence.

### Administrateur
- Gérer les agences (création, modification, suppression).
- Gérer les comptes utilisateurs et l'attribution des rôles.
- Superviser l'ensemble du réseau multi-agences.
- Configurer les paramètres globaux de la plateforme.

## 5. Base de données

```sql
CREATE DATABASE autoloc_db CHARACTER SET utf8mb4;
```

---
*Auteur : Founoun Dhhibi — founoun.dhhibi@esprit.tn*
