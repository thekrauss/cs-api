# Modélisation PostgreSQL

## 1) Tables principales

### Catalogue

- `products` : informations produit (`sku`, `name`, `nature`, `is_active`, etc.).
- `categories` : informations catégorie (`name`, `slug`).

### Abonnement et semaine de vente

- `subscriptions` : abonnement client, jour de livraison, prochaine livraison.
- `sales_weeks` : fenêtre de vente (`open_at`, `close_at`, `status`).

### Stock (WMS + vente)

- `wms_files` : historique des fichiers importés (checksum, statut, erreur).
- `wms_stock_staging` : données brutes validées avant réconciliation.
- `stock_batches` : stock par lot (entrepôt, DDM, quantités).
- `stock_movements` : journal immuable des mouvements (`RESERVE`, `RELEASE`, etc.).
- `stock_slots` : projection de vente par `week_id + product_id + delivery_date`.

### Panier et réservation

- `carts` : panier client pour une semaine.
- `cart_items` : lignes du panier.
- `reservations` : réservations de stock associées aux lignes panier.

## 2) Index importants

- `products(category_id, is_active, nature)`
- `subscriptions(customer_id, status)`
- `stock_batches(product_id, ddm)`
- `stock_slots(week_id, delivery_date, product_id)` (lecture API)
- `stock_slots(product_id, week_id, delivery_date)` (réservation)
- `reservations(cart_item_id, status)`
- `wms_files(checksum unique)` pour l'idempotence

## 3) Règles d'intégrité

- Les quantités ne peuvent pas être négatives.
- `reserved_qty` ne peut jamais dépasser `sellable_qty`.
- Pour `FRAIS`, `ddm` est obligatoire dans `stock_batches`.
- Un fichier WMS ne doit être traité qu'une fois par checksum.
- Les FK entre ventes/paniers/réservations doivent rester strictes.

## 4) Réservation atomique (stock limité)

Dans une transaction SQL :
1. Verrouiller la ligne `stock_slots` avec `SELECT ... FOR UPDATE`.
2. Vérifier que `sellable_qty - reserved_qty >= qty`.
3. Incrémenter `reserved_qty`.
4. Écrire la réservation et le mouvement de stock.

Ce mécanisme protège contre la survente en forte concurrence.

## 5) Règles de disponibilité API

Pour `GET /category/{categoryID}/products` :
- `FRUITS_LEGUMES` : disponible (si produit actif).
- `EPICERIE` : disponible si `sellable_qty - reserved_qty > 0`.
- `FRAIS` : disponible seulement si `ddm >= delivery_date`.
