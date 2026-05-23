# SGFH — Guía de implementación, ejecución y despliegue

**Sistema de Gestión de Farmacia Hospitalaria**  
Hospital Universitario Nacional · v1.0.0 · Mayo 2026

---

## Requisitos previos

| Herramienta | Versión mínima | Verificar |
|---|---|---|
| Java JDK | 21 | `java -version` |
| Maven | 3.9+ | `mvn -version` |
| Docker Desktop | 4.x | `docker --version` |
| Node.js | 20+ | `node --version` |
| npm | 9+ | `npm --version` |
| Git | cualquiera | `git --version` |

**Hardware mínimo:** 16 GB RAM · 4 núcleos CPU · 10 GB disco libre

---

## 1. Clonar el repositorio

```bash
git clone <URL_REPOSITORIO>
cd sgfh_proyecto_completo
```

---

## 2. Levantar la infraestructura Docker

Todos los contenedores (bases de datos, Kafka, Redis) se levantan con un solo comando.

```powershell
cd docker
docker compose up -d
```

Verificar que los 8 contenedores están activos:

```powershell
docker compose ps
```

Resultado esperado:

```
NAME                     STATUS
sgfh_db_inventario       Up (healthy)   :5432
sgfh_db_prescripciones   Up (healthy)   :5433
sgfh_db_dispensacion     Up (healthy)   :5434
sgfh_db_usuarios         Up (healthy)   :5435
sgfh_kafka               Up             :9092
sgfh_zookeeper           Up             :2181
sgfh_kafka_ui            Up             :8090
sgfh_redis               Up             :6379
```

Si algún contenedor no arranca:

```powershell
docker compose logs <nombre_contenedor>
```

---

## 3. Configurar las bases de datos (primera vez)

Las tablas se crean automáticamente via Flyway cuando arrancan los microservicios. Si Flyway falla, crear las tablas manualmente:

### 3.1 Inventario (datos de prueba incluidos)

```powershell
docker exec -it sgfh_db_inventario psql -U postgres -d farmacia_inventario
```

Dentro de psql, ejecutar:

```sql
CREATE TABLE IF NOT EXISTS medicamentos (
    id BIGSERIAL PRIMARY KEY,
    codigo VARCHAR(20) NOT NULL UNIQUE,
    nombre_generico VARCHAR(200) NOT NULL,
    nombre_comercial VARCHAR(200),
    dci VARCHAR(200),
    forma_farmaceutica VARCHAR(30) NOT NULL,
    concentracion VARCHAR(50) NOT NULL,
    unidad_medida VARCHAR(20) NOT NULL,
    categoria VARCHAR(30) NOT NULL DEFAULT 'GENERAL',
    requiere_receta BOOLEAN NOT NULL DEFAULT TRUE,
    stock_minimo INTEGER NOT NULL DEFAULT 10,
    stock_maximo INTEGER NOT NULL DEFAULT 1000,
    activo BOOLEAN NOT NULL DEFAULT TRUE,
    creado_en TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS lotes (
    id BIGSERIAL PRIMARY KEY,
    medicamento_id BIGINT NOT NULL REFERENCES medicamentos(id),
    numero_lote VARCHAR(50) NOT NULL UNIQUE,
    fecha_fabricacion DATE,
    fecha_vencimiento DATE NOT NULL,
    cantidad_inicial INTEGER NOT NULL,
    cantidad_disponible INTEGER NOT NULL,
    precio_unitario NUMERIC(10,2) NOT NULL,
    ubicacion_almacen VARCHAR(50),
    estado VARCHAR(20) NOT NULL DEFAULT 'ACTIVO'
);

CREATE TABLE IF NOT EXISTS movimientos_stock (
    id BIGSERIAL PRIMARY KEY,
    lote_id BIGINT REFERENCES lotes(id),
    medicamento_id BIGINT NOT NULL REFERENCES medicamentos(id),
    tipo_movimiento VARCHAR(30) NOT NULL,
    cantidad INTEGER NOT NULL,
    cantidad_antes INTEGER NOT NULL,
    cantidad_despues INTEGER NOT NULL,
    motivo VARCHAR(200),
    referencia_id VARCHAR(50),
    usuario_id BIGINT,
    creado_en TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS alertas_stock (
    id BIGSERIAL PRIMARY KEY,
    medicamento_id BIGINT NOT NULL REFERENCES medicamentos(id),
    tipo_alerta VARCHAR(30) NOT NULL,
    mensaje VARCHAR(500),
    activa BOOLEAN NOT NULL DEFAULT TRUE,
    creado_en TIMESTAMP NOT NULL DEFAULT NOW()
);

INSERT INTO medicamentos (codigo, nombre_generico, nombre_comercial, dci, forma_farmaceutica, concentracion, unidad_medida, categoria, requiere_receta, stock_minimo, stock_maximo) VALUES
('MED-001', 'Paracetamol', 'Panadol', 'Acetaminofen', 'TABLETA', '500', 'mg', 'GENERAL', false, 50, 1000),
('MED-002', 'Amoxicilina', 'Amoxil', 'Amoxicilina', 'CAPSULA', '500', 'mg', 'GENERAL', true, 30, 500),
('MED-003', 'Morfina', 'MST', 'Morfina', 'AMPOLLA', '10', 'mg/ml', 'CONTROLADO', true, 10, 100),
('MED-004', 'Insulina Glargina', 'Lantus', 'Insulina glargina', 'SOLUCION_INYECTABLE', '100', 'UI/ml', 'REFRIGERADO', true, 20, 200),
('MED-005', 'Metformina', 'Glucophage', 'Metformina', 'TABLETA', '850', 'mg', 'GENERAL', true, 50, 800);

INSERT INTO lotes (medicamento_id, numero_lote, fecha_fabricacion, fecha_vencimiento, cantidad_inicial, cantidad_disponible, precio_unitario, ubicacion_almacen, estado) VALUES
(1, 'L-PAR-2024-001', '2024-01-01', '2026-08-15', 500, 120, 0.15, 'A-01-01', 'ACTIVO'),
(1, 'L-PAR-2024-002', '2024-06-01', '2026-12-31', 500, 350, 0.15, 'A-01-02', 'ACTIVO'),
(2, 'L-AMO-2024-001', '2024-03-01', '2026-09-30', 300, 45, 0.80, 'A-02-01', 'ACTIVO'),
(2, 'L-AMO-2025-001', '2025-01-01', '2027-01-31', 400, 380, 0.78, 'A-02-02', 'ACTIVO'),
(3, 'L-MOR-2025-001', '2025-03-01', '2027-03-01', 50, 42, 8.50, 'C-01-01', 'ACTIVO'),
(4, 'L-INS-2025-001', '2025-01-01', '2026-07-31', 60, 18, 45.00, 'R-01-01', 'ACTIVO'),
(4, 'L-INS-2025-002', '2025-06-01', '2027-01-31', 80, 75, 44.50, 'R-01-02', 'ACTIVO'),
(5, 'L-MET-2024-001', '2024-09-01', '2026-10-31', 600, 55, 0.25, 'A-03-01', 'ACTIVO'),
(5, 'L-MET-2025-001', '2025-02-01', '2027-02-28', 700, 690, 0.24, 'A-03-02', 'ACTIVO');

\q
```

### 3.2 Prescripciones, Dispensación y Usuarios

Flyway crea las tablas automáticamente al iniciar los microservicios. No requiere intervención manual.

---

## 4. Levantar los microservicios

Abrir **5 terminales separadas** en la raíz del proyecto. Arrancar en este orden:

### Terminal 1 — ms-usuarios

```powershell
cd backend\ms-usuarios
mvn spring-boot:run
```

Listo cuando muestra:
```
Started MsUsuariosApplication in X.XXX seconds
```

### Terminal 2 — ms-inventario

```powershell
cd backend\ms-inventario
mvn spring-boot:run
```

> **Nota:** Si Redis causa error al arrancar, verificar que `application.yml` tenga `spring.cache.type: none`

### Terminal 3 — ms-prescripciones

```powershell
cd backend\ms-prescripciones
mvn spring-boot:run
```

### Terminal 4 — ms-dispensacion

```powershell
cd backend\ms-dispensacion
mvn spring-boot:run
```

### Terminal 5 — api-gateway (siempre el último)

```powershell
cd backend\api-gateway
mvn spring-boot:run
```

---

## 5. Levantar el frontend

```powershell
cd frontend
npm install
npm run dev
```

---

## 6. Verificar que todo funciona

```powershell
# Salud de los microservicios
curl -UseBasicParsing http://localhost:8082/actuator/health
curl -UseBasicParsing http://localhost:8081/actuator/health
curl -UseBasicParsing http://localhost:8083/actuator/health
curl -UseBasicParsing http://localhost:8086/actuator/health
curl -UseBasicParsing http://localhost:8080/actuator/health

# Datos de prueba
curl -UseBasicParsing http://localhost:8080/api/v1/medicamentos
curl -UseBasicParsing http://localhost:8080/api/v1/prescripciones
curl -UseBasicParsing http://localhost:8080/api/v1/dispensaciones
```

Todos deben responder `StatusCode: 200`.

Abrir en el navegador: **http://localhost:3000**

---

## 7. Credenciales de acceso

| Usuario | Password | Rol |
|---|---|---|
| admin.farmacia | Admin123 | FARMACEUTICO_JEFE |
| jefe.farmacia | Admin123 | FARMACEUTICO_JEFE |
| farm.tecnico1 | Admin123 | TECNICO_FARMACIA |

---

## 8. Puertos del sistema

| Servicio | Puerto |
|---|---|
| Frontend React | 3000 |
| API Gateway | 8080 |
| ms-prescripciones | 8081 |
| ms-inventario | 8082 |
| ms-dispensacion | 8083 |
| ms-usuarios | 8086 |
| Kafka UI | 8090 |
| PostgreSQL inventario | 5432 |
| PostgreSQL prescripciones | 5433 |
| PostgreSQL dispensación | 5434 |
| PostgreSQL usuarios | 5435 |
| Kafka | 9092 |
| Redis | 6379 |

---

## 9. Apagar el sistema

```powershell
# Detener microservicios: Ctrl+C en cada terminal

# Detener Docker
cd docker
docker compose down

# Detener Docker conservando los datos
docker compose down --volumes=false
```

---

## 10. Problemas frecuentes

### Puerto ya en uso

```powershell
taskkill /F /IM java.exe
```

### ms-inventario no arranca por Redis

En `backend\ms-inventario\src\main\resources\application.yml` verificar:

```yaml
spring:
  cache:
    type: none
```

### Tablas no existen (Flyway falló)

```powershell
docker exec -it sgfh_db_inventario psql -U postgres -d farmacia_inventario
```

```sql
DROP SCHEMA public CASCADE;
CREATE SCHEMA public;
GRANT ALL ON SCHEMA public TO postgres;
\q
```

Reiniciar el microservicio. Flyway recreará las tablas desde cero.

### Frontend muestra datos vacíos

El token JWT dura 15 minutos. Cerrar sesión y volver a ingresar.

### Limpiar caché de Vite

```powershell
cd frontend
rmdir /s /q node_modules\.vite
npm run dev
```

### Healthcheck falla en Docker

El healthcheck usa `sgfh_user` pero el usuario real es `postgres`. Es un error en `docker-compose.yml` que no afecta el funcionamiento. Ignorar.

---

## 11. Variables de entorno (opcional)

Los microservicios usan valores por defecto si no se definen variables de entorno:

| Variable | Valor por defecto | Descripción |
|---|---|---|
| SPRING_DATASOURCE_URL | jdbc:postgresql://localhost:5432/... | URL de la BD |
| SPRING_DATASOURCE_USERNAME | postgres | Usuario BD |
| SPRING_DATASOURCE_PASSWORD | XXXXXXXX | Contraseña BD |
| SPRING_KAFKA_BOOTSTRAP_SERVERS | localhost:9092 | Kafka |
| SPRING_REDIS_HOST | localhost | Redis |
| VITE_API_URL | http://localhost:8080 | URL del gateway |

Para cambiar la URL del gateway en el frontend, crear `frontend/.env`:

```
VITE_API_URL=http://localhost:8080
```

---

## 12. Monitoreo

| Interfaz | URL | Descripción |
|---|---|---|
| Kafka UI | http://localhost:8090 | Topics, mensajes, consumer lag |
| Swagger ms-inventario | http://localhost:8082/swagger-ui.html | API REST |
| Swagger ms-prescripciones | http://localhost:8081/swagger-ui.html | API REST |
| Swagger ms-dispensacion | http://localhost:8083/swagger-ui.html | API REST |
| Swagger ms-usuarios | http://localhost:8086/swagger-ui.html | API REST |

---

## 13. Despliegue en producción (AWS)

> Esta sección es una guía referencial. El entorno actual es desarrollo local.

### 13.1 Prerequisitos AWS

- AWS CLI configurado (`aws configure`)
- kubectl instalado
- Helm 3 instalado
- Acceso a ECR, EKS y RDS

### 13.2 Construir imágenes Docker

```bash
# Compilar cada microservicio
cd backend/ms-inventario
mvn clean package -DskipTests
docker build -t sgfh/ms-inventario:1.0.0 .

# Repetir para cada microservicio
```

### 13.3 Publicar en AWS ECR

```bash
# Login a ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com

# Tag y push
docker tag sgfh/ms-inventario:1.0.0 <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/sgfh/ms-inventario:1.0.0
docker push <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/sgfh/ms-inventario:1.0.0
```

### 13.4 Desplegar en EKS

```bash
# Conectar al cluster
aws eks update-kubeconfig --region us-east-1 --name sgfh-cluster

# Desplegar con Helm
helm upgrade --install sgfh-inventario ./helm/ms-inventario \
  --set image.tag=1.0.0 \
  --set db.host=<RDS_ENDPOINT> \
  --namespace sgfh

# Verificar pods
kubectl get pods -n sgfh
```

### 13.5 Variables de entorno en producción

Usar AWS Secrets Manager o Kubernetes Secrets para credenciales:

```bash
kubectl create secret generic sgfh-db-secret \
  --from-literal=password=<PASSWORD> \
  --namespace sgfh
```
