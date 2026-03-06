# Modules et Workers (Go)

## 1) Arborescence proposée

```text
internal/
  modules/
    catalog/
    subscription/
    stock/
    cart/
    smartcart/
    wms/
    saleswindow/
    order/
  infra/
    router/
    database/
      migrations/
    worker/
      wwms/
      wsmartcart/
      wsales/
```

## 2) Rôle de chaque module

- `catalog` : produits, catégories, métadonnées.
- `subscription` : jour et date de livraison client.
- `stock` : réserver, relâcher, valider le stock.
- `cart` : gestion du panier et orchestration des réservations.
- `smartcart` : logique de pré-remplissage.
- `wms` : lecture XML, validation, mapping SKU, réconciliation.
- `saleswindow` : ouverture/fermeture de la semaine de vente.
- `order` : transformation panier -> commande.

## 3) Workers

### `wwms` (synchronisation stock)

Sous-jobs :
1. `poll_sftp_job`
2. `parse_validate_xml_job`
3. `reconcile_stock_job`
4. `rebuild_slots_job`

Garanties :
- idempotence par checksum,
- retries exponentiels,
- dead-letter queue pour les fichiers invalides.

### `wsmartcart` (pic du jeudi)

Sous-jobs :
1. `enqueue_prefill_batch_job`
2. `prefill_cart_job` (1 client)

Garanties :
- idempotence par clé `prefill:{week_id}:{customer_id}`,
- concurrence bornée,
- retries courts avec jitter.

### `wsales` (fermeture)

Sous-jobs :
1. `close_sales_week_job`
2. `finalize_cart_to_order_job`
3. `export_orders_wms_job`

### Exemple d'implémentation 

Ces extraits montrent le pattern concret :
- un processor central qui enregistre les handlers,
- un distributor pour pousser les jobs,
- un handler spécialisé par domaine.

#### Processor central (`RedisTaskProcessor`)

```go
package worker

type RedisTaskProcessor struct {
    server *asynq.Server
    mux    *asynq.ServeMux

    db               *gorm.DB
    idempotency      idempotency.Manager
    stockSyncService wstock.SyncService
    smartCartService wsmartcart.PrefillService
    salesService     wsales.CloseWeekService
}

func NewRedisTaskProcessor(
    redisOpt asynq.RedisClientOpt,
    concurrency int,
    db *gorm.DB,
    idp idempotency.Manager,
    stockSyncService wstock.SyncService,
    smartCartService wsmartcart.PrefillService,
    salesService wsales.CloseWeekService,
) *RedisTaskProcessor {
    server := asynq.NewServer(redisOpt, asynq.Config{
        Concurrency: concurrency,
        Queues: map[string]int{
            "critical": 6, // sync WMS / fermeture semaine
            "default":  4, // prefill Smart Cart
            "low":      2, // jobs secondaires
        },
    })

    return &RedisTaskProcessor{
        server:           server,
        mux:              asynq.NewServeMux(),
        db:               db,
        idempotency:      idp,
        stockSyncService: stockSyncService,
        smartCartService: smartCartService,
        salesService:     salesService,
    }
}

func (p *RedisTaskProcessor) Start() error {
    p.registerHandlers()
    return p.server.Run(p.mux)
}

func (p *RedisTaskProcessor) registerHandlers() {
    stockHandler := wstock.NewInventoryTaskHandler(p.db, p.idempotency, p.stockSyncService)
    p.mux.HandleFunc(wstock.TypeInventorySync, stockHandler.HandleInventorySyncTask)

    smartCartHandler := wsmartcart.NewPrefillTaskHandler(p.smartCartService, p.idempotency)
    p.mux.HandleFunc(wsmartcart.TypePrefill, smartCartHandler.HandlePrefillTask)

    salesHandler := wsales.NewCloseWeekTaskHandler(p.salesService, p.idempotency)
    p.mux.HandleFunc(wsales.TypeCloseWeek, salesHandler.HandleCloseWeekTask)
}

func (p *RedisTaskProcessor) Shutdown() {
    p.server.Shutdown()
}
```

#### Distributor (`TaskDistributor`)

```go
package worker

type TaskDistributor interface {
    EnqueueTask(ctx context.Context, task *asynq.Task, opts ...asynq.Option) error
}

type RedisTaskDistributor struct {
    client *asynq.Client
}

func NewRedisTaskDistributor(redisOpt asynq.RedisClientOpt) TaskDistributor {
    return &RedisTaskDistributor{client: asynq.NewClient(redisOpt)}
}

func (d *RedisTaskDistributor) EnqueueTask(
    ctx context.Context,
    task *asynq.Task,
    opts ...asynq.Option,
) error {
    _, err := d.client.EnqueueContext(ctx, task, opts...)
    return err
}
```

#### Handler WMS (`wstock`)

```go
package wstock

const TypeInventorySync = "inventory:wms:sync"

type InventorySyncPayload struct {
    ShopID       uuid.UUID `json:"shop_id"`
    FileID       uuid.UUID `json:"file_id"`
    FileChecksum string    `json:"file_checksum"`
}

type InventoryTaskHandler struct {
    db          *gorm.DB
    idempotency idempotency.Manager
    syncService SyncService
}

func NewInventoryTaskHandler(
    db *gorm.DB,
    idp idempotency.Manager,
    syncService SyncService,
) *InventoryTaskHandler {
    return &InventoryTaskHandler{db: db, idempotency: idp, syncService: syncService}
}

func (h *InventoryTaskHandler) HandleInventorySyncTask(ctx context.Context, t *asynq.Task) error {
    var p InventorySyncPayload
    if err := json.Unmarshal(t.Payload(), &p); err != nil {
        return fmt.Errorf("failed to unmarshal payload: %w", asynq.SkipRetry)
    }

    idemKey := fmt.Sprintf("wms:sync:%s", p.FileChecksum)
    locked, release, err := h.idempotency.Acquire(ctx, idemKey, 10*time.Minute)
    if err != nil {
        return fmt.Errorf("idempotency acquire: %w", err)
    }
    if !locked {
        return nil
    }
    defer release()

    return h.syncService.ProcessFile(ctx, p.ShopID, p.FileID)
}
```

## 4) API à exposer

- `GET /category/{categoryID}/products`
- `POST /carts/{id}/items`
- `PATCH /carts/{id}/items/{itemId}`
- `POST /sales-weeks/{id}/close`

## 5) Découpage de code 

Dans chaque module :
- `handler` (HTTP),
- `service` (métier),
- `repository` (base de données),
- `types` (DTO).
