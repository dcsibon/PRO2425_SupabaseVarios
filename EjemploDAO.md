# Ejemplo DAO con Supabase (JDBC)

Implementación completa de un DAO para Supabase accediendo directamente con **JDBC**, siguiendo el patrón DAO.

---

## 1. Interfaz base: `IStudentDao`

```kotlin
package dao

import model.Student

interface IStudentDao {
    fun getAll(): List<Student>
    fun add(name: String)
    fun update(id: Int, name: String)
    fun delete(id: Int)
}
```

---

## 2. Modelo: `Student.kt`

```kotlin
package model

data class Student(val id: Int = 0, val name: String)
```

---

## 3. Fuente de datos JDBC: `SupabaseDB.kt`

```kotlin
package db

import java.sql.Connection
import java.sql.DriverManager
import java.sql.SQLException

object SupabaseDB {
    private const val URL = "jdbc:postgresql://<TU-HOST>.supabase.co:5432/postgres"
    private const val USER = "<TU-USUARIO>"
    private const val PASSWORD = "<TU-PASSWORD>"

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

## 4. DAO: `StudentDAOSupabase.kt`

```kotlin
package dao

import db.SupabaseDB
import model.Student
import java.sql.Connection

class StudentDAOSupabase : IStudentDao {

    override fun getAll(): List<Student> {
        val students = mutableListOf<Student>()
        var conn: Connection? = null
        var stmt = null
        var rs = null

        try {
            conn = SupabaseDB.getConnection()
            stmt = conn.prepareStatement("SELECT * FROM students ORDER BY id")
            rs = stmt.executeQuery()
            while (rs.next()) {
                students.add(Student(rs.getInt("id"), rs.getString("name")))
            }
        } catch (e: Exception) {
            println("Error al recuperar estudiantes: ${e.message}")
        } finally {
            try { rs?.close() } catch (_: Exception) {}
            try { stmt?.close() } catch (_: Exception) {}
            SupabaseDB.closeConnection(conn)
        }
        return students
    }

    override fun add(name: String) {
        var conn: Connection? = null
        var stmt = null
        try {
            conn = SupabaseDB.getConnection()
            stmt = conn.prepareStatement("INSERT INTO students (name) VALUES (?)")
            stmt.setString(1, name)
            stmt.executeUpdate()
        } catch (e: Exception) {
            println("Error al añadir estudiante: ${e.message}")
        } finally {
            try { stmt?.close() } catch (_: Exception) {}
            SupabaseDB.closeConnection(conn)
        }
    }

    override fun update(id: Int, name: String) {
        var conn: Connection? = null
        var stmt = null
        try {
            conn = SupabaseDB.getConnection()
            stmt = conn.prepareStatement("UPDATE students SET name = ? WHERE id = ?")
            stmt.setString(1, name)
            stmt.setInt(2, id)
            stmt.executeUpdate()
        } catch (e: Exception) {
            println("Error al actualizar estudiante: ${e.message}")
        } finally {
            try { stmt?.close() } catch (_: Exception) {}
            SupabaseDB.closeConnection(conn)
        }
    }

    override fun delete(id: Int) {
        var conn: Connection? = null
        var stmt = null
        try {
            conn = SupabaseDB.getConnection()
            stmt = conn.prepareStatement("DELETE FROM students WHERE id = ?")
            stmt.setInt(1, id)
            stmt.executeUpdate()
        } catch (e: Exception) {
            println("Error al eliminar estudiante: ${e.message}")
        } finally {
            try { stmt?.close() } catch (_: Exception) {}
            SupabaseDB.closeConnection(conn)
        }
    }

    // Extra opcional: función para saber si un nombre ya existe
    fun existsByName(name: String): Boolean {
        var conn: Connection? = null
        var stmt = null
        var rs = null
        return try {
            conn = SupabaseDB.getConnection()
            stmt = conn.prepareStatement("SELECT COUNT(*) FROM students WHERE name = ?")
            stmt.setString(1, name)
            rs = stmt.executeQuery()
            rs.next() && rs.getInt(1) > 0
        } catch (e: Exception) {
            println("Error al verificar existencia: ${e.message}")
            false
        } finally {
            try { rs?.close() } catch (_: Exception) {}
            try { stmt?.close() } catch (_: Exception) {}
            SupabaseDB.closeConnection(conn)
        }
    }
}
```

---

## Uso de ejemplo

```kotlin
fun main() {
    val dao = StudentDAOSupabase()
    dao.add("Mario")
    dao.getAll().forEach { println("${it.id}: ${it.name}") }
    println("¿Existe Diego? ${dao.existsByName("Diego")}")
}
```

Vamos a cerrar el círculo incorporando:

1. El **servicio `StudentService`** que usa `IStudentDao`
2. La **interfaz `IConsoleUI`** para entrada/salida desacoplada
3. La implementación concreta `Consola`
4. Una clase `StudentsManager` que usa todo (el controlador de consola)
5. Una versión funcional de `Main.kt`

Esto crea una arquitectura limpia, modular y con separación de responsabilidades (SRP y DIP en SOLID).

---

## 1. `IStudentService.kt`

```kotlin
package service

import model.Student

interface IStudentService {
    fun listAll(): List<Student>
    fun addStudent(name: String)
    fun updateStudent(id: Int, name: String)
    fun deleteStudent(id: Int)
}
```

---

## 2. `StudentService.kt`

```kotlin
package service

import dao.IStudentDao
import model.Student

class StudentService(private val dao: IStudentDao) : IStudentService {

    override fun listAll(): List<Student> = dao.getAll()

    override fun addStudent(name: String) {
        require(name.isNotBlank()) { "El nombre no puede estar vacío." }
        dao.add(name.trim())
    }

    override fun updateStudent(id: Int, name: String) {
        require(id > 0) { "ID inválido." }
        require(name.isNotBlank()) { "El nombre no puede estar vacío." }
        dao.update(id, name.trim())
    }

    override fun deleteStudent(id: Int) {
        require(id > 0) { "ID inválido." }
        dao.delete(id)
    }
}
```

---

## 3. `IConsoleUI.kt`

```kotlin
package ui

interface IConsoleUI {
    fun mostrarTexto(texto: String)
    fun leerTexto(prompt: String = ""): String
    fun mostrarError(mensaje: String)
    fun mostrarLineaVacia()
}
```

---

## 4. `Consola.kt`

```kotlin
package ui

class Consola : IConsoleUI {
    override fun mostrarTexto(texto: String) {
        println(texto)
    }

    override fun leerTexto(prompt: String): String {
        if (prompt.isNotBlank()) print(prompt)
        return readLine() ?: ""
    }

    override fun mostrarError(mensaje: String) {
        println("⚠$mensaje")
    }

    override fun mostrarLineaVacia() {
        println()
    }
}
```

---

## 5. `StudentsManager.kt` (el controlador principal)

```kotlin
package app

import service.IStudentService
import ui.IConsoleUI

class StudentsManager(
    private val service: IStudentService,
    private val ui: IConsoleUI
) {
    private var running = true

    fun menu() {
        while (running) {
            ui.mostrarTexto("""
                === MENÚ ===
                1. Mostrar estudiantes
                2. Agregar estudiante
                3. Editar estudiante
                4. Eliminar estudiante
                5. Salir
            """.trimIndent())

            when (ui.leerTexto("Elige una opción: ")) {
                "1" -> mostrar()
                "2" -> agregar()
                "3" -> editar()
                "4" -> eliminar()
                "5" -> salir()
                else -> ui.mostrarError("Opción no válida.")
            }

            ui.mostrarLineaVacia()
        }
    }

    private fun mostrar() {
        val estudiantes = service.listAll()
        if (estudiantes.isEmpty()) {
            ui.mostrarTexto("No hay estudiantes.")
        } else {
            estudiantes.forEach { ui.mostrarTexto("ID: ${it.id} - Nombre: ${it.name}") }
        }
    }

    private fun agregar() {
        val nombre = ui.leerTexto("Introduce el nombre: ")
        try {
            service.addStudent(nombre)
            ui.mostrarTexto("Estudiante añadido.")
        } catch (e: Exception) {
            ui.mostrarError(e.message ?: "Error al añadir.")
        }
    }

    private fun editar() {
        val id = ui.leerTexto("ID a editar: ").toIntOrNull()
        if (id != null) {
            val nuevo = ui.leerTexto("Nuevo nombre: ")
            try {
                service.updateStudent(id, nuevo)
                ui.mostrarTexto("Estudiante actualizado.")
            } catch (e: Exception) {
                ui.mostrarError(e.message ?: "Error al editar.")
            }
        } else {
            ui.mostrarError("ID no válido.")
        }
    }

    private fun eliminar() {
        val id = ui.leerTexto("ID a eliminar: ").toIntOrNull()
        if (id != null) {
            try {
                service.deleteStudent(id)
                ui.mostrarTexto("Estudiante eliminado.")
            } catch (e: Exception) {
                ui.mostrarError(e.message ?: "Error al eliminar.")
            }
        } else {
            ui.mostrarError("ID no válido.")
        }
    }

    private fun salir() {
        ui.mostrarTexto("¡Hasta luego!")
        running = false
    }
}
```

---

## 6. `Main.kt`

```kotlin
import app.StudentsManager
import dao.StudentDAOSupabase
import service.StudentService
import ui.Consola

fun main() {
    val dao = StudentDAOSupabase()
    val service = StudentService(dao)
    val ui = Consola()
    val manager = StudentsManager(service, ui)
    manager.menu()
}
```

---

## Resultado:

* DAO desacoplado de la lógica de negocio
* Servicio de negocio con validaciones y control de errores
* Interfaz de usuario de consola desacoplada
* `Main.kt` claro y limpio

---

## Estructura de carpetas recomendada

Una posible estructura de carpetas **clara, modular y adecuada para un proyecto académico** con patrón DAO, servicios y consola desacoplada:

```
src/
└── main/
    └── kotlin/
        ├── app/                  # Lógica de control de la app
        │   └── StudentsManager.kt
        │
        ├── dao/                  # Acceso a datos
        │   ├── IStudentDao.kt
        │   └── StudentDAOSupabase.kt
        │
        ├── db/                   # Configuración de conexiones a la BD
        │   └── SupabaseDB.kt
        │
        ├── model/                # Modelos de datos
        │   └── Student.kt
        │
        ├── service/              # Lógica de negocio
        │   ├── IStudentService.kt
        │   └── StudentService.kt
        │
        ├── ui/                   # Interfaz de entrada/salida (I/O)
        │   ├── IConsoleUI.kt
        │   └── Consola.kt
        │
        └── Main.kt               # Punto de entrada
```

---

## ¿Por qué esta organización?

| Carpeta   | Contenido                                                | Justificación                                  |
| --------- | -------------------------------------------------------- | ---------------------------------------------- |
| `app`     | Orquestadores o controladores de flujo                   | Separa la lógica del programa principal        |
| `dao`     | Clases de acceso a datos (Supabase/PostgreSQL, etc.)     | Aplicación clara del patrón DAO                |
| `db`      | Objetos de configuración de conexión                     | Facilita el cambio de base de datos            |
| `model`   | Data classes o DTOs                                      | Responsabilidad única: estructura de datos     |
| `service` | Reglas de negocio, validaciones, operaciones agrupadas   | Separa la lógica de negocio del almacenamiento |
| `ui`      | Entrada y salida del usuario (consola, futura GUI, etc.) | Permite desacoplar la interfaz                 |
| `Main.kt` | Inicia e interconecta los componentes                    | Mantiene el `main()` limpio                    |

---

## Extras opcionales para escalar el proyecto

* `utils/` → para utilidades comunes si las necesitas
* `test/` → si decides hacer pruebas unitarias
* `resources/` → si usas ficheros `.properties`, scripts SQL, etc.

---

## `build.gradle.kts`

Archivo `build.gradle.kts` completo y limpio, listo para un **proyecto académico en consola con acceso a Supabase vía JDBC**:

```kotlin
plugins {
    kotlin("jvm") version "1.9.22"
    application
}

group = "es.prog2425"
version = "1.0-SNAPSHOT"

repositories {
    mavenCentral()
}

dependencies {
    // Driver JDBC de PostgreSQL para Supabase
    implementation("org.postgresql:postgresql:42.7.3")

    // Logging simple (SLF4J)
    implementation("org.slf4j:slf4j-simple:2.0.12")
}

application {
    mainClass.set("MainKt")  // Asegúrate de que tu archivo Main.kt no tenga paquete
}
```

---

## Estructura recomendada en IntelliJ

```
.
├── build.gradle.kts
├── settings.gradle.kts
└── src
    └── main
        ├── kotlin
        │   ├── Main.kt
        │   ├── app/
        │   ├── dao/
        │   ├── db/
        │   ├── model/
        │   ├── service/
        │   └── ui/
        └── resources/
```

---

## Notas importantes

* El `mainClass.set("MainKt")` asume que `Main.kt` **está fuera de paquetes**. Si lo metes en un paquete (`es.prog2425` por ejemplo), deberías cambiarlo a:

  ```kotlin
  mainClass.set("es.prog2425.MainKt")
  ```

* Si usas otros paquetes para organizar tu proyecto, asegúrate de ajustar los `package` en la parte superior de los archivos `.kt`.
