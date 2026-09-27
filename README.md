# Post-contenido — Unidad 5: Integración en Aplicaciones Web

## Cómo ejecutar
```
$ mvn clean package
$ mvn spring-boot:run
```
API REST: http://localhost:8080/api/reservas
Vista MVC: http://localhost:8080/reservas

## Decisiones de diseño

### Punto de decisión 1 — Ubicación de la validación de solapamiento
Para detectar si una nueva reserva se solapa con una reserva activa existente del mismo laboratorio, existían dos alternativas: (a) traer a memoria todas las reservas del laboratorio con findByLaboratorioId y comparar los rangos de fecha con lógica Java pura dentro del Service, o (b) delegar el filtrado a una consulta JPQL en el Repository (buscarSolapamientos) que resuelve la comparación de rangos directamente en la base de datos.   

Se optó por la segunda alternativa. La razón es de rendimiento y escalabilidad: la cantidad de reservas históricas de un laboratorio crece indefinidamente con el tiempo, mientras que las reservas que realmente pueden solaparse con un rango dado son solo las que caen cerca de ese rango. Traer todo el historial a memoria para filtrarlo en Java es un trabajo que crece sin límite y que la base de datos ya sabe resolver de forma eficiente con un índice sobre laboratorio_id, inicio y fin.    

Sin embargo, es importante distinguir qué responsabilidad quedó en cada capa: el Repository solo responde una pregunta de datos ("¿qué reservas se solapan con este rango de tiempo?"), no toma ninguna decisión de negocio. Es ReservaService quien interpreta ese resultado y decide si la reserva se permite o no, lanzando ReservaConflictException con un mensaje claro cuando corresponde. Si el Controller llamara directamente a buscarSolapamientos() sin pasar por el Service, la API dejaría de aplicar la regla de negocio: devolvería la lista de conflictos, pero nada impediría que igualmente se guardara la reserva solapada, porque la decisión de rechazarla vive únicamente en ReservaService.crear(). Es decir, se perdería la garantía de integridad de la regla de negocio, que quedaría a criterio de cada consumidor del Repository en lugar de estar centralizada en un solo lugar.   


### Punto de decisión 2 — Reglas con y sin apoyo del Repository   

No todas las reglas de negocio de ReservaService necesitan acceso a la base de datos. La validación de horario de atención (07:00–21:00) y de duración permitida (30 minutos a 3 horas), implementada en validarHorarioYDuracion, depende exclusivamente de los campos inicio y fin del propio objeto Reserva que se está creando; no requiere comparar contra ninguna otra fila de la tabla reservas ni de ninguna otra tabla. Por eso esta regla se implementó enteramente en el Service con Java puro (cálculo de Duration y comparación de LocalTime), sin tocar el Repository ni generar ninguna consulta SQL.

El criterio general que separa ambos tipos de regla es el siguiente: si la regla necesita comparar el objeto contra datos que solo la base de datos conoce —como la existencia de otras reservas activas en un rango de tiempo—, conviene apoyarse en una consulta específica del Repository, porque intentar resolverlo solo con el objeto en memoria implicaría, de todos modos, tener que consultar esos otros datos. En cambio, si la regla solo depende de los propios atributos del objeto que se está validando, involucrar al Repository o a la base de datos no aporta nada: solo agregaría una consulta innecesaria y acoplaría una validación puramente de dominio a la capa de persistencia. Aplicar este criterio de forma consistente es, además, lo que evita que ReservaService se convierta en un Service anémico: cada regla vive en el lugar que le corresponde según qué datos necesita para evaluarse, no según una plantilla mecánica de "todo pasa por el Repository" o "todo se valida en el Service".