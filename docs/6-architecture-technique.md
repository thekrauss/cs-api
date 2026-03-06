# Architecture Technique (Go)

## 1) Un binaire, deux modes

Le même binaire peut tourner en mode API ou en mode worker selon `APP_MODE`.

- mode `server` : lance l'API HTTP,
- mode `worker` : lance les jobs asynchrones.

```go
mode := os.Getenv("APP_MODE")
switch mode {
case "worker":
    if err := app.RunWorker(ctx); err != nil { log.Fatalf("worker error: %v", err) }
default:
    if err := app.RunServer(ctx); err != nil { log.Fatalf("server error: %v", err) }
}
```

## 2) Injection de dépendances

L'application est construite en couches :
`Database -> Repository -> UseCase -> Handler`.

Avantages :
- code testable,
- responsabilités claires,
- dépendances explicites.

## 3) Structure d'un module (exemple `catalog`)

```text
catalog/
├── hcatalog.go
├── repository
│   ├── collection.go
│   └── product.go
├── types
│   ├── catalog_types.go
│   ├── collection_types.go
│   └── inventory_types.go
└── usecase
    ├── delete.go
    ├── get.go
    ├── post.go
    ├── put.go
    └── usecase.go
```

## 4) Repository (naming Bene Bono)

Exemple unique : lecture des produits disponibles pour `GET /category/{categoryID}/products`.

```go
package repository

type CatalogAvailabilityRepository interface {
    ListAvailableByCategory(
        ctx context.Context,
        weekID uuid.UUID,
        deliveryDate time.Time,
        categoryID uuid.UUID,
    ) ([]domain.Product, error)
}

type catalogAvailabilityRepository struct {
    db *gorm.DB
}

func NewCatalogAvailabilityRepository(db *gorm.DB) CatalogAvailabilityRepository {
    return &catalogAvailabilityRepository{db: db}
}

func (r *catalogAvailabilityRepository) ListAvailableByCategory(
    ctx context.Context,
    weekID uuid.UUID,
    deliveryDate time.Time,
    categoryID uuid.UUID,
) ([]domain.Product, error) {
    var products []domain.Product
    err := r.db.WithContext(ctx).
        Joins("JOIN stock_slots ss ON ss.product_id = products.id").
        Where("products.category_id = ?", categoryID).
        Where("ss.week_id = ? AND ss.delivery_date = ?", weekID, deliveryDate).
        Where("ss.sellable_qty - ss.reserved_qty > 0").
        Find(&products).Error
    return products, err
}
```

## 5) UseCase (naming Bene Bono)

Exemple unique : résoudre semaine ouverte + date de livraison client, puis lire le catalogue disponible.

```go
package usecase

type CatalogUseCase interface {
    GetCategoryProducts(ctx context.Context, customerID uuid.UUID, categoryID uuid.UUID) ([]types.CategoryProductResponse, error)
}

type SalesWindowService interface {
    GetOpenWeek(ctx context.Context) (*domain.SalesWeek, error)
}

type SubscriptionService interface {
    ResolveDeliveryDate(ctx context.Context, customerID uuid.UUID, weekID uuid.UUID) (time.Time, error)
}

type catalogUseCase struct {
    repo         repository.CatalogAvailabilityRepository
    salesWindow  SalesWindowService
    subscription SubscriptionService
}

func NewCatalogUseCase(
    repo repository.CatalogAvailabilityRepository,
    salesWindow SalesWindowService,
    subscription SubscriptionService,
) CatalogUseCase {
    return &catalogUseCase{repo: repo, salesWindow: salesWindow, subscription: subscription}
}

func (u *catalogUseCase) GetCategoryProducts(
    ctx context.Context,
    customerID uuid.UUID,
    categoryID uuid.UUID,
) ([]types.CategoryProductResponse, error) {
    week, err := u.salesWindow.GetOpenWeek(ctx)
    if err != nil {
        return nil, err
    }

    deliveryDate, err := u.subscription.ResolveDeliveryDate(ctx, customerID, week.ID)
    if err != nil {
        return nil, err
    }

    products, err := u.repo.ListAvailableByCategory(ctx, week.ID, deliveryDate, categoryID)
    if err != nil {
        return nil, err
    }

    return types.MapProductsToCategoryResponse(products), nil
}
```

## 6) Handler (naming Bene Bono)

Exemple unique : handler du endpoint `GET /category/{categoryID}/products`.

```go
package catalog

type ICatalogController interface {
    GetCategoryProducts(c *gin.Context, in *types.CategoryProductsInput) ([]types.CategoryProductResponse, error)
}

type CatalogController struct {
    usecase usecase.CatalogUseCase
}

func NewCatalogController(uc usecase.CatalogUseCase) ICatalogController {
    return &CatalogController{usecase: uc}
}

func (ctrl *CatalogController) GetCategoryProducts(
    c *gin.Context,
    in *types.CategoryProductsInput,
) ([]types.CategoryProductResponse, error) {
    customerID := auth.CustomerIDFromContext(c.Request.Context())
    return ctrl.usecase.GetCategoryProducts(c.Request.Context(), customerID, in.CategoryID)
}
```

## 7) Types API (naming Bene Bono)

```go
package types

type CategoryProductsInput struct {
    CategoryID uuid.UUID `path:"categoryID" validate:"required"`
}

type CategoryProductResponse struct {
    ProductID    uuid.UUID  `json:"product_id"`
    SKU          string     `json:"sku"`
    Name         string     `json:"name"`
    Nature       string     `json:"nature"`
    AvailableQty int        `json:"available_qty"`
    DDM          *time.Time `json:"ddm,omitempty"`
}
```

## 8) Exemple App + containers + routes

```go
type App struct {
    Config *config.GlobalConfig
    Logger *logrus.Logger

    DB         *gorm.DB
    Cache      cache.AuthCache
    Middleware *middleware.Manager
    HTTPServer Server

    Repos       *RepositoryContainer
    Services    *ServiceContainer
    Controllers *ControllerContainer

    Distributor worker.TaskDistributor
    Idempotency idempotency.Manager
}

type RepositoryContainer struct {
    CatalogAvailability catalogrepo.CatalogAvailabilityRepository
}

type ServiceContainer struct {
    Catalog catalogusecase.CatalogUseCase
}

type ControllerContainer struct {
    Catalog catalogCtrl.ICatalogController
}

func AddAllRoutes(a *App) {
    if a == nil {
        return
    }
    addCatalogRoutes(a)
    // autres modules...
}
```

## 9) Centralisation des dépendances (focus `catalog`)

```go
func (a *App) initDomainLayers() error {
    if a.DB == nil {
        return fmt.Errorf("database is required")
    }

    // Repository
    catalogRepo := catalogrepo.NewCatalogAvailabilityRepository(a.DB)
    a.Repos = &RepositoryContainer{
        CatalogAvailability: catalogRepo,
    }

    // Services utilisés par le usecase catalog
    salesWindowSvc := saleswindow.NewService(a.DB)
    subscriptionSvc := subscription.NewService(a.DB)

    // UseCase
    catalogUC := catalogusecase.NewCatalogUseCase(catalogRepo, salesWindowSvc, subscriptionSvc)
    a.Services = &ServiceContainer{
        Catalog: catalogUC,
    }

    // Controller
    a.Controllers = &ControllerContainer{
        Catalog: catalogCtrl.NewCatalogController(catalogUC),
    }

    return nil
}
```

## 10) Routing catalog

```go
var (
    ShopCatalog         = ShopResource.NewGroup("/catalog", "Catalogue: produits & variantes")
    ShopCatalogProducts = ShopCatalog.NewGroup("/:categoryID/products", "Client unique")
)

func addCatalogRoutes(a *App) {
    ctrl := a.Controllers.Catalog

    ShopCatalogProducts.AddRoute("",http.MethodGet,"Lister les produits disponibles d'une catégorie",tonic.Handler(ctrl.GetCategoryProducts, httpStatusOK),)
}
```

## 11) Workers Asynq

Les jobs clés dans ce contexte :
- sync stock WMS,
- prefill Smart Cart,
- fermeture de semaine.

```go
func (p *RedisTaskProcessor) registerHandlers() {
    p.mux.HandleFunc(wstock.TypeInventorySync, stockHandler.HandleInventorySyncTask)
    p.mux.HandleFunc(wsmartcart.TypePrefill, smartCartHandler.HandlePrefillTask)
    p.mux.HandleFunc(wsales.TypeCloseWeek, salesHandler.HandleCloseWeekTask)
}
```

## 12) Bénéfices

- Noms alignés avec le métier Bene Bono.
- Flux lisible du endpoint jusqu'à la DB.
- Wiring centralisé et facile à maintenir.
