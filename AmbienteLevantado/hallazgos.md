# Semana3_quiz — Hallazgos y Pruebas

## FASE 1 — Levantar el ambiente

**Repo:** `https://github.com/lanvargas94/Semana3_quiz` :contentReference[oaicite:0]{index=0}

**Levante sugerido (por docker-compose):**
- `docker compose up --build`

**Criterio de éxito**
- Endpoint: `GET http://localhost:8080/health`
- Respuesta esperada:
```json
{"ok": true}

## FASE 2 — Auditoría del código


> Nota: varios archivos del repo están en una sola línea (minificados).  
> Por eso “línea aprox.” se marca como 1 y se referencia por método/bloque.

| # | Descripción del problema | Archivo | Línea aprox. | Principio violado | Riesgo |
|---|--------------------------|--------|--------------|-------------------|--------|
| 1 | Construcción de consultas SQL por concatenación en login (vulnerable a SQL Injection). | `UserRepository.java` (findByUsername) | 1 | Seguridad básica (SQL Injection) | Alto |
| 2 | Inserción SQL por concatenación en register (otro vector de SQL Injection). | `UserRepository.java` (save) | 1 | Seguridad básica (SQL Injection) | Alto |
| 3 | Contraseñas almacenadas con MD5 (hash débil, sin salt). | `AuthService.java` (método `md5`) | 1 | Seguridad básica (hashing inseguro) | Alto |
| 4 | Exposición de datos sensibles: el login retorna el hash del password ingresado. | `AuthService.java` (login) | 1 | Seguridad / Principio de mínima exposición | Alto |
| 5 | Violación de **SRP**: `AuthService` mezcla validación, hashing, lógica de negocio y logs. | `AuthService.java` | Archivo completo | SOLID — SRP | Medio |
| 6 | Violación de **DIP**: `AuthService` depende directamente de `UserRepository` (acoplamiento a implementación concreta). | `AuthService.java` (instancia directa del repo) | 1 | SOLID — DIP | Medio |
| 7 | Violación de **OCP**: la regla de contraseña está “quemada” (`p.length() > 3`), para cambiarla hay que modificar el método. | `AuthService.java` (register) | 1 | SOLID — OCP | Bajo / Medio |
| 8 | Credenciales de la base de datos hardcodeadas en el repositorio. | `UserRepository.java` | 1 | Clean Code / Seguridad | Alto |
| 9 | Fuga de recursos: conexiones/statement/resultset sin `try-with-resources`. | `UserRepository.java` | 1 | Clean Code / Robustez | Medio |
| 10 | Password enviado por query params (`/login?u=...&p=...`), queda expuesto en logs y proxies. | `AuthController.java` | 1 | Seguridad básica | Alto |

## 🧪 FASE 3 — Pruebas funcionales

A continuación se documentan las pruebas realizadas sobre los endpoints de autenticación del sistema.

---

### Prueba 1 — Login válido

**Comando ejecutado**
```bash
POST "http://localhost:8080/login?u=admin&p=12345"

Resultado observado
El servicio responde con un JSON indicando que el login fue exitoso (ok: true).
En la respuesta se incluye el nombre de usuario autenticado.
También se retorna un campo hash correspondiente al hash (MD5) de la contraseña ingresada.

### Prueba 2 - SQL injection
POST "http://localhost:8080/login?u=admin'--&p=cualquiercosa"
Resultado observado
El sistema procesa el request sin sanitizar el parámetro u.
La consulta SQL se construye por concatenación, por lo que el payload admin'-- altera la estructura del query.
Aunque el login no se completa por la validación de la contraseña, el endpoint demuestra que es vulnerable a inyección SQL.


### Prueba - Registro de password debil
POST "http://localhost:8080/register?u=test&p=123&e=test@test.com"
POST "http://localhost:8080/register?u=test&p=1234&e=test@test.com"
Resultado observado
El registro con contraseña 123 es rechazado.
El registro con contraseña 1234 es aceptado.
