# PRO2425_SupabaseVarios

## Consideraciones de seguridad

Acceder por JDBC **requiere exponer el host, usuario y contraseña del proyecto de Supabase**, lo cual **no es seguro para entornos públicos**. Es válido en un entorno **de backend o educativo controlado**.

---

## 1. Añade esta dependencia al `build.gradle.kts`:

```kotlin
implementation("org.postgresql:postgresql:42.7.3")
```

---

## 2. Crea el `object SupabaseDataSource` con JDBC

```kotlin
import java.sql.Connection
import java.sql.DriverManager
import java.sql.SQLException

object SupabaseDataSource {
    private const val URL = "jdbc:postgresql://<tu-host>.supabase.co:5432/postgres"
    private const val USER = "<tu-usuario>"
    private const val PASSWORD = "<tu-password>"

    init {
        try {
            Class.forName("org.postgresql.Driver")
        } catch (e: ClassNotFoundException) {
            throw IllegalStateException("No se pudo cargar el driver de PostgreSQL", e)
        }
    }

    fun getConnection(): Connection {
        return try {
            DriverManager.getConnection(URL, USER, PASSWORD)
        } catch (e: SQLException) {
            throw IllegalStateException("Error al conectar a Supabase por JDBC: ${e.message}", e)
        }
    }

    fun closeConnection(conn: Connection?) {
        try {
            conn?.close()
        } catch (e: SQLException) {
            println("Error al cerrar la conexión: ${e.message}")
        }
    }
}
```

---

## 3. Función `existeStudent(nombre: String): Boolean`

```kotlin
fun existeStudent(nombre: String): Boolean {
    var conn: Connection? = null
    var stmt = null
    var rs = null

    return try {
        conn = SupabaseDataSource.getConnection()
        stmt = conn.prepareStatement("SELECT COUNT(*) FROM students WHERE name = ?")
        stmt.setString(1, nombre)
        rs = stmt.executeQuery()
        rs.next() && rs.getInt(1) > 0
    } catch (e: Exception) {
        println("Error al verificar existencia: ${e.message}")
        false
    } finally {
        try { rs?.close() } catch (_: Exception) {}
        try { stmt?.close() } catch (_: Exception) {}
        SupabaseDataSource.closeConnection(conn)
    }
}
```

---

## 4. Crear tabla en Supabase (si no existe):

```sql
create table students (
    id serial primary key,
    name text not null
);
```

---

## Ejemplo de uso en `main`:

```kotlin
fun main() {
    val nombre = "Diego"
    val existe = existeStudent(nombre)
    println("¿Existe '$nombre'? ${if (existe) "Sí" else "No"}")
}
```

---

## Resultado

Tienes una solución **100% JDBC**, que:

* Es compatible con tu enfoque académico
* Usa `DriverManager`
* Accede directamente a Supabase como a una base PostgreSQL estándar

