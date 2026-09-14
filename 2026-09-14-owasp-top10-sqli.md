# OWASP Top 10 Deep Dive: Rompiendo la Lógica de las Bases de Datos con SQLi

**Fecha:** 14 de Septiembre de 2026  
**Categoría:** Seguridad Web | OWASP Top 10 | Pentesting  
**Autor:** 0xBlackCanary  

---

## 📌 Introducción al Riesgo: A03:2021-Injection
En el marco de trabajo de OWASP, las inyecciones ocupan el tercer puesto de los riesgos más críticos en aplicaciones web a nivel mundial. Una **Inyección SQL (SQLi)** ocurre cuando datos proporcionados por el usuario son interpretados directamente por el motor de la base de datos como comandos de código estructurado, rompiendo la barrera de seguridad de la aplicación.

Hoy analizaremos cómo un auditor puede saltarse una pasarela de autenticación y extraer información privilegiada del búnker mediante payloads manuales y automatizados.

---

## 🛠️ Escenario de Laboratorio: El Panel de Autenticación Vulnerable

Imaginemos un backend clásico desarrollado en PHP que gestiona el acceso de los administradores mediante la siguiente consulta insegura:

```php
\$query = "SELECT * FROM usuarios WHERE username = '" . \(_POST['user'] . "' AND password = '" . \)_POST['pass'] . "'";
```

### El Fallo de Diseño
El código concatena directamente los inputs del usuario sin sanitizar ni utilizar sentencias preparadas (*Prepared Statements*). Esto permite alterar la lógica booleana de la consulta estructurada SQL.

---

## 🧪 Vectores de Ataque: Fases de la Infiltración

### 1. Bypass de Autenticación (Burlar el Login)
Si un auditor o atacante introduce en el campo de usuario el payload `' OR '1'='1`, la consulta interpretada por el servidor se transforma en:

```sql
SELECT * FROM usuarios WHERE username = '' OR '1'='1' AND password = '...';
```
Dado que la condición `'1'='1'` siempre es verdadera (True), la base de datos valida la fila y concede acceso de administrador sin necesidad de conocer una contraseña válida.

### 2. Extracción de Datos mediante UNION-Based SQLi
Una vez confirmada la inyección, podemos utilizar el operador `UNION` para fusionar los resultados de la consulta original con los de una consulta diseñada por nosotros:

* **Paso A: Determinar el número de columnas.**  
  Usamos `ORDER BY` de forma incremental hasta provocar un error en el servidor:
  `' ORDER BY 1-- -`, `' ORDER BY 5-- -`... Si falla en el 5, sabemos que la tabla tiene exactamente 4 columnas.
  
* **Paso B: Extraer nombres de tablas y esquemas.**  
  Inyectamos una consulta dirigida al diccionario de datos interno (Information Schema):
  ```sql
  ' UNION SELECT 1, table_name, 3, 4 FROM information_schema.tables WHERE table_schema=database()-- -
  ```

---

## 🛡️ Mitigación y Buenas Prácticas de Ingeniería
Parchear esta vulnerabilidad en producción no consiste en crear "listas negras" de palabras prohibidas. La única solución definitiva contra el SQLi es la **separación estricta entre el código y los datos**:

1. **Consultas Parametrizadas (PDO en PHP):**
   ```php
   \$stmt = \(pdo->prepare('SELECT * FROM usuarios WHERE username = :user AND password = :pass');\)stmt->execute(['user' => userInput, 'pass' => passwordInput]);
   ```
2. **Principio de Menor Privilegio:** Asegurar que el usuario de la base de datos que emplea la web no tenga permisos de superadministrador (`DBA`).

---

<blockquote>
<strong>📌 Nota del Analista:</strong> Automatizar estas pruebas con herramientas como SQLMap es excelente para ahorrar tiempo, pero comprender la matemática y la sintaxis detrás de cada comilla simple (`'`) es lo que separa a un ejecutor de herramientas de un verdadero Analista de Ciberseguridad. ¡Mantened vuestros entornos actualizados y vuestros inputs sanitizados!
</blockquote>
