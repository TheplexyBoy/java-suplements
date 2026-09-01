# Guía de estudio completa — TallerExpress

Este documento explica **todo** el código del proyecto, capa por capa, línea por línea donde importa, más los conceptos que te pueden preguntar en la sustentación. Está pensado para leerlo de arriba a abajo una vez, y después usarlo como referencia rápida.

---

## PARTE 0 — Los conceptos grandes (esto es lo primero que debes poder explicar)

### ¿Qué es una arquitectura por capas y por qué la usamos?

Es dividir el programa en "pisos", donde cada piso solo habla con el piso de al lado, nunca se salta ninguno:

```
presentation (JOptionPane)
      ↓ ↑
  controller
      ↓ ↑
   service   (reglas de negocio)
      ↓ ↑
     dao     (habla con la base de datos)
      ↓ ↑
  base de datos (H2)
```

**¿Por qué no hacer todo junto?** Porque si mañana cambias de JOptionPane a una interfaz web, solo tienes que rehacer `presentation` — `service` y `dao` no se tocan. Si cambias de H2 a MySQL, solo tocas `dao`. Cada capa se puede cambiar sin romper las demás. Eso se llama **bajo acoplamiento**.

**Regla de oro que seguimos todo el proyecto:** una pantalla (`presentation`) nunca llama directamente a un DAO. Siempre pasa por `controller` → `service` → `dao`. Si te preguntan "¿por qué existe el controller si solo delega al service?", la respuesta es: para que el día de mañana, si agregas otra interfaz (una web, una API), reuses el mismo `service` con otro `controller`, sin duplicar reglas de negocio.

### Los 4 pilares de la Programación Orientada a Objetos (POO) — dónde están en el código

1. **Encapsulamiento**: todos los atributos de nuestras clases (`model`) son `private`, y solo se accede a ellos por `getters`/`setters`. Nadie desde afuera puede poner un `stockDisponible` negativo directamente tocando el atributo — tiene que pasar por un método, y esos métodos (en `service`) validan.

2. **Abstracción**: las interfaces (`ClienteDAO`, `RepuestoDAO`, etc.) son abstracción pura — dicen **qué** se puede hacer ("crear un cliente") sin decir **cómo** ("con esta consulta SQL exacta"). El resto del programa programa contra la interfaz, no contra la implementación.

3. **Herencia**: en este proyecto la herencia es débil a propósito — no armamos una superclase `Entidad` porque decidimos mantenerlo simple. Donde sí hay una relación fuerte entre clases es la **composición**: `Vehiculo` tiene un `Cliente` adentro, `Orden` tiene un `Vehiculo` adentro, `DetalleOrden` tiene un `Repuesto` adentro. Si te preguntan por herencia y no la ven, explica que se usó composición (relación "tiene un") en vez de herencia (relación "es un"), porque nuestras clases no son variantes unas de otras.

4. **Polimorfismo**: aparece en el patrón **Decorador** de `Usuario` (`CreadorUsuario`, `CreadorUsuarioBase`, `CreadorUsuarioConValoresPorDefecto`) — las dos clases implementan la misma interfaz `CreadorUsuario`, y el programa las trata indistintamente a través de esa interfaz, sin saber (ni importarle) cuál de las dos está usando en cada momento.

### Patrones de diseño que usamos

- **DAO (Data Access Object)**: separa "cómo se guardan los datos" del resto del programa. Cada entidad tiene una interfaz (el contrato) y una implementación (`...Impl`, el que realmente escribe SQL).
- **Service Layer**: concentra las reglas de negocio (validaciones) en un solo lugar, para no repetirlas en cada pantalla.
- **Decorador**: envuelve un objeto (o comportamiento) para agregarle funcionalidad sin modificar el original. Lo usamos para que la creación de un `Usuario` tenga valores por defecto (rol, estado, fecha) sin tocar la lógica base de "crear usuario".
- **MVC-ish (Modelo - Vista - Controlador)**: `model` = Modelo, `presentation` = Vista, `controller` = Controlador. No es MVC puro porque tenemos `service` y `dao` de más, pero la idea de raíz es la misma.

### JDBC — lo que hay que saber sí o sí

- **`Connection`**: representa la conexión abierta con la base de datos.
- **`PreparedStatement`**: una consulta SQL con `?` en vez de valores, que después se rellenan con `.setString()`, `.setInt()`, etc. **Se usa siempre en vez de concatenar Strings** porque previene inyección SQL (alguien no puede meter código SQL malicioso en un campo de texto).
- **`ResultSet`**: el resultado de un `SELECT`, se recorre con `.next()` fila por fila.
- **`try-with-resources`** (`try (Connection con = ...) { }`): cierra automáticamente la conexión al salir del bloque, incluso si hubo una excepción. Evita fugas de conexiones abiertas.
- **Transacciones** (`setAutoCommit(false)`, `commit()`, `rollback()`): agrupan varias operaciones para que sean "todo o nada". Las usamos al registrar una orden de servicio, porque implica dos pasos (guardar la orden + descontar stock) que deben ocurrir juntos o no ocurrir.

### Excepciones personalizadas — por qué existen dos y no una sola

- **`ValidacionException`**: se lanza cuando el usuario intenta hacer algo que viola una regla de negocio (ej: registrar un repuesto con código duplicado). Es un error "esperado", del usuario.
- **`PersistenciaException`**: se lanza cuando algo falla hablando con la base de datos (ej: se cae la conexión). Es un error técnico, no culpa del usuario.

Separarlas permite que la pantalla reaccione distinto: a una `ValidacionException` le muestra el mensaje tal cual ("ya existe ese código"), a una `PersistenciaException` le puede mostrar un mensaje más genérico ("hubo un problema técnico, intenta de nuevo").

---

## PARTE 1 — `model` (las clases de datos)

Todas siguen el mismo patrón: atributos `private`, un constructor vacío, un constructor completo, getters/setters, y un `toString()` para mostrarlas legible. Esto es **encapsulamiento**: nadie modifica un atributo sin pasar por un método.

### Cliente.java

```java
package hub3.tallerexpress.model;

import java.time.LocalDateTime;

public class Cliente {

    private int id;                    // clave primaria, autogenerada por la BD
    private String nombre;
    private String documento;          // cédula/NIT — UNIQUE en la tabla
    private String telefono;
    private String email;
    private String direccion;
    private boolean activo;            // regla de negocio "cliente activo" se basa en este campo
    private LocalDateTime createdAt;   // fecha de registro, la pone la BD o el service

    public Cliente() {
        // constructor vacío: lo usa el DAO antes de llenar los campos con setters
    }

    public Cliente(int id, String nombre, String documento, String telefono,
                   String email, String direccion, boolean activo, LocalDateTime createdAt) {
        // constructor completo: lo usa el DAO para "armar" un Cliente leído desde la BD
        this.id = id;
        this.nombre = nombre;
        this.documento = documento;
        this.telefono = telefono;
        this.email = email;
        this.direccion = direccion;
        this.activo = activo;
        this.createdAt = createdAt;
    }

    // getters: devuelven el valor de un atributo privado (única forma de leerlo desde afuera)
    public int getId() { return id; }
    public String getNombre() { return nombre; }
    public String getDocumento() { return documento; }
    public String getTelefono() { return telefono; }
    public String getEmail() { return email; }
    public String getDireccion() { return direccion; }
    public boolean isActivo() { return activo; }   // "is" en vez de "get" porque es boolean (convención Java)
    public LocalDateTime getCreatedAt() { return createdAt; }

    // setters: única forma de cambiar un atributo privado desde afuera
    public void setId(int id) { this.id = id; }
    public void setNombre(String nombre) { this.nombre = nombre; }
    public void setDocumento(String documento) { this.documento = documento; }
    public void setTelefono(String telefono) { this.telefono = telefono; }
    public void setEmail(String email) { this.email = email; }
    public void setDireccion(String direccion) { this.direccion = direccion; }
    public void setActivo(boolean activo) { this.activo = activo; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }

    @Override
    public String toString() {
        // se sobreescribe el toString() heredado de Object (que por defecto muestra
        // algo como "Cliente@1a2b3c") para que se vea legible al imprimirlo
        String mostrarNombre = (this.nombre != null && !this.nombre.trim().isEmpty())
                ? this.nombre : "Cliente inexistente";
        String textoEstado = (this.activo) ? "Activo " : "Inactivo";
        return "Nombre: " + mostrarNombre + " | Estado: " + textoEstado;
    }
}
```
**Concepto clave si preguntan por `toString()`:** es un método que TODA clase en Java hereda de `Object` (la clase raíz de la que todo hereda, aunque no lo escribas). Sobreescribirlo (`@Override`) es **polimorfismo**: `System.out.println(cliente)` internamente llama a `cliente.toString()`, y como nosotros lo redefinimos, sale nuestro texto y no el de `Object`.

### Vehiculo.java

Igual estructura, con una diferencia importante: **tiene un `Cliente` adentro**, no solo un `clienteId` suelto.

```java
private Cliente cliente; // composición: un Vehiculo "tiene un" Cliente
```
Esto nos permite escribir `vehiculo.getCliente().getNombre()` directamente, sin tener que ir a buscar el cliente aparte cada vez. La contrapartida es que el DAO (`VehiculoDAOImpl`) tiene que encargarse de "rellenar" ese objeto `Cliente` completo cuando lee de la base de datos (ver la sección de DAO).

### Repuesto.java

Nada especial en la estructura, pero dos campos merecen explicación:
- `stockTotal` vs `stockDisponible`: `stockTotal` es cuánto se ha comprado en total históricamente; `stockDisponible` es cuánto queda hoy. Cuando se usa un repuesto en una orden, se descuenta `stockDisponible`, nunca `stockTotal`.
- `isActivo` (con ese nombre raro para el getter `isIsActivo()`): viene de que el campo se llama `isActivo` (con el "is" ya adentro del nombre), y Java, al generar el getter de un boolean, le antepone otro "is" — por eso queda `isIsActivo()`. No es un error, es una consecuencia de cómo se nombró el atributo.

### Usuario.java

```java
private String rol; // "ADMIN" o "RECEPCIONISTA" — es un String simple, no un enum
```
**Si preguntan "¿por qué no usaron un `enum` para el rol?"**: la respuesta honesta es que se mantuvo simple a propósito (un `enum Rol { ADMIN, RECEPCIONISTA }` sería más seguro porque el compilador no dejaría escribir un rol inválido), pero un `String` es más rápido de escribir y sigue funcionando porque la validación real ocurre en la base de datos con la restricción `CHECK (role IN ('ADMIN','RECEPCIONISTA'))`.

### DetalleOrden.java

Esta es la clase "puente" entre una `Orden` y un `Repuesto` — representa una línea "usé este repuesto, esta cantidad, a este precio".

```java
public double getSubtotal() {
    return precioUnitario * cantidad;
}
```
**Por qué guardamos `precioUnitario` en el detalle y no lo leemos siempre del repuesto:** porque el precio de un repuesto puede cambiar con el tiempo. Si el repuesto valía $50 cuando se usó en la orden, y después el proveedor sube el precio a $60, la orden ya cerrada debe seguir mostrando $50 — congelamos el precio en el momento en que se usó.

### Orden.java

```java
private List<DetalleOrden> repuestosUtilizados;

public void agregarRepuesto(DetalleOrden detalle) {
    this.repuestosUtilizados.add(detalle);
}

public double calcularCostoTotal() {
    double total = 0;
    for (DetalleOrden d : repuestosUtilizados) {
        total += d.getSubtotal();
    }
    this.costoTotal = total;
    return total;
}
```
**Este método es el corazón del requisito "calcular el costo total de la reparación".** Recorre cada línea de repuesto usado, suma sus subtotales, y guarda el resultado en `costoTotal`. Se llama justo antes de guardar la orden (en `OrdenService.registrarOrden()`).

```java
private String estado; // RECIBIDA, EN_DIAGNOSTICO, EN_REPARACION, FINALIZADA, ENTREGADA
```
Este es el ciclo de vida de una orden — el requisito "permitir actualizar el estado de la orden" se refiere a mover un pedido por estos 5 valores.

---

## PARTE 2 — `exception` (excepciones personalizadas)

```java
package hub3.tallerexpress.exception;

public class ValidacionException extends RuntimeException {
    public ValidacionException(String mensaje) {
        super(mensaje);
    }
}
```
```java
package hub3.tallerexpress.exception;

public class PersistenciaException extends RuntimeException {
    public PersistenciaException(String mensaje, Throwable causa) {
        super(mensaje, causa);
    }
    public PersistenciaException(String mensaje) {
        super(mensaje);
    }
}
```

**Conceptos clave:**
- `extends RuntimeException`: en Java hay dos tipos de excepciones — las **checked** (obligan a poner `throws` o `try/catch`, como `SQLException`) y las **unchecked** (`RuntimeException` y sus hijas), que no obligan a nada. Elegimos `RuntimeException` para no llenar todo el código de `throws` en cada método.
- **`super(mensaje)`**: llama al constructor de la clase padre (`RuntimeException`), que es quien realmente guarda el mensaje de error.
- **`Throwable causa`** (solo en `PersistenciaException`): permite "encadenar" la excepción original (por ejemplo, la `SQLException` real de la base de datos) dentro de nuestra excepción personalizada, sin perder esa información. Así, cuando algo falla, puedes ver tanto tu mensaje ("Error al guardar el cliente") como la causa técnica real debajo (`Caused by: ...`) — esto es justo lo que viste en la consola cuando debuggeamos el error de la tabla vacía.
- **¿Por qué dos clases y no una genérica `MiExcepcion`?** Porque así el `catch` puede reaccionar distinto según el tipo: `catch (ValidacionException e)` → error del usuario, se lo muestras tal cual. `catch (PersistenciaException e)` → error técnico.

---

## PARTE 3 — `config` (conexión a la base de datos)

```java
package hub3.tallerexpress.config;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class ConexionDB {

    private static final String URL = "jdbc:h2:./tallerexpress_db";
    private static final String USER = "sa";
    private static final String PASSWORD = "";

    public static Connection obtenerConexion() {
        try {
            return DriverManager.getConnection(URL, USER, PASSWORD);
        } catch (SQLException e) {
            System.out.println("Error fatal al conectar con H2: " + e.getMessage());
            return null;
        }
    }
}
```

- **`static final`**: `static` significa que esas constantes pertenecen a la clase, no a un objeto — no necesitas hacer `new ConexionDB()` para usarlas. `final` significa que no se pueden reasignar después.
- **`DriverManager.getConnection(...)`**: el punto de entrada estándar de JDBC — le das una URL con el formato `jdbc:<motor>:<ubicación>`, usuario y contraseña, y te devuelve una `Connection` abierta.
- **`jdbc:h2:./tallerexpress_db`**: le dice a Java "usa el driver de H2, y el archivo de base de datos está en la carpeta actual, se llama `tallerexpress_db`".
- **Método `static`**: `obtenerConexion()` también es `static`, por eso en todo el proyecto lo llamamos como `ConexionDB.obtenerConexion()`, sin crear un objeto `ConexionDB` primero.

---

## PARTE 4 — `dao` (acceso a datos)

Explico un DAO completo a fondo (`RepuestoDAO`/`RepuestoDAOImpl`) y después señalo solo lo que cambia en los demás, para no repetir.

### RepuestoDAO.java (la interfaz — el contrato)

```java
public interface RepuestoDAO {
    void crear(Repuesto repuesto) throws PersistenciaException;
    Repuesto obtenerPorId(int id) throws PersistenciaException;
    Repuesto obtenerPorCodigo(String codigoReferencia) throws PersistenciaException;
    List<Repuesto> obtenerTodos() throws PersistenciaException;
    List<Repuesto> filtrar(String categoria, String proveedor) throws PersistenciaException;
    boolean existeCodigo(String codigoReferencia) throws PersistenciaException;
    void actualizar(Repuesto repuesto) throws PersistenciaException;
    void actualizarStock(int id, int nuevoStockDisponible) throws PersistenciaException;
    void cambiarEstado(int id, boolean nuevoEstado) throws PersistenciaException;
}
```
Una interfaz **no tiene código**, solo firmas de métodos. Es un contrato: "cualquier clase que implemente `RepuestoDAO` DEBE tener estos 9 métodos". Programamos contra esta interfaz en el resto del proyecto (`service` usa `RepuestoDAO`, no `RepuestoDAOImpl` directamente) — así, si mañana quieres una implementación distinta (por ejemplo una que hable con MySQL en vez de H2), solo escribes una nueva clase que implemente esta misma interfaz, y nada más en el proyecto se entera del cambio.

### RepuestoDAOImpl.java (la implementación — el que sí tiene SQL)

```java
public class RepuestoDAOImpl implements RepuestoDAO {

    @Override
    public void crear(Repuesto repuesto) throws PersistenciaException {
        String sql = "INSERT INTO repuestos (codigo_referencia, nombre, categoria, proveedor, "
                   + "stock_total, stock_disponible, precio_unitario, is_activo) VALUES (?, ?, ?, ?, ?, ?, ?, ?)";
        // el SQL tiene 8 signos de interrogación: son "huecos" que se llenan después,
        // NUNCA se concatenan los valores directamente en el String (eso sería
        // vulnerable a inyección SQL)

        try (Connection con = ConexionDB.obtenerConexion();
             PreparedStatement ps = con.prepareStatement(sql)) {
            // try-with-resources: al terminar el bloque {} (por éxito o por excepción),
            // Java cierra automáticamente con y ps, sin que tengamos que escribir un finally

            ps.setString(1, repuesto.getCodigoReferencia()); // llena el primer "?"
            ps.setString(2, repuesto.getNombre());           // el segundo "?"
            ps.setString(3, repuesto.getCategoria());
            ps.setString(4, repuesto.getProveedor());
            ps.setInt(5, repuesto.getStockTotal());
            ps.setInt(6, repuesto.getStockDisponible());
            ps.setDouble(7, repuesto.getPrecioUnitario());
            ps.setBoolean(8, repuesto.isIsActivo());
            ps.executeUpdate(); // ejecuta el INSERT; para SELECT se usa executeQuery()
        } catch (SQLException e) {
            // SQLException es "checked" (obliga try/catch); la envolvemos en nuestra
            // propia excepción para que el resto del programa no tenga que conocer
            // detalles de JDBC
            throw new PersistenciaException("Error al guardar el repuesto.", e);
        }
    }
```

- **`.setString(1, ...)`, `.setInt(5, ...)`**: el número es la POSICIÓN del `?` en el SQL (empezando en 1, no en 0). Si el tipo de dato no coincide con la columna (ej: `setString` en una columna `INT`), truena en tiempo de ejecución.
- **`executeUpdate()` vs `executeQuery()`**: `executeUpdate()` se usa para `INSERT`, `UPDATE`, `DELETE` — devuelve un `int` (cuántas filas afectó). `executeQuery()` se usa para `SELECT` — devuelve un `ResultSet`.

```java
    @Override
    public Repuesto obtenerPorId(int id) throws PersistenciaException {
        String sql = "SELECT * FROM repuestos WHERE id = ?";
        try (Connection con = ConexionDB.obtenerConexion(); PreparedStatement ps = con.prepareStatement(sql)) {
            ps.setInt(1, id);
            try (ResultSet rs = ps.executeQuery()) {
                // ResultSet: como un cursor que apunta ANTES de la primera fila.
                // rs.next() avanza una fila y devuelve true si había una; false si ya no hay más
                if (rs.next()) {
                    return extraerRepuesto(rs); // había un repuesto con ese id: lo armamos y devolvemos
                }
            }
        } catch (SQLException e) {
            throw new PersistenciaException("Error al obtener el repuesto.", e);
        }
        return null; // no existía ningún repuesto con ese id
    }
```

```java
    private Repuesto extraerRepuesto(ResultSet rs) throws SQLException {
        return new Repuesto(
                rs.getInt("id"),                          // lee la columna "id" como int
                rs.getString("codigo_referencia"),         // lee "codigo_referencia" como String
                rs.getString("nombre"),
                rs.getString("categoria"),
                rs.getString("proveedor"),
                rs.getInt("stock_total"),
                rs.getInt("stock_disponible"),
                rs.getDouble("precio_unitario"),
                rs.getBoolean("is_activo"),
                rs.getTimestamp("created_at").toLocalDateTime()
                // getTimestamp devuelve un java.sql.Timestamp (tipo antiguo de JDBC);
                // .toLocalDateTime() lo convierte al tipo moderno de Java que usa el resto del proyecto
        );
    }
```
Este método privado es un **helper**: convierte una fila cruda de la base de datos (`ResultSet`) en un objeto `Repuesto` de Java. Se reutiliza en `obtenerPorId`, `obtenerPorCodigo`, `obtenerTodos` y `filtrar` — evita repetir ese mapeo 4 veces.

```java
    @Override
    public List<Repuesto> filtrar(String categoria, String proveedor) throws PersistenciaException {
        List<Repuesto> lista = new ArrayList<>();
        StringBuilder sql = new StringBuilder("SELECT * FROM repuestos WHERE 1=1");
        // "WHERE 1=1" es un truco: es una condición siempre verdadera, para poder
        // ir agregando "AND columna = ?" con seguridad sin preocuparte de si es
        // el primer filtro o no
        if (categoria != null && !categoria.isBlank()) sql.append(" AND categoria = ?");
        if (proveedor != null && !proveedor.isBlank()) sql.append(" AND proveedor = ?");

        try (Connection con = ConexionDB.obtenerConexion(); PreparedStatement ps = con.prepareStatement(sql.toString())) {
            int i = 1; // posición del próximo "?" a rellenar (dinámico, porque no sabemos cuántos "?" hay hasta correr el código)
            if (categoria != null && !categoria.isBlank()) ps.setString(i++, categoria); // i++ usa i, LUEGO lo incrementa
            if (proveedor != null && !proveedor.isBlank()) ps.setString(i++, proveedor);
            ...
```
**`i++` (post-incremento) explicado porque suele preguntarse:** `ps.setString(i++, categoria)` es equivalente a hacer `ps.setString(i, categoria); i = i + 1;` en dos líneas, pero en una sola instrucción. Usa el valor actual de `i` primero, y DESPUÉS lo incrementa.

Los demás métodos (`actualizar`, `actualizarStock`, `cambiarEstado`, `existeCodigo`) siguen exactamente el mismo patrón: armar el SQL con `?`, abrir conexión con try-with-resources, rellenar con `.setX()`, ejecutar, capturar `SQLException` y envolverla en `PersistenciaException`.

### Lo que cambia en cada DAO respecto a este patrón base

**ClienteDAOImpl**: idéntico patrón. Su único método distinto es `eliminarFisico()`, que hace un `DELETE FROM clientes WHERE id = ?` — un borrado real (no solo desactivar).

**VehiculoDAOImpl**: su `extraerVehiculo()` es distinto porque `Vehiculo` tiene un `Cliente` adentro:
```java
private Vehiculo extraerVehiculo(ResultSet rs) throws SQLException {
    Cliente c = new ClienteDAOImpl().obtenerPorId(rs.getInt("cliente_id"));
    // en vez de solo poner el id del cliente, usamos OTRO DAO (ClienteDAOImpl)
    // para ir a buscar el cliente COMPLETO a la base de datos
    return new Vehiculo(rs.getInt("id"), c, rs.getString("placa"), ...);
}
```
Esto significa que cada vez que lees un vehículo, por dentro se hace una segunda consulta SQL (a `clientes`) para traer los datos del dueño. Es un poco menos eficiente que traer todo en un solo `JOIN`, pero mucho más simple de leer y mantener — un intercambio consciente de "simplicidad del código" vs "rendimiento óptimo".

**UsuarioDAOImpl**: tiene una particularidad — en Java, `Usuario.activo` es `boolean` (`true`/`false`), pero en la tabla `usuarios` la columna `estado` es texto (`'ACTIVO'`/`'INACTIVO'`). Por eso este DAO, y solo este, convierte ida y vuelta:
```java
ps.setString(5, usuario.isActivo() ? "ACTIVO" : "INACTIVO"); // boolean -> texto, al guardar
...
"ACTIVO".equals(rs.getString("estado")) // texto -> boolean, al leer
```
**`? :` es el operador ternario**: `condición ? valorSiVerdadero : valorSiFalso`. Es un `if/else` corto, en una sola línea, que además puede usarse como valor (no solo como sentencia).

**OrdenDAOImpl**: el más complejo, porque una orden tiene una LISTA de repuestos usados (tabla `orden_repuestos`), no un solo valor:

```java
@Override
public void guardar(Orden orden) {
    String sqlOrden = "INSERT INTO ordenes_servicio (...) VALUES (...)";
    String sqlDetalle = "INSERT INTO orden_repuestos (...) VALUES (...)";

    try (Connection con = ConexionDB.obtenerConexion()) {
        int ordenId;
        try (PreparedStatement stmt = con.prepareStatement(sqlOrden, Statement.RETURN_GENERATED_KEYS)) {
            // RETURN_GENERATED_KEYS: le dice a H2 "después de insertar, dame el id
            // autogenerado que le asignaste a esta fila" — lo necesitamos para poder
            // insertar después el detalle, que necesita saber a qué orden pertenece
            ...
            stmt.executeUpdate();
            try (ResultSet keys = stmt.getGeneratedKeys()) {
                keys.next();
                ordenId = keys.getInt(1); // la primera (y única) columna del resultado es el id nuevo
            }
        }

        try (PreparedStatement stmtDetalle = con.prepareStatement(sqlDetalle)) {
            for (DetalleOrden d : orden.getRepuestosUtilizados()) {
                stmtDetalle.setInt(1, ordenId);
                stmtDetalle.setInt(2, d.getRepuesto().getId());
                stmtDetalle.setInt(3, d.getCantidad());
                stmtDetalle.setDouble(4, d.getPrecioUnitario());
                stmtDetalle.addBatch(); // en vez de ejecutar cada INSERT uno por uno,
                                        // los "acumula" en un lote
            }
            stmtDetalle.executeBatch(); // ejecuta TODOS los INSERT acumulados de una sola vez
                                        // (más eficiente que uno por uno si hay varios repuestos)
        }
        orden.setId(ordenId);
    } catch (SQLException ex) {
        throw new PersistenciaException("Error al guardar la orden en la base de datos.", ex);
    }
}
```

**`buscarDetalles()`** hace el camino inverso: dado un `ordenId`, trae todas sus filas de `orden_repuestos`. Se usa en `buscarPorId()` y `listarTodos()` para reconstruir la lista completa de repuestos usados.

Además, `OrdenDAOImpl` tiene dos métodos especiales para transacciones (los explico a fondo en la sección de `service`, porque ahí es donde se orquestan):
```java
void guardarTransaccional(Connection con, Orden orden) throws SQLException;
```
Nota que este método **recibe una `Connection` como parámetro**, en vez de abrir una nueva con `ConexionDB.obtenerConexion()`. Esto es clave para que la transacción funcione — lo explico abajo.

---

## PARTE 5 — `service` (reglas de negocio)

### ClienteService.java

```java
public void registrarCliente(Cliente cliente) {
    if (cliente.getNombre() == null || cliente.getNombre().isBlank()) {
        throw new ValidacionException("El nombre del cliente es obligatorio.");
    }
    if (cliente.getDocumento() == null || cliente.getDocumento().isBlank()) {
        throw new ValidacionException("El documento del cliente es obligatorio.");
    }
    cliente.setActivo(true);
    cliente.setCreatedAt(LocalDateTime.now());
    clienteDAO.crear(cliente);
    LogHttp.log("POST", "/clientes", "Cliente creado: " + cliente.getDocumento());
}
```
- **`.isBlank()`**: método de `String` que devuelve `true` si el texto está vacío o son solo espacios (" " cuenta como "en blanco", a diferencia de `.isEmpty()` que solo detecta "").
- **`LocalDateTime.now()`**: la fecha y hora actual del sistema — así el `service`, no el usuario, decide cuándo se creó el registro (evita que alguien mande una fecha falsa desde la pantalla).
- **`LogHttp.log("POST", "/clientes", ...)`**: esto es la simulación de "trazas de llamadas HTTP" que pide el enunciado — no hay un servidor web real, pero el patrón de logging imita cómo se vería en una API REST (POST = crear, GET = leer, PATCH = actualizar).

```java
public boolean esClienteActivoValido(int clienteId) {
    Cliente c = clienteDAO.obtenerPorId(clienteId);
    return c != null && c.isActivo();
}
```
**`&&` es "cortocircuito"**: si `c != null` es `false`, Java ni siquiera evalúa `c.isActivo()` — porque ya sabe que el resultado final será `false` de todas formas. Esto es intencional y necesario aquí: si evaluara `c.isActivo()` sobre un `c` que es `null`, tronaría con `NullPointerException`.

### VehiculoService.java

```java
public void registrarVehiculo(Vehiculo vehiculo) {
    ...
    if (!clienteService.esClienteActivoValido(vehiculo.getCliente().getId())) {
        throw new ValidacionException("El cliente no existe o está inactivo.");
    }
    if (vehiculoDAO.existePlaca(vehiculo.getPlaca())) {
        throw new ValidacionException("Ya existe un vehículo registrado con esa placa.");
    }
    ...
}
```
Aquí un `VehiculoService` **usa a otro service** (`ClienteService`) para reutilizar la regla "cliente activo", en vez de volver a escribirla. Esto es evitar duplicar lógica (principio DRY: "Don't Repeat Yourself").

### RepuestoService.java

```java
public void descontarStock(int repuestoId, int cantidad) {
    Repuesto r = repuestoDAO.obtenerPorId(repuestoId);
    if (r == null) {
        throw new ValidacionException("El repuesto no existe.");
    }
    if (cantidad > r.getStockDisponible()) {
        throw new ValidacionException("Stock insuficiente para el repuesto " + r.getCodigoReferencia());
    }
    repuestoDAO.actualizarStock(repuestoId, r.getStockDisponible() - cantidad);
    LogHttp.log("PATCH", "/repuestos/" + repuestoId, "Stock actualizado");
}
```
Esta es la validación de negocio "stock mayor o igual a cero" en acción: antes de descontar, se comprueba que hay suficiente. **Ojo:** esta versión NO es la que usa `OrdenService` al registrar una orden — para eso existe una versión más segura dentro del propio `RepuestoDAOImpl.descontarStock(Connection, ...)`, que veremos abajo, y que hace la validación DENTRO de la misma transacción (evita una condición de carrera si dos usuarios registran órdenes al mismo tiempo).

### UsuarioService.java y el patrón Decorador — la parte más importante para explicar bien

```java
public interface CreadorUsuario {
    void crear(Usuario usuario);
}
```
El contrato: "cualquier cosa que sepa crear un usuario".

```java
public class CreadorUsuarioBase implements CreadorUsuario {
    private final UsuarioDAO usuarioDAO;

    public CreadorUsuarioBase(UsuarioDAO usuarioDAO) {
        this.usuarioDAO = usuarioDAO;
    }

    @Override
    public void crear(Usuario usuario) {
        if (usuario.getUsername() == null || usuario.getUsername().isBlank()) {
            throw new ValidacionException("El username es obligatorio.");
        }
        if (usuarioDAO.existeUsername(usuario.getUsername())) {
            throw new ValidacionException("Ya existe un usuario con ese username.");
        }
        usuarioDAO.crear(usuario);
        LogHttp.log("POST", "/usuarios", "Usuario creado: " + usuario.getUsername());
    }
}
```
Esta es la "lógica base" — validar y guardar. No sabe nada de valores por defecto.

```java
public class CreadorUsuarioConValoresPorDefecto implements CreadorUsuario {
    private final CreadorUsuario creadorOriginal; // guarda una REFERENCIA a otro CreadorUsuario

    public CreadorUsuarioConValoresPorDefecto(CreadorUsuario creadorOriginal) {
        this.creadorOriginal = creadorOriginal;
    }

    @Override
    public void crear(Usuario usuario) {
        if (usuario.getRol() == null || usuario.getRol().isBlank()) {
            usuario.setRol("RECEPCIONISTA");
        }
        usuario.setActivo(true);
        usuario.setCreatedAt(LocalDateTime.now());

        creadorOriginal.crear(usuario); // AQUÍ delega al objeto que envuelve
    }
}
```
**Esto es el patrón Decorador explicado paso a paso:**
1. `CreadorUsuarioConValoresPorDefecto` implementa la MISMA interfaz que envuelve (`CreadorUsuario`). Por eso se puede "encadenar": un decorador puede envolver a otro decorador, que envuelve a la base, y así sucesivamente (aquí no lo hacemos, pero el patrón lo permite).
2. Recibe en su constructor el objeto que va a decorar (`creadorOriginal`) — esto se llama **inyección de dependencias**: en vez de que la clase cree sus propias dependencias con `new`, se las "inyectan" desde afuera.
3. Agrega SU comportamiento (rol/estado/fecha por defecto) y al final **llama al objeto original** (`creadorOriginal.crear(usuario)`) para que haga lo suyo. La lógica base nunca se tocó ni se copió.

```java
public class UsuarioService {
    private final CreadorUsuario creador;

    public UsuarioService() {
        this.usuarioDAO = new UsuarioDAOImpl();
        this.creador = new CreadorUsuarioConValoresPorDefecto(new CreadorUsuarioBase(usuarioDAO));
        // así se "arma" el regalo envuelto: primero se crea la base, y la envolvemos
        // con el decorador, todo en una sola línea
    }

    public void registrarUsuario(Usuario usuario) {
        creador.crear(usuario);
        // UsuarioService no sabe (ni le importa) si "creador" es la base sola o
        // la base envuelta — solo sabe que cumple la interfaz CreadorUsuario.
        // Esto TAMBIÉN es polimorfismo.
    }
```

```java
public Usuario login(String username, String password) {
    Usuario u = usuarioDAO.obtenerPorUsername(username);
    if (u == null || !u.getPassword().equals(password)) {
        throw new ValidacionException("Usuario o contraseña incorrectos.");
    }
    if (!u.isActivo()) {
        throw new ValidacionException("El usuario está inactivo.");
    }
    LogHttp.log("POST", "/login", "Login exitoso: " + username);
    return u;
}
```
**Detalle importante para exponer con honestidad:** este login compara la contraseña en texto plano (`.equals(password)`). En un sistema real, las contraseñas NUNCA se guardan ni se comparan así — se guarda un "hash" (una huella digital irreversible de la contraseña, con librerías como BCrypt), y se compara el hash, nunca el texto plano. Aquí se simplificó a propósito porque el enunciado no pide seguridad de contraseñas, solo autenticación básica — pero si te preguntan "¿esto es seguro?", la respuesta correcta es "no, y aquí está la razón, y así se arreglaría en producción".

### OrdenService.java y las transacciones — el otro punto crítico

```java
public void registrarOrden(Orden orden) {
    ...
    orden.setFechaIngreso(LocalDateTime.now());
    orden.calcularCostoTotal();

    try (Connection con = ConexionDB.obtenerConexion()) {
        con.setAutoCommit(false);
        // por defecto, JDBC hace "auto-commit": cada sentencia SQL se confirma sola,
        // al instante. Al ponerlo en false, le decimos "espera mis instrucciones,
        // no confirmes nada todavía"
        try {
            ordenDAO.guardarTransaccional(con, orden);
            // OJO: le pasamos la MISMA conexión "con" — así el INSERT de la orden
            // ocurre DENTRO de esta transacción, no en una conexión aparte

            for (DetalleOrden detalle : orden.getRepuestosUtilizados()) {
                repuestoDAO.descontarStock(con, detalle.getRepuesto().getId(), detalle.getCantidad());
                // MISMA conexión otra vez — el descuento de stock de cada repuesto
                // también ocurre dentro de esta transacción
            }

            con.commit();
            // si llegamos hasta aquí sin excepciones: confirmamos TODO de una vez.
            // Antes de este commit(), nada de esto es visible para otras consultas
            // ni queda grabado permanentemente
            LogHttp.log("POST", "/ordenes", "Orden registrada #" + orden.getId());
        } catch (SQLException | ValidacionException e) {
            con.rollback();
            // algo falló (por ejemplo, stock insuficiente en el tercer repuesto,
            // después de ya haber "insertado" la orden y descontado los dos primeros):
            // rollback() DESHACE todo lo que se hizo desde el último commit,
            // como si nunca hubiera pasado
            throw new PersistenciaException("No se pudo registrar la orden, se revirtieron los cambios.", e);
        } finally {
            con.setAutoCommit(true);
            // se restaura el comportamiento normal de la conexión (por si se reutilizara)
        }
    } catch (SQLException e) {
        throw new PersistenciaException("Error de conexión al registrar la orden.", e);
    }
}
```

**Esto es exactamente lo que pide el punto 6 del enunciado**: *"Registro de Orden de Servicio → setAutoCommit(false) → insertar orden → actualizar inventario → commit()/rollback()"*.

**Pregunta típica: "¿por qué es tan importante que sea todo o nada aquí?"** Porque si se guardara la orden pero fallara el descuento de stock (por ejemplo, porque el internet se cortó a mitad de camino, o porque justo en ese milisegundo alguien más se llevó el último repuesto disponible), quedaría una orden registrada en el sistema diciendo "usé 3 tornillos de este tipo", pero el inventario seguiría mostrando que esos 3 tornillos siguen disponibles. Alguien más podría "usarlos" de nuevo en otra orden, y terminarías vendiendo/usando repuestos que no existen. La transacción evita ese desfase.

**Multi-catch** (`catch (SQLException | ValidacionException e)`): permite capturar dos tipos de excepción distintos en un solo bloque `catch`, cuando el manejo que le vas a dar es el mismo para ambos (en este caso, siempre: `rollback()` y relanzar como `PersistenciaException`).

### RepuestoDAOImpl.descontarStock() — la versión transaccional

```java
@Override
public void descontarStock(Connection con, int repuestoId, int cantidad) throws SQLException {
    String sqlLeer = "SELECT stock_disponible FROM repuestos WHERE id = ? FOR UPDATE";
    // "FOR UPDATE" le dice a la base de datos: "bloquea esta fila hasta que yo
    // termine mi transacción" — así, si dos usuarios intentan usar el mismo
    // repuesto exactamente al mismo tiempo, el segundo tiene que ESPERAR a que
    // el primero termine (commit o rollback), en vez de leer un stock desactualizado
    ...
    if (cantidad > stockActual) {
        throw new hub3.tallerexpress.exception.ValidacionException("Stock insuficiente para el repuesto id=" + repuestoId);
    }
    ...
}
```
**Esto es una "condición de carrera" (race condition) y cómo se evita.** Sin el `FOR UPDATE`, podría pasar: usuario A lee "quedan 2", usuario B lee "quedan 2" (al mismo tiempo), A descuenta 2 (queda en 0), B también descuenta 2 (queda en -2, o sobreescribe el 0 con "2 - 2 = 0" pero ya vendió de más). Con `FOR UPDATE`, B tiene que esperar a que A termine su transacción completa antes de poder leer esa fila.

---

## PARTE 6 — `controller` (el "mesero")

Todos los controllers siguen el mismo patrón trivial: reciben la llamada de la pantalla y la reenvían al `service` correspondiente, sin lógica propia. Ejemplo (`ClienteController`):

```java
public class ClienteController {
    private final ClienteService clienteService;

    public ClienteController() {
        this.clienteService = new ClienteService();
    }

    public void registrar(Cliente cliente) {
        clienteService.registrarCliente(cliente);
    }
    ...
}
```
**`private final ClienteService clienteService`**: `final` en un atributo significa que, una vez asignado (en el constructor), no se puede volver a reasignar. Se usa mucho en este proyecto para las dependencias de una clase, porque una vez que un `ClienteController` tiene SU `ClienteService`, no tiene sentido que cambie a mitad de camino.

**Único controller con algo extra: `UsuarioController`**, porque guarda quién inició sesión:
```java
private Usuario usuarioEnSesion;

public Usuario login(String username, String password) {
    this.usuarioEnSesion = usuarioService.login(username, password);
    return usuarioEnSesion;
}

public boolean esAdmin() {
    return usuarioEnSesion != null && "ADMIN".equals(usuarioEnSesion.getRol());
}
```
**`"ADMIN".equals(usuarioEnSesion.getRol())` en vez de `usuarioEnSesion.getRol().equals("ADMIN")`:** es un truco defensivo a propósito. Si `getRol()` devolviera `null`, la segunda forma tronaría (`NullPointerException`); la primera forma nunca truena, porque `"ADMIN"` nunca es `null` — en el peor caso, simplemente devuelve `false`.

---

## PARTE 7 — `presentation` (las ventanas JOptionPane)

### El patrón que se repite en TODAS las vistas

```java
public void mostrar() {
    boolean volver = false;
    while (!volver) {
        String[] opciones = {"Registrar", "Editar", ..., "Volver"};
        int seleccion = JOptionPane.showOptionDialog(null, "...", "...",
                JOptionPane.DEFAULT_OPTION, JOptionPane.PLAIN_MESSAGE, null, opciones, opciones[0]);
        if (seleccion == -1) return; // el usuario cerró la ventana con la X

        switch (opciones[seleccion]) {
            case "Registrar" -> registrar();
            ...
            case "Volver" -> volver = true;
        }
    }
}
```
- **`while (!volver)`**: un bucle que se repite hasta que el usuario elige "Volver" — así el menú de repuestos, por ejemplo, se sigue mostrando una y otra vez después de cada operación, en vez de cerrarse solo.
- **`JOptionPane.showOptionDialog(...)`**: crea una ventana con botones personalizados (uno por cada elemento de `opciones`). Devuelve el ÍNDICE (posición) del botón que se presionó, o `-1` si se cerró la ventana sin elegir nada.
- **`switch (...) { case "X" -> ... }`**: esta es la sintaxis moderna de `switch` en Java (desde Java 14), con flechas `->` en vez de `case "X": ... break;`. Hace lo mismo que un switch tradicional pero sin necesidad de `break` (evita el error clásico de olvidar el `break` y que el código "caiga" al siguiente caso).

### Manejo de errores en cada operación

```java
private void registrar() {
    try {
        ...
        repuestoController.registrar(r);
        JOptionPane.showMessageDialog(null, "Repuesto registrado con éxito.", "Éxito", JOptionPane.INFORMATION_MESSAGE);
    } catch (NumberFormatException e) {
        JOptionPane.showMessageDialog(null, "Stock y precio deben ser números válidos.", "Error", JOptionPane.ERROR_MESSAGE);
    } catch (ValidacionException | PersistenciaException e) {
        JOptionPane.showMessageDialog(null, e.getMessage(), "Error", JOptionPane.ERROR_MESSAGE);
    }
}
```
- **`NumberFormatException`**: se lanza automáticamente cuando `Integer.parseInt("abc")` o `Double.parseDouble("abc")` reciben un texto que no es un número válido. La capturamos aparte porque su mensaje por defecto es feo/técnico, y preferimos mostrar uno propio.
- **`ValidacionException | PersistenciaException`**: para estas dos sí mostramos `e.getMessage()` directamente, porque esos mensajes YA los escribimos nosotros en el `service` pensando en que el usuario los lea (ej: "Ya existe un repuesto con ese código de referencia.").

### LoginView.java — el uso de `JPasswordField`

```java
JPasswordField campoPassword = new JPasswordField();
int opcion = JOptionPane.showConfirmDialog(null, campoPassword, "Contraseña:", JOptionPane.OK_CANCEL_OPTION, JOptionPane.PLAIN_MESSAGE);
if (opcion != JOptionPane.OK_OPTION) {
    return null;
}
String password = new String(campoPassword.getPassword());
```
- `JOptionPane.showInputDialog` no tiene versión oculta, así que en vez de eso metemos un componente de Swing (`JPasswordField`) DENTRO de un `showConfirmDialog` genérico — el segundo parámetro de `showConfirmDialog` puede ser cualquier componente visual, no solo texto.
- **`campoPassword.getPassword()`** devuelve un `char[]` (arreglo de caracteres), no un `String` — por seguridad (los `String` en Java son inmutables y pueden quedar en memoria más tiempo del deseado; un `char[]` se puede "borrar" explícitamente). Por eso hay que convertirlo con `new String(...)`.

### OrdenView.java — el flujo más largo, para armar el detalle de repuestos

```java
boolean seguirAgregando = true;
while (seguirAgregando) {
    String codigo = JOptionPane.showInputDialog("Código del repuesto a usar (Cancelar para terminar):");
    if (codigo == null) break; // el usuario le dio "Cancelar": se sale del while
    Repuesto r = repuestoController.buscarPorCodigo(codigo);
    if (r == null) {
        JOptionPane.showMessageDialog(null, "No existe un repuesto con ese código.", "Error", JOptionPane.ERROR_MESSAGE);
        continue; // salta el resto de este ciclo y vuelve a preguntar el código
    }
    int cantidad = Integer.parseInt(JOptionPane.showInputDialog("Cantidad de \"" + r.getNombre() + "\":"));
    orden.agregarRepuesto(new DetalleOrden(0, r, cantidad, r.getPrecioUnitario()));

    int opcion = JOptionPane.showConfirmDialog(null, "¿Agregar otro repuesto?", "Repuestos", JOptionPane.YES_NO_OPTION);
    seguirAgregando = (opcion == JOptionPane.YES_OPTION);
}
```
- **`break` vs `continue`**: `break` sale COMPLETAMENTE del bucle. `continue` salta solo el resto de la vuelta actual y sigue con la siguiente. Aquí se usan los dos con propósitos distintos: cancelar termina todo, código inválido solo repite la pregunta.
- El precio que se guarda en cada `DetalleOrden` es `r.getPrecioUnitario()` — el precio ACTUAL del repuesto en el momento de armar la orden, que después queda "congelado" en esa línea aunque el precio del repuesto cambie después (como se explicó en `DetalleOrden`).

### MenuPrincipalView.java — cómo se decide qué botones mostrar según el rol

```java
String[] opcionesAdmin = {"Repuestos", "Clientes", "Vehículos", "Usuarios", "Órdenes de Servicio", "Salir"};
String[] opcionesRecepcionista = {"Repuestos", "Clientes", "Vehículos", "Órdenes de Servicio", "Salir"};
String[] opciones = usuarioController.esAdmin() ? opcionesAdmin : opcionesRecepcionista;
```
Esta línea es la implementación completa del requisito "roles ADMIN/RECEPCIONISTA": el recepcionista simplemente nunca ve el botón "Usuarios" en su menú — no es que se le niegue el acceso con un mensaje de error, directamente la opción no existe para él. (Nota honesta si preguntan: esto es solo a nivel de interfaz. Si alguien llamara directamente al `UsuarioController` sin pasar por el menú, técnicamente podría saltarse esa restricción — en un sistema real, la validación de permisos también debería estar en el `service`, no solo en la pantalla.)

---

## PARTE 8 — `util` (utilidades)

### LogHttp.java
```java
public static void log(String metodo, String endpoint, String detalle) {
    System.out.println("[" + metodo + "] " + endpoint + " -> " + detalle);
}
```
Simula el formato de un log de API REST (`[POST] /clientes -> Cliente creado: 12345`), aunque no hay ningún servidor HTTP real corriendo — es puramente para cumplir el requisito de "registrar operaciones CRUD simulando trazas de llamadas HTTP en consola".

### TablaHelper.java
```java
sb.append(String.format("%-10s %-20s %-12s %8s %8s %10s %-10s%n",
        "CODIGO", "NOMBRE", "CATEGORIA", "STOCK_T", "STOCK_D", "PRECIO", "ESTADO"));
```
**`String.format` con `%-10s`**: el `%s` es "aquí va un texto"; el número es el ancho mínimo en caracteres; el `-` significa "alineado a la izquierda" (sin `-`, se alinea a la derecha). Esto es lo que hace que las columnas queden alineadas como una tabla real cuando se imprime con una fuente monoespaciada.

### VentanaUtil.java
```java
JTextArea area = new JTextArea(contenido);
area.setFont(new Font("Monospaced", Font.PLAIN, 12));
```
JOptionPane normal usa una fuente donde cada letra tiene un ancho distinto (como este texto que estás leyendo) — en esa fuente, una tabla con columnas alineadas por espacios se ve torcida. `"Monospaced"` es una fuente donde TODOS los caracteres miden lo mismo de ancho (como en un editor de código), así los espacios de `TablaHelper` sí alinean visualmente.

### InicializarBD.java
```java
String sql = Files.readString(Paths.get("tablas.sql"));
...
for (String comando : sql.split(";")) {
    if (!comando.isBlank()) {
        stmt.execute(comando);
    }
}
```
- **`Files.readString(...)`**: lee todo el contenido de un archivo de texto y lo devuelve como un único `String`.
- **`.split(";")`**: parte ese texto gigante en varios pedazos, cortando cada vez que encuentra un punto y coma — porque `tablas.sql` tiene varias sentencias `CREATE TABLE ...;` seguidas, y JDBC solo puede ejecutar una sentencia SQL a la vez con `.execute()`.

---

## PARTE 9 — Preguntas típicas que te pueden hacer, y cómo responderlas

**"¿Por qué usaste JDBC puro y no un ORM como Hibernate?"**
Porque el enunciado pide explícitamente JDBC — un ORM (Object-Relational Mapper) automatiza el mapeo entre objetos Java y tablas SQL, pero aquí se pidió hacerlo a mano precisamente para aprender qué hace ese mapeo por dentro.

**"¿Qué pasa si dos personas usan el sistema al mismo tiempo?"**
En general no hay control de concurrencia salvo en un punto crítico: `RepuestoDAOImpl.descontarStock()`, que usa `SELECT ... FOR UPDATE` dentro de la transacción de una orden, justamente para evitar que dos órdenes descuenten el mismo stock al mismo tiempo de forma incorrecta.

**"¿Por qué separaste `service` de `controller` si el controller casi no hace nada?"**
Porque son responsabilidades distintas aunque hoy se vean parecidas: `controller` es "la puerta de entrada desde una interfaz específica" (JOptionPane); `service` es "las reglas de negocio, sin importar qué interfaz las llame". Si mañana agregas una segunda interfaz (por ejemplo, una API REST), escribes un `RepuestoRestController` nuevo que reutiliza el MISMO `RepuestoService` — no duplicas ninguna validación.

**"¿Dónde está la herencia?"**
No hay una jerarquía de herencia entre las clases del `model` (se decidió mantenerlas simples, sin una superclase común). Donde SÍ hay relaciones fuertes entre clases es composición (`Vehiculo` tiene un `Cliente`, `Orden` tiene un `Vehiculo` y una lista de `DetalleOrden`, `DetalleOrden` tiene un `Repuesto`). El polimorfismo real del proyecto está en el patrón Decorador de `Usuario` (dos clases distintas cumpliendo la misma interfaz `CreadorUsuario`).

**"¿Qué pasa si se cae la conexión a la base de datos a la mitad de una transacción?"**
El `try (Connection con = ...)` de `OrdenService.registrarOrden()` está anidado en otro `try/catch` que captura `SQLException` en el nivel más externo también, y lo convierte en `PersistenciaException` — el usuario ve un mensaje de error, y como nunca se llamó a `commit()`, nada quedó guardado a medias.

**"¿Por qué el username/password del login se comparan en texto plano?"**
Simplificación consciente para el alcance del ejercicio (autenticación básica, no seguridad de credenciales). En producción se guardaría un hash con sal (ej. BCrypt) y se compararía el hash, nunca la contraseña original.

**"¿Cómo garantizas que el código de referencia de un repuesto sea único?"**
Doble capa: la base de datos tiene `UNIQUE` en esa columna (última línea de defensa, a nivel de motor de datos), y además `RepuestoService.registrarRepuesto()` llama a `repuestoDAO.existeCodigo(...)` ANTES de intentar guardar, para poder mostrarle al usuario un mensaje claro (`ValidacionException`) en vez de que le llegue un error crudo de SQL si se topara con la restricción `UNIQUE` directamente.

**"Explícame la diferencia entre `service.actualizarStock` y `dao.actualizarStock`"**
El `service` (cuando se llama fuera del contexto de una orden, por ejemplo si un admin corrige el inventario a mano) valida antes de escribir. El `dao` es la capa de más bajo nivel que solo ejecuta el `UPDATE`; el `service` es quien decide CUÁNDO es seguro llamarlo.

---

## Checklist final de repaso

- [ ] Puedo explicar qué hace cada una de las 7 carpetas (`model`, `exception`, `config`, `dao`, `service`, `controller`, `presentation`, `util`) y en qué orden se llaman entre sí.
- [ ] Puedo señalar un ejemplo real de encapsulamiento, composición y polimorfismo en el código.
- [ ] Puedo explicar el patrón Decorador con `CreadorUsuario` sin mirar el código.
- [ ] Puedo explicar por qué usamos `PreparedStatement` en vez de concatenar Strings.
- [ ] Puedo explicar, paso a paso, qué hace `setAutoCommit(false)` → `commit()` / `rollback()` y por qué es necesario en `OrdenService.registrarOrden()`.
- [ ] Puedo explicar la diferencia entre `ValidacionException` y `PersistenciaException`, y dar un ejemplo de cuándo se lanza cada una.
- [ ] Puedo correr el programa de memoria: `InicializarBD` primero (una sola vez), después `TallerExpress`, login con `admin`/`admin123`.
