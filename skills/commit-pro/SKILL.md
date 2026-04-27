---
name: commit
description: Genera commit message convencional desde git diff
disable-model-invocation: true
---

# Commit Message Pro

## Proceso

1. Ejecuto: `git diff --cached`
2. Analizo cambios
3. Determino type + scope
4. Genero mensaje formato Conventional Commits

## Types (Conventional Commits)

- **feat:** Nueva feature
- **fix:** Bug fix
- **refactor:** Refactorización (sin cambio funcional)
- **perf:** Mejora de performance
- **test:** Agregar/modificar tests
- **docs:** Documentación
- **style:** Formato (linting, espacios, etc)
- **chore:** Mantenimiento (deps, config)
- **ci:** CI/CD changes

## Scope Inference

Basado en path de archivos modificados:

- `internal/domain/entity/user*` → scope: `user`
- `internal/application/usecase/auth*` → scope: `auth`
- `internal/infrastructure/persistence/*` → scope: `db`
- `internal/presentation/handler/*` → scope: `handler`
- `migrations/*` → scope: `migration`
- `cmd/api/*` → scope: `main`

## Formato Output

```

<type>(<scope>): <description en presente, lowercase, max 50 chars>

[Body opcional explicando POR QUÉ si es complejo]

[Footer: Breaking changes o issues]

```

## Reglas

- Description: presente, lowercase, sin punto final
- Max 50 chars primera línea (GitHub trunca después)
- Body: wrap a 72 chars
- Breaking changes: `BREAKING CHANGE:` en footer

## Ejecución

Después de generar, ejecuto:

```bash
git commit -m "mensaje generado"
```

## Ejemplos

### Example 1: Feature simple

```
git diff --cached:
+ internal/application/usecase/login.go: func Execute()

Output:
feat(auth): implement login use case
```

### Example 2: Bug fix con context

```
git diff --cached:
- internal/infrastructure/persistence/postgres_user_repo.go: missing ErrNotFound mapping

Output:
fix(db): map sql.ErrNoRows to repository.ErrNotFound

FindByEmail was returning raw sql.ErrNoRows instead of
the sentinel error, breaking callers that used errors.Is().
```

### Example 3: Breaking change

```
git diff --cached:
- internal/domain/repository/user_repository.go: FindByID signature changed

Output:
feat(domain): add context to UserRepository interface

BREAKING CHANGE: All repository methods now require context.Context.
Implementations must be updated to pass ctx to sql calls.
```
