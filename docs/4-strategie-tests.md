# Stratégie de Tests

## 1) But

Protéger les règles les plus sensibles :
- pas de survente,
- respect de la DDM sur les produits frais,
- ingestion WMS idempotente.

## 2) Priorité des tests

1. `stock/services`
2. `smartcart/services`
3. `wms/services`
4. `cart/services`
5. `catalog/services`

## 3) Cas unitaires essentiels

### Stock et réservation

- Réservation OK si stock disponible.
- Réservation KO si stock insuffisant.
- Deux réservations concurrentes : une seule passe si le stock est limité.
- `release` rend le stock disponible.
- `commit` transforme la réservation en mouvement final.

### Produits frais (DDM)

- Produit visible si `delivery_date <= ddm`.
- Produit non visible si `delivery_date > ddm`.
- Recalcul correct après nouvel import WMS.

### Smart Cart

- Prefill idempotent par client et par semaine.
- En cas d'échec d'un item : fallback ou skip propre.
- Pas de duplication d'item, pas de quantité négative.

### WMS

- Même checksum => fichier traité une seule fois.
- XML invalide => erreur enregistrée, pas de mise à jour stock.
- Panne pendant reconcile => rollback.
- Retry => état final cohérent.

### API Catalogue

- `GET /category/{id}/products` retourne seulement les produits disponibles.
- Fruits/légumes visibles (si actifs).
- Épicerie filtrée par stock.
- Frais filtré par DDM + date de livraison.

## 4) Intégration ciblée

- Pipeline complet : XML -> staging -> `stock_slots`.
- Pic du jeudi : nombreux jobs Smart Cart simultanés sans survente.
- Fermeture dimanche : paniers finalisés + export généré.

## 5) Outils recommandés

- Doubles de repository (`*_stub`) pour les services.
- Tests SQL transactionnels sur PostgreSQL local/docker.
- `go test -race` sur `stock` et `smartcart`.
- Fixtures déterministes (`week_id`, `delivery_date`, `ddm`).
