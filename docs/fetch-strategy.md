# Stratégies Fetch & Cascade — Modèle AutoLoc

> **Atelier 2 — Associations JPA : Cascade et Fetch**  
> Projet : `wassimouni4cce10` | Spring Boot + Spring Data JPA + Hibernate

---

## Principes généraux appliqués

- **Fetch LAZY par défaut sur toutes les associations** : évite de charger des graphes d'objets inutiles à chaque requête.
- **@ManyToOne et @OneToOne** ont un fetch EAGER par défaut — ils sont tous explicitement forcés à LAZY dans ce projet.
- **Cascade justifiée uniquement par composition** : une cascade de suppression n'est ajoutée que si l'entité enfant ne peut pas exister sans son parent.
- **@Data interdit sur les entités avec associations** : risque de `StackOverflowError` via `toString()`/`hashCode()` cycliques. On utilise `@Getter`/`@Setter` ciblés.

---

## Tableau récapitulatif

| Association | Type | Fetch | Cascade | orphanRemoval | Justification |
|---|---|---|---|---|---|
| Contrat → Paiement | `@OneToMany` | LAZY | ALL | true | Un paiement ne peut pas exister sans son contrat (composition). Supprimer le contrat doit supprimer ses paiements. Retirer un paiement de la liste doit le supprimer en base. |
| Paiement → Contrat | `@ManyToOne` | LAZY | — | — | Côté propriétaire : porte la FK. Pas de cascade remontante vers le parent. |
| Reservation ↔ Contrat (Reservation côté inverse) | `@OneToOne` | LAZY | ALL | — | Un contrat est lié à une seule réservation. Créer/supprimer une réservation propage l'opération sur le contrat associé. |
| Contrat → Reservation (Contrat côté propriétaire) | `@OneToOne` | LAZY | — | — | Porte la FK `id_reservation`. Pas de cascade depuis le contrat vers la réservation. |
| Agence → Vehicule | `@OneToMany` | LAZY | Aucune | false | Un véhicule survit à la suppression de son agence (agrégation). Aucune cascade de suppression. |
| Vehicule → Agence | `@ManyToOne` | LAZY | — | — | Côté propriétaire : porte la FK `id_agence`. |
| Agence → Employe | `@OneToMany` | LAZY | Aucune | false | Un employé peut être réaffecté à une autre agence ; il survit à la suppression de son agence. Aucune cascade. |
| Employe → Agence | `@ManyToOne` | LAZY | — | — | Côté propriétaire : porte la FK `id_agence`. |
| Vehicule ↔ Equipement | `@ManyToMany` | LAZY | Aucune | — | Les équipements sont partagés entre plusieurs véhicules (agrégation partagée). Supprimer un véhicule ne doit pas supprimer les équipements. On utilise un `Set` pour éviter les doublons. Table de jointure : `vehicule_equipement`. |
| Client → Reservation | `@OneToMany` | LAZY | PERSIST | false | Permet d'enregistrer les réservations en même temps que le client (pratique à la création). Pas de cascade REMOVE : une réservation confirmée ne doit pas disparaître si le compte client est supprimé. |
| Reservation → Client | `@ManyToOne` | LAZY | — | — | Côté propriétaire : porte la FK `id_client`. |
| Reservation → Vehicule | `@ManyToOne` | LAZY | Aucune | — | Un véhicule est indépendant de ses réservations. Supprimer une réservation ne doit pas supprimer le véhicule. |
| Vehicule → Maintenance | `@OneToMany` | LAZY | PERSIST | false | Permet d'enregistrer une maintenance en même temps que le véhicule. Pas de cascade REMOVE : on conserve l'historique de maintenance même en cas de modification du véhicule. |
| Maintenance → Vehicule | `@ManyToOne` | LAZY | — | — | Côté propriétaire : porte la FK `id_vehicule`. |

---

## Détail par entité

### Contrat ↔ Paiement
```java
// Contrat.java — côté inverse
@OneToMany(mappedBy = "contrat", cascade = CascadeType.ALL,
        orphanRemoval = true, fetch = FetchType.LAZY)
private List<Paiement> paiements = new ArrayList<>();

// Paiement.java — côté propriétaire
@ManyToOne(fetch = FetchType.LAZY)
private Contrat contrat;
```
`cascade = ALL` + `orphanRemoval = true` : composition forte — le paiement n'a aucun sens hors d'un contrat.

---

### Reservation ↔ Contrat
```java
// Contrat.java — côté propriétaire (porte la FK id_reservation)
@OneToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "id_reservation")
private Reservation reservation;

// Reservation.java — côté inverse
@OneToOne(mappedBy = "reservation", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
private Contrat contrat;
```
Cascade ALL côté Reservation : créer une réservation avec un contrat propage la persistance.

---

### Agence → Vehicule / Employe
```java
// Agence.java
@OneToMany(mappedBy = "agence", fetch = FetchType.LAZY)
private List<Vehicule> vehicules = new ArrayList<>();

@OneToMany(mappedBy = "agence", fetch = FetchType.LAZY)
private List<Employe> employes = new ArrayList<>();
```
Aucune cascade : agrégation simple. Véhicules et employés ont un cycle de vie indépendant.

---

### Vehicule ↔ Equipement
```java
// Vehicule.java — côté propriétaire
@ManyToMany(fetch = FetchType.LAZY)
@JoinTable(
    name = "vehicule_equipement",
    joinColumns = @JoinColumn(name = "id_vehicule"),
    inverseJoinColumns = @JoinColumn(name = "id_equipement")
)
private Set<Equipement> equipements = new HashSet<>();

// Equipement.java — côté inverse
@ManyToMany(mappedBy = "equipements", fetch = FetchType.LAZY)
private Set<Vehicule> vehicules = new HashSet<>();
```
`Set` plutôt que `List` pour éviter les doublons. Aucune cascade : les équipements sont des ressources partagées.

---

### Client → Reservation
```java
// Client.java — côté inverse
@OneToMany(mappedBy = "client", cascade = CascadeType.PERSIST, fetch = FetchType.LAZY)
private List<Reservation> reservations = new ArrayList<>();

// Reservation.java — côté propriétaire
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "id_client")
private Client client;
```
`cascade = PERSIST` uniquement : on peut créer les réservations avec le client, mais la suppression du client ne cascade pas.

---

### Vehicule → Maintenance
```java
// Vehicule.java — côté inverse
@OneToMany(mappedBy = "vehicule", cascade = CascadeType.PERSIST, fetch = FetchType.LAZY)
private List<Maintenance> maintenances = new ArrayList<>();

// Maintenance.java — côté propriétaire
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "id_vehicule")
private Vehicule vehicule;
```
`cascade = PERSIST` : l'historique de maintenance doit être conservé même si le véhicule est modifié.
