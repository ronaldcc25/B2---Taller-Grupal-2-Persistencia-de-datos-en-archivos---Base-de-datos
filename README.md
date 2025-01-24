# B2---Taller-Grupal-2-Persistencia-de-datos-en-archivos---Base-de-datos

Informacion agregada en un archivo CSV: 

    nombre,edad,calificacion,genero
    Andrés,10,20,M
    Ana,11,19,F
    Luis,9,18,M
    Cecilia,9,18,F
    Katy,11,15,F
    Jorge,8,17,M
    Rosario,11,18,F
    Nieves,10,20,F
    Pablo,9,19,M
    Daniel,10,20,M


## DATA BASE
```sql
CREATE DATABASE estuddiantes;
USE esrudiantes;
CREATE TABLE estudiantes (
    nombre VARCHAR(50),
    edad INT,
    calificacion INT,
    genero CHAR(1)
);
```

## Application.config
```scala
db {
  driver = "com.mysql.cj.jdbc.Driver"
  url = "jdbc:mysql://localhost:3306/estudiantes_db"
  user = "root"
  password = "UTPL"
}
```
## Codigo
**Main**

```Scala
import cats.effect.{IO, IOApp}
import kantan.csv._
import kantan.csv.ops._
import kantan.csv.generic._
import java.io.File
import models.Estudiante
import dao.EstudianteDAO
import doobie._
import doobie.implicits._
import cats.effect.{IO, IOApp}
import cats.implicits._
import models._
import config.Database

// Extiende de IOApp.Simple para manejar efectos IO y recursos de forma segura
object Main extends IOApp.Simple {

  val path2DataFile2 = "src/main/resources/data/estudiantes.csv"

  val dataSource = new File(path2DataFile2)
    .readCsv[List, Estudiante](rfc.withHeader.withCellSeparator(','))

  val estudiantes = dataSource.collect {
    case Right(estudiante) => estudiante
  }

  // Secuencia de operaciones IO usando for-comprehension
  def run: IO[Unit] = for {
    result <- EstudianteDAO.insertAll(estudiantes) // Inserta datos y extrae resultado con <-
    _ <- IO.println(s"Registros insertados: ${result.size}")  // Imprime cantidad
    allEstudiantes <- Database.transactor.use(xa => EstudianteDAO.obtenerTodos.transact(xa))
    _ <- IO.println("Lista de estudiantes:")
    _ <- allEstudiantes.traverse(e => IO.println(e))
  } yield ()  // Completa la operación

}
```

**Case class**
```Scala
case class Estudiante(
                        nombre: String,
                        edad: Int,
                        calificacion: Int,
                        genero: String
                      )
```

**EstudianteDAO**
```Scala
package dao

import doobie._
import doobie.implicits._
import cats.effect.IO
import cats.implicits._

import models.Estudiante
import config.Database

object EstudianteDAO {
  def insert(estudiante: Estudiante): ConnectionIO[Int] = {
    sql"""
     INSERT INTO estudiantes (nombre, edad, calificacion, genero)
     VALUES (
       ${estudiante.nombre},
       ${estudiante.edad},
       ${estudiante.calificacion},
       ${estudiante.genero}
     )
   """.update.run
  }

  def insertAll(estudiantes: List[Estudiante]): IO[List[Int]] = {
    Database.transactor.use { xa =>
      estudiantes.traverse(t => insert(t).transact(xa))
    }
  }

  def obtenerTodos: ConnectionIO[List[(String, Int, Int, String)]] =
    sql"SELECT nombre, edad, calificacion, genero FROM estudiantes".query[(String, Int, Int, String)].to[List]

}
```

**Database**
```Scala
package config

import cats.effect.{IO, Resource}
import com.typesafe.config.ConfigFactory
import doobie.hikari.HikariTransactor

import scala.concurrent.ExecutionContext

object Database {
  private val connectEC: ExecutionContext = ExecutionContext.global

  def transactor: Resource[IO, HikariTransactor[IO]] = {
    val config = ConfigFactory.load().getConfig("db")
    HikariTransactor.newHikariTransactor[
      IO
    ](
      config.getString("driver"),
      config.getString("url"),
      config.getString("user"),
      config.getString("password"),
      connectEC // ExecutionContext requerido para Doobie
    )
  }
}
```
