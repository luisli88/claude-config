---
name: review
description: Code review exhaustivo con checklist
disable-model-invocation: false
---

# Code Review Skill

## Objetivo

Revisar código staged antes de commit para detectar issues.

## Checklist de Revisión

### 🐛 Bugs Potenciales

- [ ] Errores ignorados (err asignado pero no chequeado)
- [ ] Nil pointer dereference
- [ ] Off-by-one en slices
- [ ] Race conditions en goroutines (vars compartidas sin mutex)
- [ ] Goroutine leaks (goroutines que no terminan)
- [ ] Context cancelation no propagada

### 🔒 Security Issues

- [ ] Input validation (SQL injection, path traversal)
- [ ] Autenticación en endpoints sensibles
- [ ] Secrets hardcodeados (tokens, passwords en código)
- [ ] CORS configurado correctamente
- [ ] Rate limiting en endpoints públicos
- [ ] Datos sensibles en logs

### ⚡ Performance

- [ ] Loops O(n²) o peor
- [ ] Queries N+1 (DB)
- [ ] Allocations innecesarias en hot path
- [ ] DB connections no cerradas (defer rows.Close())
- [ ] Large payloads sin pagination

### 🎨 Code Quality

- [ ] Nombres descriptivos (Go idiomático: `err` no `error`, `r` para reader)
- [ ] Funciones < 50 líneas
- [ ] DRY violations
- [ ] Magic numbers (usar constantes)
- [ ] Comentarios solo donde necesario (exported types/funcs documentadas)
- [ ] No `interface{}` sin justificación
- [ ] Dependency rule respetada (no imports cross-layer incorrectos)

### ✅ Testing

- [ ] Happy path covered
- [ ] Error cases covered (cada rama de error)
- [ ] Edge cases (empty string, zero value, nil)
- [ ] Mocks apropiados (sin tocar DB real en unit tests)
- [ ] Subtests con `t.Run` para legibilidad

### 📚 Documentation

- [ ] Comentarios godoc en funciones/types exportados
- [ ] README actualizado si nueva feature
- [ ] Errores sentinel documentados

## Proceso de Review

1. Leo archivos staged: `git diff --cached`
2. Reviso contra checklist
3. Reporto findings por severidad
4. Sugiero fixes específicos

## Output Format

```markdown
## 🔍 Code Review Results

### ❌ Critical Issues (must fix)

- **File:** `src/auth/login.ts:45`
  **Issue:** SQL injection vulnerability
  **Fix:** Use parameterized query: `prisma.user.findFirst({ where: { email } })`

### ⚠️ Warnings (should fix)

- **File:** `src/services/user.ts:120`
  **Issue:** Function too long (85 lines)
  **Fix:** Extract validation logic to separate function

### 💡 Suggestions (nice to have)

- **File:** `src/utils/format.ts:10`
  **Issue:** Magic number `3600`
  **Fix:** Create constant `SECONDS_IN_HOUR = 3600`

### ✅ Strengths

- Excellent error handling in controllers
- Good test coverage (87%)
- Clear naming conventions

## Recommendation

🔴 DO NOT MERGE - Fix critical issues first
🟡 MERGE WITH CAUTION - Address warnings in follow-up
🟢 APPROVED - Ready to merge
```
