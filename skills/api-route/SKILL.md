---
name: api-route
description: Genera API endpoint con Clean Architecture en Go
disable-model-invocation: false
---

# API Route Generator Skill

## Cuándo usar este skill

Cuando necesites crear un nuevo endpoint REST en Go con Clean Architecture.

## Arquitectura que SIEMPRE uso

```
internal/
├── domain/
│   ├── entity/          ← Entidades (structs puros, sin deps)
│   └── repository/      ← Interfaces de repositorio
├── application/
│   └── usecase/         ← Lógica de negocio (depende solo de domain)
├── infrastructure/
│   └── persistence/     ← Implementaciones concretas de repositorio
└── presentation/
    ├── dto/             ← Request/Response structs
    ├── handler/         ← HTTP handlers
    └── router/          ← Registro de rutas
```

### Capa 1: Domain Entity (internal/domain/entity/\*.go)

**Responsabilidad:** Struct puro sin dependencias externas

```go
package entity

import "time"

type Product struct {
    ID        int
    Name      string
    Price     float64
    Category  string
    CreatedAt time.Time
}
```

### Capa 2: Repository Interface (internal/domain/repository/\*.go)

**Responsabilidad:** Contrato de acceso a datos (solo interfaces)

```go
package repository

import "myapp/internal/domain/entity"

var ErrNotFound = errors.New("not found")

type ProductRepository interface {
    Create(p entity.Product) (*entity.Product, error)
    FindByID(id int) (*entity.Product, error)
}
```

### Capa 3: Use Case (internal/application/usecase/\*.go)

**Responsabilidad:** Lógica de negocio, validación, orquestación

```go
package usecase

import (
    "errors"
    "myapp/internal/domain/entity"
    "myapp/internal/domain/repository"
)

var (
    ErrInvalidInput = errors.New("invalid input")
    ErrNotFound     = errors.New("product not found")
)

type CreateProduct struct {
    repo repository.ProductRepository
}

func NewCreateProduct(repo repository.ProductRepository) *CreateProduct {
    return &CreateProduct{repo: repo}
}

func (uc *CreateProduct) Execute(name string, price float64, category string) (*entity.Product, error) {
    if name == "" {
        return nil, ErrInvalidInput
    }
    if price < 0 {
        return nil, ErrInvalidInput
    }
    p := entity.Product{Name: name, Price: price, Category: category}
    return uc.repo.Create(p)
}
```

### Capa 4: DTO (internal/presentation/dto/\*.go)

**Responsabilidad:** Estructuras de serialización HTTP

```go
package dto

type CreateProductRequest struct {
    Name     string  `json:"name"`
    Price    float64 `json:"price"`
    Category string  `json:"category"`
}

type CreateProductResponse struct {
    ID        int     `json:"id"`
    Name      string  `json:"name"`
    Price     float64 `json:"price"`
    Category  string  `json:"category"`
    CreatedAt string  `json:"created_at"`
}
```

### Capa 5: Handler (internal/presentation/handler/\*.go)

**Responsabilidad:** Manejar request/response HTTP, NO lógica de negocio

```go
package handler

import (
    "encoding/json"
    "net/http"
    "myapp/internal/application/usecase"
    "myapp/internal/presentation/dto"
)

type ProductHandler struct {
    createProduct *usecase.CreateProduct
}

func NewProductHandler(uc *usecase.CreateProduct) *ProductHandler {
    return &ProductHandler{createProduct: uc}
}

func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    var req dto.CreateProductRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid body", http.StatusBadRequest)
        return
    }

    product, err := h.createProduct.Execute(req.Name, req.Price, req.Category)
    if err != nil {
        if errors.Is(err, usecase.ErrInvalidInput) {
            http.Error(w, err.Error(), http.StatusBadRequest)
            return
        }
        http.Error(w, "internal error", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(dto.CreateProductResponse{
        ID:       product.ID,
        Name:     product.Name,
        Price:    product.Price,
        Category: product.Category,
    })
}
```

### Tests (internal/application/usecase/\*\_test.go)

**Responsabilidad:** Validar comportamiento con mock del repositorio

```go
package usecase_test

import (
    "testing"
    "myapp/internal/application/usecase"
    "myapp/internal/domain/repository"
)

func TestCreateProduct_Success(t *testing.T) {
    repo := repository.NewMockProductRepository()
    uc := usecase.NewCreateProduct(repo)

    product, err := uc.Execute("Widget", 9.99, "tools")

    if err != nil {
        t.Fatalf("expected no error, got %v", err)
    }
    if product.Name != "Widget" {
        t.Errorf("expected name Widget, got %s", product.Name)
    }
}

func TestCreateProduct_InvalidInput(t *testing.T) {
    repo := repository.NewMockProductRepository()
    uc := usecase.NewCreateProduct(repo)

    _, err := uc.Execute("", 9.99, "tools")

    if !errors.Is(err, usecase.ErrInvalidInput) {
        t.Errorf("expected ErrInvalidInput, got %v", err)
    }
}
```

## Checklist de implementación

Cuando crees un endpoint, DEBES:

- [ ] Entity en `domain/entity/`
- [ ] Interface en `domain/repository/`
- [ ] Use case en `application/usecase/`
- [ ] DTO request/response en `presentation/dto/`
- [ ] Handler en `presentation/handler/`
- [ ] Ruta registrada en `presentation/router/`
- [ ] Use case wired en `cmd/api/main.go`
- [ ] Mínimo 2 tests de use case (happy + error path)

## Errores comunes a EVITAR

❌ Lógica de negocio en handler
❌ Acceso directo a DB desde handler o use case
❌ Importar `infrastructure/` desde `application/` o `domain/`
❌ `interface{}` sin justificación
❌ Commits sin tests
