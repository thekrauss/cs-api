#  Documentation Design (Stock et Catalogue)

## Objectif du projet

Construire un système fiable pour :
- synchroniser le stock depuis le WMS (SFTP + XML),
- gérer la vente hebdomadaire par abonnement,
- absorber un gros pic de réservations (Smart Cart),
- exposer une API catalogue stable : `GET /category/{categoryID}/products`.

## Ordre de lecture 

0. [Introduction](./0-introduction.md)
1. [Architecture globale](./1-architecture-globale.md)
2. [Modélisation PostgreSQL](./2-modelisation-postgresql.md)
3. [Modules et workers](./3-modules-workers.md)
4. [Stratégie de tests](./4-strategie-tests.md)
5. [Stratégie de cache](./5-strategie-cache.md)
6. [Architecture technique Go](./6-architecture-technique.md)

## Principes importants

- Le backend Go est la source de vérité pour le catalogue.
- Le WMS est la source de vérité pour le stock.
- Les réservations sont atomiques pour éviter la survente.
- La disponibilité est pré-calculée pour garder une API rapide.
- L'ingestion WMS est asynchrone, idempotente et rejouable.

https://link.excalidraw.com/l/ZTJJsaum1t/1eVcEVf7FYC