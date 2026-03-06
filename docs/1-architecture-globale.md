# Architecture Globale

## 1) Vue d'ensemble

```text
Mobile/Web App
   -> API Gateway (Gin)
      -> Catalog API / Availability API
      -> Cart API / Smart Cart API
      -> Subscription API
      -> Reservation Service
      -> PostgreSQL / Redis

SFTP (WMS XML)
   -> Worker Poller
   -> Worker Parser/Validator
   -> Worker Reconcile Stock
   -> PostgreSQL (staging + stock tables)
```

En pratique :
- l'application cliente parle à l'API,
- l'API lit/écrit dans PostgreSQL et Redis,
- les fichiers WMS sont traités en arrière-plan par des workers.

## 2) Semaine type

- Lundi -> jeudi (approvisionnement) :
  on synchronise le WMS et on prépare le stock de la semaine.
- Jeudi soir -> samedi 23h59 (vente ouverte) :
  le catalogue est lu depuis une projection rapide,
  Smart Cart réserve en masse.
- Dimanche (fermeture) :
  on fige les paniers, on facture, on exporte les commandes au WMS.

## 3) Règles métier clés

- `FRUITS_LEGUMES` : vendable en continu (pas de rupture côté backend).
- `EPICERIE` : stock limité, réservation obligatoire.
- `FRAIS` : stock limité + règle DDM.
  Un produit frais est vendable seulement si `DDM >= date_livraison_client`.

## 4) Gestion du pic Smart Cart

Choix d'architecture :
- lecture rapide via des données pré-calculées,
- réservation transactionnelle en base (`reserved_qty`),
- files de jobs avec concurrence limitée,
- idempotence : 1 prefill maximum par `customer_id + week_id`.

Résultat attendu :
- API rapide pendant le pic,
- pas de corruption de stock en concurrence.

## 5) API cible

Endpoint : `GET /category/{categoryID}/products`

Entrées :
- `categoryID` (path),
- contexte client (auth) : `customer_id`, abonnement actif, jour de livraison.

Sortie :
- uniquement les produits disponibles pour ce client sur la semaine ouverte.

Étapes :
1. Trouver la semaine de vente ouverte.
2. Trouver la date de livraison du client.
3. Lire la projection de disponibilité.
4. Joindre les informations produit/catégorie.

## 6) Résilience

- Pipeline WMS en 3 phases : `raw file -> staging -> reconcile`.
- Transactions courtes.
- Idempotence via checksum fichier.
- Un fichier en erreur peut être rejoué sans doublon logique.
