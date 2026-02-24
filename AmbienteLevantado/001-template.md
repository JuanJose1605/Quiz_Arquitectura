# ADR-001: Refactorización del módulo de autenticación por vulnerabilidades críticas de seguridad y violaciones a SOLID

## Contexto
El sistema actual implementa autenticación mediante los endpoints `/login` y `/register`, recibiendo credenciales por query params y validándolas contra la base de datos. En la auditoría del código se identificaron problemas críticos: construcción de consultas SQL por concatenación (hallazgos #1 y #2), uso de MD5 para el hashing de contraseñas (hallazgo #3), exposición de información sensible en la respuesta del login (hallazgo #4) y credenciales de base de datos hardcodeadas (hallazgo #8). Además, el módulo presenta problemas de diseño al concentrar múltiples responsabilidades en una sola clase (violación de SRP) y depender directamente de implementaciones concretas de acceso a datos (violación de DIP), lo que dificulta pruebas y cambios futuros (hallazgos #5 y #6).  
Estos problemas son urgentes porque el módulo de autenticación es el principal punto de entrada del sistema; una vulnerabilidad aquí puede comprometer cuentas de usuarios, datos y la confianza en la aplicación. El equipo se ve afectado por la deuda técnica y la dificultad de mantener el código, mientras que los usuarios están expuestos a riesgos de acceso no autorizado y filtración de información.

## Decisión
1) **Eliminar SQL Injection con consultas parametrizadas**  
Se reemplazará la construcción de SQL por concatenación por consultas parametrizadas (PreparedStatement/JdbcTemplate o equivalente). Esta decisión se alinea con prácticas de seguridad estándar y evita que datos del usuario alteren la estructura de las consultas. Se elige esta opción por ser efectiva, simple de aplicar y ampliamente soportada por el stack actual.

2) **Fortalecer el manejo de contraseñas y la exposición de datos**  
Se sustituirá MD5 por un algoritmo de hashing seguro para contraseñas (por ejemplo BCrypt con salt) y se eliminará el retorno de información sensible en las respuestas del login (no más `hash` ni datos internos). Esta solución se elige porque BCrypt está diseñado para contraseñas (resistente a fuerza bruta) y reduce el impacto en caso de filtraciones.

3) **Aplicar SOLID para separar responsabilidades y reducir acoplamiento**  
Se dividirá el módulo de autenticación en capas claras:  
- `AuthController` (orquestación de requests/responses),  
- `AuthService` (lógica de negocio),  
- `PasswordService` (hashing y validación),  
- `UserRepository` (acceso a datos).  
Además, el servicio dependerá de una interfaz del repositorio (DIP) para facilitar pruebas y cambios de implementación. Esto previene “código espagueti”, mejora la mantenibilidad y habilita pruebas unitarias.

4) **Mejorar validaciones y configuración segura**  
Se ampliarán las reglas de validación de contraseñas (más que longitud mínima) y se moverán las credenciales de la base de datos a variables de entorno/configuración. Esta decisión se toma para reducir cuentas débiles y eliminar secretos del código fuente.

## Consecuencias

### Consecuencias positivas
- Se elimina el principal vector de SQL Injection en login y registro.  
- Se reduce el riesgo de filtración de credenciales al usar hashing adecuado y no exponer datos sensibles.  
- El código queda más legible, testeable y mantenible al aplicar SRP y DIP.  
- Se facilita la evolución del módulo (nuevas políticas de contraseña, cambios de persistencia) sin romper otras partes.

### Consecuencias o riesgos
- Requiere tiempo de refactorización y pruebas para evitar regresiones en autenticación.  
- Puede requerir una estrategia de migración para contraseñas existentes (compatibilidad temporal).  
- Cambios en el contrato de respuestas pueden impactar consumidores actuales del endpoint.  
- Existe riesgo de introducir bugs si no se acompañan los cambios con pruebas básicas.

## Alternativas consideradas
- **Reescribir el módulo de autenticación desde cero** → descartado por el tiempo y el alcance del quiz; se prioriza refactor incremental para mitigar riesgos críticos primero.  
- **Aplicar solo parches de seguridad sin refactorizar el diseño** → descartado porque, aunque reduce riesgos inmediatos, mantiene problemas de SRP/DIP que dificultan el mantenimiento y favorecen la reintroducción de errores en el futuro.