# Guía de implementación: Champion's Gym en AWS

**Curso:** Infraestructura como Código (UPAO 2026-2)
**Equipo:** Integrante 1 = Fabrizio Montero · Integrante 2 = Piero Aguirre · Integrante 3 = Joaquín Portella
**Herramientas:** Terraform 1.14.8 · AWS provider 6.67.0 · GitHub Actions · Spring Boot 4.1 con Java 21 · Docker

> Los fragmentos de Terraform de esta guía pasaron `terraform validate` (Terraform 1.14.8, AWS 6.67.0) y los workflows pasaron `actionlint`. Todavía no se desplegaron: la primera prueba real en AWS es el paso 1.4.

---

## 0. Resumen

| Fase | Quién | Qué queda funcionando | Termina con |
|---|---|---|---|
| **1. Base** | Solo Integrante 1 | API Spring Boot en 2 tareas Fargate (subredes privadas, 2 zonas), RDS PostgreSQL Multi-AZ, API Gateway + VPC Link + Cloud Map, NAT + Internet Gateway, ECR, y GitHub Actions desplegando cada cambio que llega a `main` | tag `v0.1.0` |
| **2. Completar el diagrama** | Los 3 en paralelo | Web en S3 + CloudFront, WAF, autoescalado de 2 a 10, rollback, respaldos y failover de la BD, límite de peticiones en API Gateway | tag `v0.2.0` |
| **3. Funcionalidades** | Los 3 en paralelo | Socios y suscripciones, aforo y reservas, correos con SES, logs en CloudWatch, pagos con Mercado Pago, dominio con Route 53 (opcional) | tag `v1.0.0` |

Quedan fuera de la fase 1, como se pidió: monitoreo, funcionalidades del gimnasio y pasarela de pagos. La única excepción es un grupo de logs de los contenedores (7 días, sin alarmas), porque sin él no hay forma de ver por qué falla una tarea.

**Regla de oro:** desde que termina la fase 1, nadie ejecuta `terraform apply` desde su computadora. Todo cambio entra por un pull request y lo despliega GitHub Actions cuando llega a `main`. Así los 3 no se pisan el estado de Terraform.

---

## 1. Antes de empezar (Integrante 1, urgente)

1. **Se usa la cuenta AWS de Fabrizio (Integrante 1).** Tiene que estar en el plan de pago: el plan gratuito cierra la cuenta al terminar sus 6 meses y bloquea algunos servicios. Si está en el plan gratuito, pasarla con "Upgrade plan" en la consola o en Billing and Cost Management; los créditos que queden se siguen aplicando a las facturas. Anotar el ID de cuenta (12 dígitos): va en el nombre del bucket del estado y en los ARN de los roles.
2. **Presupuesto con alerta:** Billing and Cost Management → Budgets → presupuesto mensual de US$20 con aviso por correo al 80 %. No es parte de la arquitectura; sirve para que no llegue una sorpresa a la tarjeta.
3. **Herramientas (Windows + Git Bash):** Git, AWS CLI v2 (`aws configure` con un usuario IAM administrador de Fabrizio, nunca con el usuario raíz), Terraform 1.14.8 (`terraform version`), Docker Desktop y JDK 21 (Temurin) con IntelliJ o VS Code.
4. **Los Integrantes 2 y 3 no necesitan claves de AWS** para programar: prueban en local con Docker y su código llega a AWS por el pipeline. En el paso 1.6 reciben un usuario de solo lectura para mirar la consola.
5. **Costo:** con todo encendido como en el diagrama son unos **US$0,36 por hora (≈ US$8,70 por día)** en sa-east-1. Por eso existe el botón "apagar" (sección 8). No lo dejen prendido de noche.

---

## 2. Qué se construye (con las correcciones del 2 de noviembre)

```
Usuario ──HTTPS──▶ [fase 2: CloudFront + WAF] ──/api/*──▶ API Gateway (HTTP API)
                                                              │ VPC Link (borde de la VPC)
                                                              ▼ Cloud Map: ¿qué tareas están sanas?
Región sa-east-1 · VPC 10.0.0.0/16 · Internet Gateway (borde de la VPC)
┌ Zona sa-east-1a ───────────────────┐   ┌ Zona sa-east-1b ───────────────────┐
│ Pública 10.0.1.0/24   [NAT Gateway]│   │ Pública 10.0.2.0/24   [NAT Gateway]│
│ Privada 10.0.3.0/24   [Tarea]      │   │ Privada 10.0.4.0/24   [Tarea]      │ ← Cluster ECS (regional)
│ Privada 10.0.11.0/24  [RDS princ.] │══▶│ Privada 10.0.12.0/24  [RDS standby]│   réplica síncrona
└────────────────────────────────────┘   └────────────────────────────────────┘
Salida a internet: tarea → NAT de su misma zona → Internet Gateway → ECR (fase 1), SES y Mercado Pago (fase 3)
```

| Alcance | Servicios | Fase |
|---|---|---|
| Global | IAM · CloudFront y WAF · Route 53 y ACM | IAM en la 1 · CloudFront y WAF en la 2 · Route 53 y ACM en la 3 (opcional) |
| Regional | VPC, Internet Gateway, VPC Link, API Gateway, Cloud Map, ECR, cluster ECS · S3 (web) · SES, CloudWatch | Fase 1 · S3 en la 2 · SES y CloudWatch completo en la 3 |
| Zona de disponibilidad | Subredes, NAT Gateway, tareas Fargate, RDS principal y standby | Fase 1 |
| Externo | GitHub Actions · Mercado Pago | Fase 1 · Fase 3 |

**Correcciones aplicadas:**
- Las tareas Fargate van en subredes **privadas** (10.0.3.0/24 y 10.0.4.0/24), sin IP pública.
- Hay un **NAT Gateway por zona** en las subredes públicas y un **Internet Gateway** en la VPC. Por ahí salen las tareas a internet, y después a Mercado Pago. CloudFront solo recibe tráfico; no sirve como salida.
- RDS va en sus propias subredes privadas (10.0.11.0/24 y 10.0.12.0/24), en Multi-AZ.
- El cluster ECS es regional; cada tarea vive en la subred de una zona.
- Se usan los números del informe: CPU 70 % durante 5 minutos, de 2 a 10 tareas y 99,5 % de disponibilidad.

**Valores fijos (no cambiarlos):**

| Qué | Valor |
|---|---|
| Región y zonas | `sa-east-1`, zonas `sa-east-1a` y `sa-east-1b` |
| Nombres | prefijo `champions-gym-prod-` (los roles IAM de la app **deben** empezar así) |
| API | puerto 8080, todo bajo `/api` |
| Salud del contenedor | `GET /api/actuator/health/liveness` (no revisa la BD) |
| Versión en línea | `GET /api/version` → `{"version":"<SHA del commit>"}` |
| Base de datos | PostgreSQL 17, `db.t4g.micro`, BD `gym`, usuario `gym_admin` |
| Tarea Fargate | 0,5 vCPU y 1 GB, `X86_64` |
| Imagen | ECR `champions-gym-api`, tag = SHA del commit (tags inmutables) |
| Estado de Terraform | S3 `champions-gym-tfstate-<ACCOUNT_ID>`, key `champions-gym/prod/terraform.tfstate`, `use_lockfile = true` |
| Versiones | Terraform `1.14.8`, aws `6.67.0`, random `3.9.1` (fijas, sin `~>`), Java 21, Spring Boot 4.1.x |

**Recursos de apoyo que no se dibujan** (por si el profesor pregunta): bucket del estado de Terraform, Parameter Store (contraseña de la BD), grupos de seguridad, tablas de rutas, roles IAM de ECS y de GitHub, y el grupo de logs de los contenedores.

---

## 3. Estructura del repositorio

```
champions-gym/
├── AGENTS.md                ← contexto para las IAs (ver sección 10)
├── CLAUDE.md                ← una sola línea: @AGENTS.md (para Claude Code)
├── README.md  CONTRIBUTING.md  .gitignore  .gitattributes
├── docker-compose.yml       ← QA local: PostgreSQL + API
├── .github/
│   ├── pull_request_template.md
│   └── workflows/  ci.yml  deploy.yml  apagar.yml   (fase 2: rollback.yml)
├── app/                     ← API Spring Boot (monolito modular)
│   ├── Dockerfile  .dockerignore  pom.xml  mvnw
│   └── src/main/java/pe/championsgym/api/{comun,sedes,...}
├── frontend/                ← fase 2: HTML/CSS/JS para S3
├── infra/
│   ├── versions.tf  providers.tf  backend.tf  variables.tf  main.tf  outputs.tf  .terraform.lock.hcl
│   ├── envs/prod.tfvars  envs/prod.backend.hcl
│   ├── bootstrap/           ← se aplica 1 vez a mano, con estado local (el .tfstate NO se sube)
│   └── modules/
│       ├── network/      (I1)          VPC, subredes, IGW, NAT, rutas, grupos de seguridad
│       ├── database/     (I1)          RDS + contraseña en Parameter Store
│       ├── ecs_api/      (I1)          cluster, tarea, servicio, Cloud Map, roles, logs
│       ├── api_gateway/  (I1)          HTTP API, VPC Link, ruta /api/*
│       ├── frontend/     (I2, fase 2)  S3 + CloudFront
│       ├── autoscaling/  (I2, fase 2)  2 a 10 tareas por CPU
│       └── waf/          (I3, fase 2)  WAF (se crea en us-east-1)
└── docs/GUIA-IMPLEMENTACION.md   ← este archivo
```

---

## 4. Cómo trabajamos con ramas (los 3)

| Rama | Uso | ¿Despliega en AWS? |
|---|---|---|
| `main` | Lo que está en AWS. Solo recibe PR desde `develop` | Sí: cada merge corre `deploy.yml` |
| `develop` | Integración. Recibe los PR de cada tarea | No: solo pruebas (`ci.yml`) |
| `feature/<tema>` | Una tarea; sale de `develop` | No |
| `fix/<tema>` | Una corrección; sale de `develop` | No |

```bash
git checkout develop && git pull origin develop
git checkout -b feature/<tema>
# ... cambios y pruebas locales ...
git push -u origin feature/<tema>
# PR hacia develop → ci.yml corre tests, docker build y terraform plan
# → lo aprueba OTRO integrante → "Create a merge commit" → borrar la rama
```

- **Para ver un cambio en AWS:** PR de `develop` a `main` con el título "Release vX.Y.Z". Al hacer merge, `deploy.yml` lo publica. Hagan un release cada vez que terminen una tarea que quieran probar en AWS.
- **Commits:** Conventional Commits en inglés, como en el proyecto anterior: `feat(ecs): add service with circuit breaker`. Ámbitos: `app`, `core`, `network`, `database`, `ecs`, `api-gateway`, `bootstrap`, `ci`, `frontend`, `autoscaling`, `waf`, `docs`.
- **Para no pisarse:**
  - Cada módulo tiene dueño (tabla de la sección 3). Si necesitas tocar el de otro, que sea un cambio pequeño y avísale en el PR.
  - En `infra/main.tf` y `infra/outputs.tf` cada uno agrega su bloque **al final**, con un comentario que diga de quién es.
  - Las migraciones de BD llevan fecha y hora en el nombre (`V202611051530__crear_reservas.sql`), así dos ramas nunca usan el mismo número.
- **En local solo se hace esto** (no hace falta tener credenciales):
  ```bash
  docker compose up --build
  cd infra && terraform fmt -recursive && terraform init -backend=false && terraform validate
  ```
  El `terraform plan` real lo ven en el PR, en el check `terraform`.

---

## 5. Fase 1: el Integrante 1 levanta la base

Mientras tanto, los Integrantes 2 y 3 leen esta guía, instalan Docker y JDK 21, y cuando exista el paso 1.2 prueban la API en local.

### Paso 1.1 · Repositorio y ramas

1. Crear el repo `fabriziomontero36-cyber/champions-gym` en GitHub (con README). Agregar como colaboradores a Piero (`PieroAguirreDC`) y a Joaquín.
2. Clonarlo y crear `develop`: `git clone https://github.com/fabriziomontero36-cyber/champions-gym.git && cd champions-gym && git checkout -b develop && git push -u origin develop`.
3. En la rama `feature/estructura-repo`, agregar las carpetas de la sección 3 (con `.gitkeep`), `CONTRIBUTING.md` (copiar el del procesador de imágenes y cambiar los ámbitos), la plantilla de PR, `AGENTS.md`, `CLAUDE.md` (con la línea `@AGENTS.md`), esta guía en `docs/`, y estos dos archivos:

`.gitignore`
```gitignore
# Terraform
.terraform/
*.tfstate
*.tfstate.*
*.tfplan
crash*.log
override.tf
*_override.tf
.terraformrc
terraform.rc
# Java / Maven
target/
*.class
# Secretos y entorno local
.env
*.pem
.aws/
# Editor / SO
.vscode/
.idea/
*.iml
.DS_Store
Thumbs.db
```

`.gitattributes` (evita que Windows cambie los saltos de línea y rompa Docker y la CI)
```gitattributes
* text=auto eol=lf
*.cmd text eol=crlf
*.bat text eol=crlf
```

| # | Commit |
|---|---|
| 1 | `chore: add gitignore, gitattributes and contributing guide` |
| 2 | `docs: add implementation guide and ai context` |
| 3 | `chore: add project folder structure and pr template` |

PR hacia `develop`; lo revisa el Integrante 2 o el 3 (así practican desde el inicio).
**Listo cuando:** `develop` tiene la estructura y los otros 2 pueden clonarla.

### Paso 1.2 · API mínima y QA local (rama `feature/api-base`)

1. Generar el proyecto en [start.spring.io](https://start.spring.io): Maven, Java 21, Spring Boot **4.1.x** (la última 4.1 estable), group `pe.championsgym`, artifact `api`, paquete `pe.championsgym.api`. Dependencias: Spring Web, Spring Data JPA, PostgreSQL Driver, Flyway Migration, Spring Boot Actuator y Validation. Descomprimir dentro de `app/`.
   > En Spring Boot 4 los starters cambiaron de nombre (por ejemplo, Flyway necesita `spring-boot-starter-flyway` y `flyway-database-postgresql`). Initializr ya los pone bien: **no los cambien a mano**.
2. Borrar `application.properties` y crear `app/src/main/resources/application.yml`:
   ```yaml
   server:
     port: 8080
     servlet:
       context-path: /api        # todo queda bajo /api
     shutdown: graceful          # termina las peticiones en curso antes de apagarse
   spring:
     application:
       name: champions-gym-api
     datasource:                 # valores locales; en AWS los reemplazan las variables SPRING_DATASOURCE_*
       url: jdbc:postgresql://localhost:5432/gym
       username: gym_admin
       password: local123
       hikari:
         connection-timeout: 3000   # si la BD no responde, falla en 3 s en vez de colgarse
     jpa:
       open-in-view: false
       hibernate:
         ddl-auto: validate      # las tablas solo las crea Flyway
     flyway:
       out-of-order: true        # migraciones de ramas distintas
     lifecycle:
       timeout-per-shutdown-phase: 20s
   management:
     endpoints:
       web:
         exposure:
           include: health
     endpoint:
       health:
         probes:
           enabled: true         # crea /api/actuator/health/liveness
   app:
     version: ${APP_VERSION:local}
   ```
3. `GET /api/version` en `pe.championsgym.api.comun.VersionController`:
   ```java
   @RestController
   public class VersionController {

       @Value("${app.version}")
       private String version;

       @GetMapping("/version")
       public Map<String, String> version() {
           return Map.of("version", version);
       }
   }
   ```
4. Módulo `sedes` (paquete `pe.championsgym.api.sedes`): entidad `Sede` (id, nombre, ciudad, aforoMaximo), `SedeRepository extends JpaRepository<Sede, Long>` y `SedeController` con `GET /sedes`, que queda en `/api/sedes`. Migración `app/src/main/resources/db/migration/V202611020001__crear_tabla_sede.sql`:
   ```sql
   CREATE TABLE sede (
       id           BIGSERIAL PRIMARY KEY,
       nombre       VARCHAR(100) NOT NULL UNIQUE,
       ciudad       VARCHAR(100) NOT NULL,
       aforo_maximo INTEGER      NOT NULL CHECK (aforo_maximo > 0)
   );

   -- Aforos de ejemplo; ajustarlos con los datos reales del gimnasio.
   INSERT INTO sede (nombre, ciudad, aforo_maximo) VALUES
       ('Champion''s Gym Trujillo', 'Trujillo', 120),
       ('Champion''s Gym Chiclayo', 'Chiclayo', 100);
   ```
5. `app/Dockerfile` y `app/.dockerignore` (con una línea: `target/`):
   ```dockerfile
   # Etapa 1: compilar
   FROM maven:3.9-eclipse-temurin-21 AS build
   WORKDIR /build
   COPY pom.xml .
   RUN mvn -B -q dependency:go-offline
   COPY src ./src
   RUN mvn -B -q package -DskipTests

   # Etapa 2: ejecutar
   FROM eclipse-temurin:21-jre
   # curl lo usa el health check de ECS
   RUN apt-get update && apt-get install -y --no-install-recommends curl \
       && rm -rf /var/lib/apt/lists/* \
       && groupadd -r app && useradd -r -g app app
   WORKDIR /app
   COPY --from=build /build/target/*.jar app.jar
   USER app
   EXPOSE 8080
   ENV JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75"
   ENTRYPOINT ["java", "-jar", "/app/app.jar"]
   ```
6. `docker-compose.yml` en la raíz (es el QA local del informe):
   ```yaml
   services:
     bd:
       image: postgres:17
       environment:
         POSTGRES_DB: gym
         POSTGRES_USER: gym_admin
         POSTGRES_PASSWORD: local123
       ports:
         - "5432:5432"
       healthcheck:
         test: ["CMD-SHELL", "pg_isready -U gym_admin -d gym"]
         interval: 5s
         timeout: 5s
         retries: 10

     api:
       build: ./app
       environment:
         SPRING_DATASOURCE_URL: jdbc:postgresql://bd:5432/gym
         SPRING_DATASOURCE_USERNAME: gym_admin
         SPRING_DATASOURCE_PASSWORD: local123
         APP_VERSION: local
       ports:
         - "8080:8080"
       depends_on:
         bd:
           condition: service_healthy
   ```

Probar:
```bash
docker compose up --build
curl http://localhost:8080/api/sedes                     # las 2 sedes
curl http://localhost:8080/api/version                   # {"version":"local"}
curl http://localhost:8080/api/actuator/health/liveness  # {"status":"UP"}
```

| # | Commit |
|---|---|
| 1 | `feat(app): add spring boot api project` |
| 2 | `feat(app): add version endpoint and health probes` |
| 3 | `feat(app): add sedes table, entity and endpoint` |
| 4 | `build(app): add dockerfile and docker compose for local qa` |

**Listo cuando:** los 3 `curl` responden y `cd app && ./mvnw verify` pasa con el compose levantado.

### Paso 1.3 · Bootstrap (rama `feature/infra-bootstrap`)

`infra/bootstrap/main.tf` se aplica **una sola vez**, desde la computadora del Integrante 1 y con estado local. Crea lo que el pipeline necesita antes de existir:

| Recurso | Para qué |
|---|---|
| Bucket `champions-gym-tfstate-<ACCOUNT_ID>` | Estado remoto de Terraform. Es el mismo bloque de `tfstate-backend/main.tf` del procesador de imágenes, con el prefijo `champions-gym` y la región `sa-east-1` |
| ECR `champions-gym-api` | Imágenes de la API: tags inmutables, escaneo al subir y se guardan las 15 últimas |
| Proveedor OIDC de GitHub | GitHub Actions entra a AWS **sin claves guardadas** |
| Rol `champions-gym-github-deploy` | Lo usa solo la rama `main` (deploy, apagar y rollback). Tiene PowerUserAccess y permiso para crear solo los roles `champions-gym-prod-*` |
| Rol `champions-gym-github-plan` | Lo usan los pull requests. Solo lectura (ReadOnlyAccess) |

Las partes que más se equivocan (copiarlas tal cual; el bucket va entre `locals` y el ECR):
```hcl
variable "aws_region" {
  type    = string
  default = "sa-east-1"
}

variable "project" {
  type    = string
  default = "champions-gym"
}

variable "github_repo" {
  description = "dueno/repositorio en GitHub, con mayusculas exactas."
  type        = string
  default     = "fabriziomontero36-cyber/champions-gym"
}

data "aws_caller_identity" "current" {}

locals {
  account_id = data.aws_caller_identity.current.account_id
}

# Bucket: "${var.project}-tfstate-${local.account_id}" (copiado de tfstate-backend)

resource "aws_ecr_repository" "api" {
  name                 = "${var.project}-api"
  image_tag_mutability = "IMMUTABLE" # cada imagen = un commit; no se pisan
  force_delete         = true

  image_scanning_configuration {
    scan_on_push = true
  }
}

resource "aws_ecr_lifecycle_policy" "api" {
  repository = aws_ecr_repository.api.name

  policy = jsonencode({
    rules = [{
      rulePriority = 1
      description  = "Guardar solo las 15 imagenes mas recientes"
      selection    = { tagStatus = "any", countType = "imageCountMoreThan", countNumber = 15 }
      action       = { type = "expire" }
    }]
  })
}

resource "aws_iam_openid_connect_provider" "github" {
  url            = "https://token.actions.githubusercontent.com"
  client_id_list = ["sts.amazonaws.com"]
}

# Rol "deploy": solo la rama main
data "aws_iam_policy_document" "deploy_trust" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:${var.github_repo}:ref:refs/heads/main"]
    }
  }
}

resource "aws_iam_role" "github_deploy" {
  name               = "${var.project}-github-deploy"
  assume_role_policy = data.aws_iam_policy_document.deploy_trust.json
}

resource "aws_iam_role_policy_attachment" "deploy_poweruser" {
  role       = aws_iam_role.github_deploy.name
  policy_arn = "arn:aws:iam::aws:policy/PowerUserAccess"
}

# PowerUser no incluye IAM: solo se le deja manejar los roles de la app (champions-gym-prod-*).
resource "aws_iam_role_policy" "deploy_iam" {
  name = "roles-de-la-app"
  role = aws_iam_role.github_deploy.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "RolesDeLaApp"
        Effect = "Allow"
        Action = [
          "iam:CreateRole", "iam:DeleteRole", "iam:GetRole", "iam:UpdateRole",
          "iam:TagRole", "iam:UntagRole", "iam:UpdateAssumeRolePolicy",
          "iam:PutRolePolicy", "iam:GetRolePolicy", "iam:DeleteRolePolicy", "iam:ListRolePolicies",
          "iam:AttachRolePolicy", "iam:DetachRolePolicy", "iam:ListAttachedRolePolicies",
          "iam:ListInstanceProfilesForRole", "iam:PassRole",
        ]
        Resource = "arn:aws:iam::${local.account_id}:role/${var.project}-prod-*"
      },
      {
        Sid      = "LeerPoliticas"
        Effect   = "Allow"
        Action   = ["iam:GetPolicy", "iam:GetPolicyVersion", "iam:ListPolicies"]
        Resource = "*"
      },
    ]
  })
}

# Rol "plan": lo usan los pull requests; solo puede leer.
data "aws_iam_policy_document" "plan_trust" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:${var.github_repo}:pull_request"]
    }
  }
}

resource "aws_iam_role" "github_plan" {
  name               = "${var.project}-github-plan"
  assume_role_policy = data.aws_iam_policy_document.plan_trust.json
}

resource "aws_iam_role_policy_attachment" "plan_readonly" {
  role       = aws_iam_role.github_plan.name
  policy_arn = "arn:aws:iam::aws:policy/ReadOnlyAccess"
}
```

Salidas: `account_id`, `tfstate_bucket`, `ecr_repository_url`, `github_deploy_role_arn` y `github_plan_role_arn`.

Aplicar:
```bash
cd infra/bootstrap
terraform init
terraform apply
terraform output
```
- Guarden una copia de `infra/bootstrap/terraform.tfstate` fuera del repo (no se sube a GitHub).
- Si sale `EntityAlreadyExists` en el proveedor OIDC, la cuenta ya tiene uno de una prueba anterior. Importarlo y volver a aplicar:
  `terraform import aws_iam_openid_connect_provider.github arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com`

| # | Commit |
|---|---|
| 1 | `feat(bootstrap): add s3 bucket for remote state` |
| 2 | `feat(bootstrap): add ecr repository with lifecycle policy` |
| 3 | `feat(bootstrap): add github oidc provider with deploy and plan roles` |

**Listo cuando:** `terraform output` muestra los 5 valores.

### Paso 1.4 · Infraestructura base (rama `feature/infra-base`)

**Archivos de la raíz `infra/`:**
- `versions.tf`: Terraform `1.14.8`, aws `6.67.0`, random `3.9.1`.
- `providers.tf`: proveedor `aws` con `region = var.aws_region` y `default_tags` (`Project`, `Environment`, `ManagedBy = "terraform"`). Además, un segundo proveedor con `alias = "us_east_1"` y `region = "us-east-1"`: lo usará el WAF en la fase 2 (por ser global, AWS lo exige en us-east-1).
- `backend.tf`: `terraform { backend "s3" {} }`.
- `envs/prod.backend.hcl` (con el `account_id` del bootstrap):
  ```hcl
  bucket       = "champions-gym-tfstate-<ACCOUNT_ID>"
  key          = "champions-gym/prod/terraform.tfstate"
  region       = "sa-east-1"
  encrypt      = true
  use_lockfile = true
  ```
- `envs/prod.tfvars`:
  ```hcl
  env         = "prod"
  aws_region  = "sa-east-1"
  nat_per_az  = true # false = 1 solo NAT mientras desarrollan (ahorra US$0,10 por hora)
  db_multi_az = true # false = sin standby mientras desarrollan (ahorra US$0,04 por hora)
  ```
- `variables.tf`: `project` (default `"champions-gym"`), `env` (solo admite `"prod"`), `aws_region` (default `"sa-east-1"`), `image_tag` (sin default: lo pasa el pipeline), `vpc_cidr`, `public_subnet_cidrs`, `app_subnet_cidrs`, `db_subnet_cidrs` (con los CIDR de la sección 2 como default), `nat_per_az` (default `true`), `db_multi_az` (default `true`) y `log_retention_days` (default `7`).
- `main.tf` (es el contrato entre módulos; no cambiar los nombres):
  ```hcl
  # Modulo raiz: solo une los modulos. Cada modulo tiene un dueno (ver docs/GUIA-IMPLEMENTACION.md).

  locals {
    azs      = ["${var.aws_region}a", "${var.aws_region}b"]
    app_port = 8080
  }

  # Lo crea infra/bootstrap (una sola vez).
  data "aws_ecr_repository" "api" {
    name = "${var.project}-api"
  }

  # Integrante 1: VPC, subredes, Internet Gateway, NAT y grupos de seguridad
  module "network" {
    source = "./modules/network"

    project             = var.project
    env                 = var.env
    azs                 = local.azs
    vpc_cidr            = var.vpc_cidr
    public_subnet_cidrs = var.public_subnet_cidrs
    app_subnet_cidrs    = var.app_subnet_cidrs
    db_subnet_cidrs     = var.db_subnet_cidrs
    nat_per_az          = var.nat_per_az
    app_port            = local.app_port
  }

  # Integrante 1: RDS PostgreSQL (principal en 1a, standby en 1b)
  module "database" {
    source = "./modules/database"

    project       = var.project
    env           = var.env
    db_subnet_ids = module.network.db_subnet_ids
    sg_db_id      = module.network.sg_db_id
    multi_az      = var.db_multi_az
  }

  # Integrante 1: cluster ECS, tareas Fargate y Cloud Map
  module "ecs_api" {
    source = "./modules/ecs_api"

    project             = var.project
    env                 = var.env
    aws_region          = var.aws_region
    vpc_id              = module.network.vpc_id
    app_subnet_ids      = module.network.app_subnet_ids
    sg_api_id           = module.network.sg_api_id
    app_port            = local.app_port
    ecr_repository_url  = data.aws_ecr_repository.api.repository_url
    image_tag           = var.image_tag
    db_address          = module.database.db_address
    db_port             = module.database.db_port
    db_name             = module.database.db_name
    db_username         = module.database.db_username
    db_password_ssm_arn = module.database.db_password_ssm_arn
    log_retention_days  = var.log_retention_days
  }

  # Integrante 1: API Gateway HTTP API + VPC Link
  module "api_gateway" {
    source = "./modules/api_gateway"

    project              = var.project
    env                  = var.env
    app_subnet_ids       = module.network.app_subnet_ids
    sg_vpclink_id        = module.network.sg_vpclink_id
    cloudmap_service_arn = module.ecs_api.cloudmap_service_arn
  }

  # Fase 2 (Integrante 2): module "frontend"    -> S3 + CloudFront
  # Fase 2 (Integrante 2): module "autoscaling" -> 2 a 10 tareas, CPU 70 %
  # Fase 2 (Integrante 3): module "waf"         -> WAF en CloudFront (providers = { aws = aws.us_east_1 })
  ```
- `outputs.tf`: `api_endpoint`, `ecs_cluster_name`, `ecs_service_name`, `db_address` y `nat_public_ips`.

Cada módulo lleva `versions.tf` (solo `source = "hashicorp/aws"`, sin versión: la hereda de la raíz; `database` y `frontend` también declaran `hashicorp/random`), `variables.tf`, `main.tf` y `outputs.tf`. Los nombres de las variables y salidas están en `AGENTS.md`, sección 6.

**Módulo `network`**

| Recurso | Detalle |
|---|---|
| `aws_vpc` | `10.0.0.0/16`, con `enable_dns_support` y `enable_dns_hostnames` en `true` (los necesitan Cloud Map y RDS) |
| `aws_internet_gateway` | Unido a la VPC |
| `aws_subnet` × 6 (`count` sobre las 2 zonas) | `public` 10.0.1.0/24 y 10.0.2.0/24 · `app` 10.0.3.0/24 y 10.0.4.0/24 · `db` 10.0.11.0/24 y 10.0.12.0/24. Nombre: `champions-gym-prod-public-1a`, etc. |
| Tabla de rutas pública | `0.0.0.0/0` → Internet Gateway, asociada a las 2 públicas |
| `aws_eip` + `aws_nat_gateway` | Uno por zona en la subred pública (o uno solo si `nat_per_az = false`) |
| Tabla de rutas `app` (una por zona) | `0.0.0.0/0` → el NAT de su misma zona |
| Tabla de rutas `db` | Sin salida a internet |
| Grupos de seguridad | `sg-vpclink` (sale a `sg-api` por 8080) · `sg-api` (entra 8080 solo desde `sg-vpclink`; sale a todo por el NAT) · `sg-db` (entra 5432 solo desde `sg-api`). Usar `aws_vpc_security_group_ingress_rule` y `aws_vpc_security_group_egress_rule`; Terraform quita la regla de salida por defecto, así que `sg-api` necesita la suya |

Dos detalles que la IA suele olvidar:
```hcl
locals {
  nat_count = var.nat_per_az ? length(var.azs) : 1
}

# Cada subred app sale por el NAT de su zona (o por el unico NAT si nat_per_az = false)
resource "aws_route_table" "app" {
  count  = length(var.azs)
  vpc_id = aws_vpc.this.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.this[var.nat_per_az ? count.index : 0].id
  }
}

# outputs.tf: quien use estas subredes espera a que existan el NAT y las rutas
# (si no, las primeras tareas no pueden descargar la imagen de ECR).
output "app_subnet_ids" {
  value      = aws_subnet.app[*].id
  depends_on = [aws_route_table_association.app]
}
```

**Módulo `database`**

| Recurso | Detalle |
|---|---|
| `random_password` | 24 caracteres, `special = false` |
| `aws_ssm_parameter` | `/champions-gym/prod/db/password`, tipo `SecureString` (Parameter Store estándar es gratis) |
| `aws_db_subnet_group` | Las 2 subredes `db` |
| `aws_db_instance` | `engine = "postgres"`, `engine_version = "17"`, `db.t4g.micro`, 20 GB `gp3` cifrado, `db_name = "gym"`, `username = "gym_admin"`, `multi_az = var.multi_az`, `publicly_accessible = false`, `backup_retention_period = 1` (la fase 2 lo ajusta), `skip_final_snapshot = true`, `deletion_protection = false`, `apply_immediately = true` |

**Módulo `ecs_api`** (completo):

```hcl
locals {
  name = "${var.project}-${var.env}"
}

# ---------- Logs minimos de los contenedores (para ver por que falla una tarea) ----------
resource "aws_cloudwatch_log_group" "api" {
  name              = "/ecs/${local.name}-api"
  retention_in_days = var.log_retention_days
}

# ---------- IAM: rol de ejecucion (lo usa ECS) y rol de la tarea (lo usa la app) ----------
data "aws_iam_policy_document" "ecs_tasks_trust" {
  statement {
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["ecs-tasks.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "execution" {
  name               = "${local.name}-ecs-execution"
  assume_role_policy = data.aws_iam_policy_document.ecs_tasks_trust.json
}

resource "aws_iam_role_policy_attachment" "execution" {
  role       = aws_iam_role.execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

resource "aws_iam_role_policy" "execution_ssm" {
  name = "leer-password-bd"
  role = aws_iam_role.execution.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["ssm:GetParameters"]
      Resource = var.db_password_ssm_arn
    }]
  })
}

# Sin permisos por ahora. En la fase 3 se le agrega SES (correos).
resource "aws_iam_role" "task" {
  name               = "${local.name}-ecs-task"
  assume_role_policy = data.aws_iam_policy_document.ecs_tasks_trust.json
}

# ---------- Cloud Map (regional): registro de las tareas sanas ----------
resource "aws_service_discovery_private_dns_namespace" "this" {
  name = "${var.project}.local"
  vpc  = var.vpc_id
}

resource "aws_service_discovery_service" "api" {
  name          = "api"
  force_destroy = true # permite el destroy aunque queden tareas registradas

  dns_config {
    namespace_id   = aws_service_discovery_private_dns_namespace.this.id
    routing_policy = "MULTIVALUE"

    # API Gateway necesita IP + puerto: con ECS eso exige registros SRV.
    dns_records {
      type = "SRV"
      ttl  = 10
    }
  }

  # ECS le avisa a Cloud Map si cada tarea esta sana. Terraform muestra un aviso de
  # "deprecated" por failure_threshold: es normal. Sin esa linea el bloque se ignora.
  health_check_custom_config {
    failure_threshold = 1
  }
}

resource "aws_ecs_cluster" "this" {
  name = "${local.name}-cluster"
}

resource "aws_ecs_task_definition" "api" {
  family                   = "${local.name}-api"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = var.cpu    # 512 = 0,5 vCPU
  memory                   = var.memory # 1024 = 1 GB
  execution_role_arn       = aws_iam_role.execution.arn
  task_role_arn            = aws_iam_role.task.arn

  runtime_platform {
    operating_system_family = "LINUX"
    cpu_architecture        = "X86_64" # GitHub Actions construye la imagen en x86
  }

  container_definitions = jsonencode([{
    name      = "api"
    image     = "${var.ecr_repository_url}:${var.image_tag}"
    essential = true

    portMappings = [{ containerPort = var.app_port, protocol = "tcp" }]

    environment = [
      { name = "SPRING_DATASOURCE_URL", value = "jdbc:postgresql://${var.db_address}:${var.db_port}/${var.db_name}?sslmode=require" },
      { name = "SPRING_DATASOURCE_USERNAME", value = var.db_username },
      { name = "APP_VERSION", value = var.image_tag },
    ]

    secrets = [
      { name = "SPRING_DATASOURCE_PASSWORD", valueFrom = var.db_password_ssm_arn },
    ]

    # liveness no revisa la BD: si RDS cambia de nodo, ECS no mata las tareas.
    healthCheck = {
      command     = ["CMD-SHELL", "curl -f http://localhost:${var.app_port}/api/actuator/health/liveness || exit 1"]
      interval    = 30
      timeout     = 5
      retries     = 3
      startPeriod = 90
    }

    stopTimeout = 30

    logConfiguration = {
      logDriver = "awslogs"
      options = {
        awslogs-group         = aws_cloudwatch_log_group.api.name
        awslogs-region        = var.aws_region
        awslogs-stream-prefix = "api"
      }
    }
  }])
}

resource "aws_ecs_service" "api" {
  name            = "${local.name}-api"
  cluster         = aws_ecs_cluster.this.id
  task_definition = aws_ecs_task_definition.api.arn
  desired_count   = var.desired_count # 2, una por zona
  launch_type     = "FARGATE"

  # Despliegue sin corte: primero suben las tareas nuevas, luego se apagan las viejas.
  deployment_minimum_healthy_percent = 100
  deployment_maximum_percent         = 200

  # Si la version nueva no arranca, ECS vuelve sola a la anterior.
  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }

  availability_zone_rebalancing = "ENABLED"

  network_configuration {
    subnets          = var.app_subnet_ids # privadas 10.0.3.0/24 y 10.0.4.0/24
    security_groups  = [var.sg_api_id]
    assign_public_ip = false
  }

  service_registries {
    registry_arn   = aws_service_discovery_service.api.arn
    container_name = "api"
    container_port = var.app_port
  }

  # La fase 2 agrega el autoescalado: desde ahi el numero de tareas lo maneja AWS.
  lifecycle {
    ignore_changes = [desired_count]
  }

  depends_on = [
    aws_iam_role_policy_attachment.execution,
    aws_iam_role_policy.execution_ssm,
  ]
}
```

**Módulo `api_gateway`** (completo):
```hcl
locals {
  name = "${var.project}-${var.env}"
}

# VPC Link (regional, en el borde de la VPC): crea sus interfaces en las subredes de las tareas.
resource "aws_apigatewayv2_vpc_link" "this" {
  name               = "${local.name}-vpclink"
  subnet_ids         = var.app_subnet_ids
  security_group_ids = [var.sg_vpclink_id]
}

resource "aws_apigatewayv2_api" "this" {
  name          = "${local.name}-http-api"
  protocol_type = "HTTP"
}

# API Gateway le pregunta a Cloud Map que tareas estan sanas y les reparte las peticiones.
resource "aws_apigatewayv2_integration" "ecs" {
  api_id                 = aws_apigatewayv2_api.this.id
  integration_type       = "HTTP_PROXY"
  integration_method     = "ANY"
  integration_uri        = var.cloudmap_service_arn
  connection_type        = "VPC_LINK"
  connection_id          = aws_apigatewayv2_vpc_link.this.id
  payload_format_version = "1.0"
}

resource "aws_apigatewayv2_route" "api" {
  api_id    = aws_apigatewayv2_api.this.id
  route_key = "ANY /api/{proxy+}"
  target    = "integrations/${aws_apigatewayv2_integration.ecs.id}"
}

# $default: la ruta llega igual a Spring Boot (/api/...), sin el nombre del stage.
resource "aws_apigatewayv2_stage" "default" {
  api_id      = aws_apigatewayv2_api.this.id
  name        = "$default"
  auto_deploy = true
}
```

**Probar desde la computadora del Integrante 1** (solo esta vez, porque el pipeline todavía no existe):
```bash
export AWS_REGION=sa-east-1
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGISTRY=$ACCOUNT_ID.dkr.ecr.sa-east-1.amazonaws.com

# 1) Subir una primera imagen a ECR (en Mac con chip M agregar --platform linux/amd64)
aws ecr get-login-password | docker login --username AWS --password-stdin $REGISTRY
docker build -t $REGISTRY/champions-gym-api:manual-1 app
docker push $REGISTRY/champions-gym-api:manual-1

# 2) Crear la infraestructura (RDS Multi-AZ puede tardar 15 a 20 minutos)
cd infra
terraform init -backend-config=envs/prod.backend.hcl
terraform plan  -var-file=envs/prod.tfvars -var="image_tag=manual-1"
terraform apply -var-file=envs/prod.tfvars -var="image_tag=manual-1"

# 3) Probar
API=$(terraform output -raw api_endpoint)
curl $API/api/sedes
curl $API/api/version        # {"version":"manual-1"}
```

| # | Commit |
|---|---|
| 1 | `feat(core): add terraform, providers, backend and prod environment` |
| 2 | `feat(network): add vpc, subnets, internet gateway and nat gateways` |
| 3 | `feat(network): add security groups for vpc link, api and database` |
| 4 | `feat(database): add rds postgresql with password in parameter store` |
| 5 | `feat(ecs): add cluster, task definition, service and cloud map` |
| 6 | `feat(api-gateway): add http api with vpc link to cloud map` |
| 7 | `feat(core): wire modules and add outputs` |

**Listo cuando:** `curl $API/api/sedes` responde las 2 sedes; ECS muestra 2 tareas `RUNNING`, una en `sa-east-1a` y otra en `sa-east-1b`; Cloud Map muestra 2 instancias sanas.
**Evidencia (RNF 4):** captura del servicio con las 2 tareas, su zona y su subred.

### Paso 1.5 · GitHub Actions (rama `feature/ci-cd`)

1. **Variables del repo** (Settings → Secrets and variables → Actions → pestaña *Variables*). No hay claves de AWS en GitHub: los workflows entran con OIDC.

   | Variable | Valor |
   |---|---|
   | `AWS_REGION` | `sa-east-1` |
   | `AWS_DEPLOY_ROLE_ARN` | salida `github_deploy_role_arn` del bootstrap |
   | `AWS_PLAN_ROLE_ARN` | salida `github_plan_role_arn` del bootstrap |
   | `ECR_REPOSITORY` | `champions-gym-api` |

2. **`.github/workflows/ci.yml`**: corre en cada PR hacia `develop` o `main`.
```yaml
name: ci

on:
  pull_request:
    branches: [develop, main]

permissions:
  contents: read
  id-token: write

jobs:
  app:
    name: app
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_DB: gym
          POSTGRES_USER: gym_admin
          POSTGRES_PASSWORD: local123
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U gym_admin -d gym"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10
    env:
      SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/gym
      SPRING_DATASOURCE_USERNAME: gym_admin
      SPRING_DATASOURCE_PASSWORD: local123
    steps:
      - uses: actions/checkout@v7

      - uses: actions/setup-java@v6
        with:
          distribution: temurin
          java-version: "21"
          cache: maven

      - name: Tests de la API
        working-directory: app
        run: mvn -B verify

      - name: Construir la imagen (sin subirla)
        run: docker build -t champions-gym-api:ci app

  terraform:
    name: terraform
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infra
    steps:
      - uses: actions/checkout@v7

      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: 1.14.8

      - name: Formato
        run: terraform fmt -check -recursive

      - name: Validar bootstrap
        working-directory: infra/bootstrap
        run: |
          terraform init -backend=false -input=false
          terraform validate

      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: ${{ vars.AWS_PLAN_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Init
        run: terraform init -input=false -backend-config=envs/prod.backend.hcl

      - name: Validar
        run: terraform validate

      - name: Plan (solo lectura, no cambia nada)
        run: terraform plan -input=false -lock=false -var-file=envs/prod.tfvars -var="image_tag=${{ github.event.pull_request.head.sha }}"
```

3. **`.github/workflows/deploy.yml`**: corre en cada merge a `main`, y también a mano (sirve para "encender" después de apagar).
```yaml
name: deploy

on:
  push:
    branches: [main]
  workflow_dispatch: # boton "Run workflow" = encender de nuevo despues de apagar

permissions:
  contents: read
  id-token: write

concurrency:
  group: aws-prod
  cancel-in-progress: false

env:
  AWS_REGION: ${{ vars.AWS_REGION }}
  ECR_REPOSITORY: ${{ vars.ECR_REPOSITORY }}
  IMAGE_TAG: ${{ github.sha }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 40
    steps:
      - uses: actions/checkout@v7

      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Login en ECR
        id: ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Construir y subir la imagen
        env:
          REGISTRY: ${{ steps.ecr.outputs.registry }}
        run: |
          if aws ecr describe-images --repository-name "$ECR_REPOSITORY" --image-ids imageTag="$IMAGE_TAG" > /dev/null 2>&1; then
            echo "La imagen $IMAGE_TAG ya existe en ECR, no se vuelve a construir."
          else
            docker build -t "$REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" app
            docker push "$REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"
          fi

      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: 1.14.8
          terraform_wrapper: false

      - name: Terraform apply
        working-directory: infra
        run: |
          terraform init -input=false -backend-config=envs/prod.backend.hcl
          terraform apply -input=false -auto-approve -var-file=envs/prod.tfvars -var="image_tag=$IMAGE_TAG"
          echo "API_URL=$(terraform output -raw api_endpoint)" >> "$GITHUB_ENV"

      - name: Esperar a que responda la nueva version (max. 10 min)
        run: |
          for i in $(seq 1 40); do
            v=$(curl -s --max-time 5 "$API_URL/api/version" | jq -r '.version' 2>/dev/null || true)
            echo "Intento $i: version en linea = ${v:-ninguna}"
            if [ "$v" = "$IMAGE_TAG" ]; then
              echo "Desplegado en $API_URL"
              exit 0
            fi
            sleep 15
          done
          echo "::error::La version $IMAGE_TAG no quedo en linea. Revisa el servicio en ECS: el circuit breaker pudo volver a la version anterior."
          exit 1
```

4. **`.github/workflows/apagar.yml`**: botón manual para destruir todo menos el bootstrap.
```yaml
name: apagar

on:
  workflow_dispatch:
    inputs:
      confirmar:
        description: "Escribe APAGAR para destruir todo (el bootstrap y las imagenes se quedan)"
        required: true
        type: string

permissions:
  contents: read
  id-token: write

concurrency:
  group: aws-prod
  cancel-in-progress: false

jobs:
  destroy:
    if: ${{ inputs.confirmar == 'APAGAR' }}
    runs-on: ubuntu-latest
    timeout-minutes: 45
    defaults:
      run:
        working-directory: infra
    steps:
      - uses: actions/checkout@v7

      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: 1.14.8

      - name: Terraform destroy
        run: |
          terraform init -input=false -backend-config=envs/prod.backend.hcl
          terraform destroy -input=false -auto-approve -var-file=envs/prod.tfvars -var="image_tag=apagado"
```

**Cómo funciona la actualización continua (para el informe, RNF 2):**
1. Un integrante hace merge a `main`.
2. GitHub Actions entra a AWS con OIDC (rol `deploy`, sin claves guardadas).
3. Construye la imagen y la sube a ECR con el SHA del commit como tag.
4. `terraform apply` crea una nueva revisión de la tarea con ese tag.
5. ECS levanta las tareas nuevas, espera a que estén sanas en Cloud Map y recién apaga las viejas: no hay corte.
6. Si la versión nueva no arranca, el circuit breaker vuelve solo a la anterior y el workflow termina en rojo.
7. El workflow espera hasta que `/api/version` responda el SHA nuevo (máximo 10 minutos).

| # | Commit |
|---|---|
| 1 | `ci: add pull request checks for app and terraform` |
| 2 | `ci: add deploy workflow to ecr and ecs on main` |
| 3 | `ci: add manual workflow to destroy the environment` |

**Listo cuando:** en el PR hacia `develop` salen en verde los checks `app` y `terraform`, y dentro de `terraform` se ve el plan.

### Paso 1.6 · Primer despliegue automático y cierre de la fase 1

1. **Primer release:** PR `develop → main` con el título "Release v0.1.0" → merge → pestaña Actions → `deploy` en verde. `curl $API/api/version` devuelve el SHA del commit.
2. **Prueba de actualización sin corte (evidencia del RNF 2):** en otra terminal deja corriendo
   `while true; do curl -s -o /dev/null -w "%{http_code} " $API/api/sedes; sleep 1; done`.
   Haz un cambio pequeño (por ejemplo, agregar `"estado": "ok"` a la respuesta de `/version`), llévalo hasta `main` y mide el tiempo desde el merge hasta que responde la versión nueva (meta: menos de 15 minutos, sin códigos 5xx en el bucle).
3. **Tag:**
   ```bash
   git checkout main && git pull
   git tag -a v0.1.0 -m "Base desplegada con CI/CD"
   git push origin v0.1.0
   ```
4. **Proteger las ramas** (Settings → Branches o Rules → Rulesets): `main` y `develop` con PR obligatorio, 1 aprobación y los checks `app` y `terraform`. Si el repo es privado y no tienen GitHub Pro (gratis con el GitHub Student Developer Pack), GitHub no deja activarlo; en ese caso la regla queda de palabra.
5. **Accesos de solo lectura para los Integrantes 2 y 3 (RNF 1):** agregar `infra/bootstrap/equipo.tf`, ejecutar `terraform apply` en `infra/bootstrap` y pasarle a cada uno su usuario, la URL de la consola y su contraseña inicial (`terraform output -json passwords_iniciales`). La cambian al entrar.
   ```hcl
   # Accesos del equipo (RNF 1): los demas integrantes solo pueden mirar.
   variable "team_members" {
     description = "Integrantes con acceso de solo lectura a la consola."
     type        = list(string)
     default     = ["piero", "joaquin"]
   }

   resource "aws_iam_group" "solo_lectura" {
     name = "${var.project}-solo-lectura"
   }

   resource "aws_iam_group_policy_attachment" "solo_lectura" {
     for_each = toset([
       "arn:aws:iam::aws:policy/ReadOnlyAccess",
       "arn:aws:iam::aws:policy/IAMUserChangePassword",
     ])

     group      = aws_iam_group.solo_lectura.name
     policy_arn = each.value
   }

   resource "aws_iam_user" "equipo" {
     for_each = toset(var.team_members)

     name          = "${var.project}-${each.key}"
     force_destroy = true
   }

   resource "aws_iam_user_group_membership" "equipo" {
     for_each = aws_iam_user.equipo

     user   = each.value.name
     groups = [aws_iam_group.solo_lectura.name]
   }

   resource "aws_iam_user_login_profile" "equipo" {
     for_each = aws_iam_user.equipo

     user                    = each.value.name
     password_reset_required = true
   }

   output "consola_url" {
     value = "https://${local.account_id}.signin.aws.amazon.com/console"
   }

   output "passwords_iniciales" {
     value     = { for k, v in aws_iam_user_login_profile.equipo : k => v.password }
     sensitive = true
   }
   ```
   Evidencia del RNF 1: Piero intenta detener una tarea desde la consola → `AccessDenied`, y el intento aparece en CloudTrail → Event history (viene activado y es gratis).
6. **Actualizar "Estado actual" en `AGENTS.md`** (rama `docs/estado-fase-1`, commit `docs: mark phase 1 as done in ai context`).
7. **Apagar** si nadie va a trabajar en las próximas horas (sección 8).

---

## 6. Fase 2: los 3 en paralelo (completar el diagrama)

| Tarea | Quién | Rama | RNF | Toca | Depende de |
|---|---|---|---|---|---|
| **I2-1** Web en S3 + CloudFront | Integrante 2 | `feature/frontend` | 7, 12 | `modules/frontend` (nuevo), `frontend/`, `deploy.yml` | Fase 1 |
| **I2-2** Entrar solo por CloudFront | Integrante 2 | `feature/origin-verify` | 7 | `app` (filtro), `modules/ecs_api` (1 variable) | I2-1 |
| **I2-3** Autoescalado de 2 a 10 | Integrante 2 | `feature/autoscaling` | 10, 11 | `modules/autoscaling` (nuevo) | Fase 1 |
| **I2-4** Rollback y circuit breaker | Integrante 2 | `feature/rollback` | 9, 8 | `.github/workflows/rollback.yml` | Fase 1 |
| **I3-1** Respaldos y failover de la BD | Integrante 3 | `feature/database-ha` | 13, 15 | `modules/database`, `app` (error 503) | Fase 1 |
| **I3-2** Límite de peticiones | Integrante 3 | `feature/api-limits` | 18 | `modules/api_gateway`, `frontend/` (mensaje) | I2-1 (para el mensaje) |
| **I3-3** WAF en CloudFront | Integrante 3 | `feature/waf` | 16 | `modules/waf` (nuevo), `infra/main.tf` | I2-1 |
| **I1-1** Pruebas de despliegue sin corte y caída de zona | Integrante 1 | `docs/evidencias-fase-2` | 2, 4 | README (evidencias) | Fase 1 |
| **I1-2** Revisar PRs, releases y comandos con permiso de escritura | Integrante 1 | (sin rama) | 1 | (nada) | Fase 1 |

Orden sugerido de merge a `develop`: I2-1, I3-1 e I2-3 pueden ir a la vez (son módulos distintos). Después I2-2, I3-2 e I3-3, que dependen de CloudFront. I2-4 puede ir en cualquier momento.
Las pruebas que **modifican** recursos (detener una tarea, forzar el failover) las ejecuta el Integrante 1, porque es el único con permiso de escritura.

### I2-1 · Web en S3 + CloudFront (RNF 7 y 12)

- **Contrato del módulo `frontend`:** entradas `project`, `env`, `api_endpoint` (salida de `api_gateway`) y `web_acl_arn` (default `null`). Salidas `cloudfront_domain`, `cloudfront_distribution_id`, `frontend_bucket` y `origin_verify_secret` (sensitive).
- Bucket S3 privado (`force_destroy = true` y bloqueo de acceso público), Origin Access Control y una política del bucket que solo deja leer a `cloudfront.amazonaws.com` con la condición `AWS:SourceArn` = ARN de la distribución.
- `frontend/index.html` + `app.js`: una página simple que muestra las sedes con `fetch('/api/sedes')`.
- Lo que la IA suele equivocar en CloudFront:
  ```hcl
  data "aws_cloudfront_cache_policy" "optimized" {
    name = "Managed-CachingOptimized"
  }

  data "aws_cloudfront_cache_policy" "disabled" {
    name = "Managed-CachingDisabled"
  }

  # API Gateway exige su propio Host: CloudFront no debe reenviar el del usuario.
  data "aws_cloudfront_origin_request_policy" "all_viewer_except_host" {
    name = "Managed-AllViewerExceptHostHeader"
  }

  resource "random_password" "origin_verify" {
    length  = 32
    special = false
  }

  resource "aws_cloudfront_distribution" "this" {
    enabled             = true
    default_root_object = "index.html"
    price_class         = "PriceClass_All" # la unica que incluye ubicaciones en Sudamerica
    web_acl_id          = var.web_acl_arn

    origin {
      origin_id                = "s3-web"
      domain_name              = aws_s3_bucket.web.bucket_regional_domain_name
      origin_access_control_id = aws_cloudfront_origin_access_control.web.id
    }

    origin {
      origin_id   = "api-gateway"
      domain_name = replace(var.api_endpoint, "https://", "")

      custom_origin_config {
        http_port              = 80
        https_port             = 443
        origin_protocol_policy = "https-only"
        origin_ssl_protocols   = ["TLSv1.2"]
      }

      custom_header {
        name  = "X-Origin-Verify"
        value = random_password.origin_verify.result
      }
    }

    default_cache_behavior {
      target_origin_id       = "s3-web"
      viewer_protocol_policy = "redirect-to-https"
      allowed_methods        = ["GET", "HEAD"]
      cached_methods         = ["GET", "HEAD"]
      cache_policy_id        = data.aws_cloudfront_cache_policy.optimized.id
      compress               = true
    }

    ordered_cache_behavior {
      path_pattern             = "/api/*"
      target_origin_id         = "api-gateway"
      viewer_protocol_policy   = "https-only"
      allowed_methods          = ["GET", "HEAD", "OPTIONS", "PUT", "POST", "PATCH", "DELETE"]
      cached_methods           = ["GET", "HEAD"]
      cache_policy_id          = data.aws_cloudfront_cache_policy.disabled.id
      origin_request_policy_id = data.aws_cloudfront_origin_request_policy.all_viewer_except_host.id
    }

    restrictions {
      geo_restriction {
        restriction_type = "none"
      }
    }

    viewer_certificate {
      cloudfront_default_certificate = true
    }
  }
  ```
- **En `deploy.yml`:** el paso "Terraform apply" pasa a guardar la URL de CloudFront (reemplaza la línea de `api_endpoint`), y justo después se agrega el paso "Publicar la web". Desde aquí la prueba de versión entra por CloudFront.
  ```yaml
          terraform apply -input=false -auto-approve -var-file=envs/prod.tfvars -var="image_tag=$IMAGE_TAG"
          echo "API_URL=https://$(terraform output -raw cloudfront_domain)" >> "$GITHUB_ENV"
          echo "WEB_BUCKET=$(terraform output -raw frontend_bucket)" >> "$GITHUB_ENV"
          echo "DISTRIBUTION_ID=$(terraform output -raw cloudfront_distribution_id)" >> "$GITHUB_ENV"

      - name: Publicar la web
        run: |
          aws s3 sync frontend "s3://$WEB_BUCKET" --delete
          aws cloudfront create-invalidation --distribution-id "$DISTRIBUTION_ID" --paths "/*"
  ```
  Agregar `cloudfront_domain`, `frontend_bucket` y `cloudfront_distribution_id` a `infra/outputs.tf`.
- **Listo cuando:** `https://<dominio>.cloudfront.net` muestra la página con las 2 sedes, y la URL directa del bucket responde `AccessDenied`.
- **Evidencia del RNF 12:** prueba de carga de unas 1 000 peticiones por minuto durante 15 minutos contra la página (con [hey](https://github.com/rakyll/hey) o k6), con 99 % o más de respuestas 200.

### I2-2 · Entrar solo por CloudFront (RNF 7)

- En `infra/main.tf`, pasar `origin_verify_secret = module.frontend.origin_verify_secret` al módulo `ecs_api`. En `ecs_api`, nueva variable `origin_verify_secret` (string, `sensitive`, default `""`) que se pasa al contenedor como `ORIGIN_VERIFY_SECRET`.
- En la app, un filtro que rechaza con 403 lo que no traiga el header correcto. **No debe filtrar `/api/actuator/`**: el health check de ECS llama directo, sin CloudFront.
  ```java
  @Component
  public class OriginVerifyFilter extends OncePerRequestFilter {

      private final String secreto;

      public OriginVerifyFilter(@Value("${ORIGIN_VERIFY_SECRET:}") String secreto) {
          this.secreto = secreto;
      }

      @Override
      protected boolean shouldNotFilter(HttpServletRequest request) {
          return secreto.isBlank() || request.getRequestURI().startsWith("/api/actuator/");
      }

      @Override
      protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                      FilterChain chain) throws ServletException, IOException {
          if (!secreto.equals(request.getHeader("X-Origin-Verify"))) {
              response.setStatus(HttpServletResponse.SC_FORBIDDEN);
              response.setContentType("application/json");
              response.getWriter().write("{\"mensaje\":\"Ingresa desde la web del gimnasio\"}");
              return;
          }
          chain.doFilter(request, response);
      }
  }
  ```
- **Listo cuando:** `curl <url de API Gateway>/api/sedes` → 403, y `curl https://<cloudfront>/api/sedes` → 200.

### I2-3 · Autoescalado de 2 a 10 tareas (RNF 10 y 11)

```hcl
# modules/autoscaling/main.tf (entradas: project, env, cluster_name, service_name)
resource "aws_appautoscaling_target" "ecs" {
  service_namespace  = "ecs"
  resource_id        = "service/${var.cluster_name}/${var.service_name}"
  scalable_dimension = "ecs:service:DesiredCount"
  min_capacity       = 2
  max_capacity       = 10
}

resource "aws_appautoscaling_policy" "cpu" {
  name               = "${var.project}-${var.env}-cpu-70"
  policy_type        = "TargetTrackingScaling"
  service_namespace  = aws_appautoscaling_target.ecs.service_namespace
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs.scalable_dimension

  target_tracking_scaling_policy_configuration {
    target_value       = 70
    scale_out_cooldown = 60
    scale_in_cooldown  = 300

    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
  }
}
```
- En la raíz: `module "autoscaling"` con `cluster_name = module.ecs_api.cluster_name` y `service_name = module.ecs_api.service_name`. El servicio ya tiene `ignore_changes = [desired_count]` desde la fase 1.
- Sobre "durante 5 minutos": AWS sube tareas cuando la CPU pasa el 70 % durante 3 minutos seguidos, así que el RNF se cumple con margen.
- **Listo cuando:** con una prueba de carga (por ejemplo, `hey -z 10m -c 50 https://<cloudfront>/api/sedes`) el servicio pasa de 2 tareas y luego vuelve a 2.
- **Evidencia:** captura de los eventos del servicio en ECS y de la política de escalado.

### I2-4 · Rollback y circuit breaker (RNF 9 y 8)

- `rollback.yml`: copia de `deploy.yml` con un input `image_tag` (obligatorio), sin el paso de construir la imagen y con `IMAGE_TAG: ${{ inputs.image_tag }}`. Se ejecuta a mano desde la rama `main`.
- **Prueba 1, rollback manual (RNF 9):** ejecutar `rollback` con el SHA de la versión anterior (está en los tags de ECR). Medir el tiempo hasta que `/api/version` muestra ese SHA (meta: menos de 10 minutos).
- **Prueba 2, circuit breaker:** ejecutar `rollback` con `image_tag = no-existe`. ECS no puede iniciar las tareas, vuelve solo a la versión anterior y la API sigue respondiendo; el workflow termina en rojo, como se espera. Después ejecutar `deploy` desde `main` para dejar todo como estaba.
- **Prueba 3, caída de una tarea (RNF 8):** con el bucle de `curl` corriendo, el Integrante 1 detiene una tarea (`aws ecs stop-task`). Contar los errores (meta: menos de 1 %) y el tiempo hasta que ECS levanta la reemplazante.

### I3-1 · Respaldos y failover de la BD (RNF 13 y 15)

```hcl
# modules/database/main.tf, dentro de aws_db_instance (reemplaza backup_retention_period = 1)
  backup_retention_period = 7
  backup_window           = "08:00-08:45"         # 3:00 a 3:45 a. m. hora de Peru (UTC-5)
  maintenance_window      = "sun:09:00-sun:10:00" # domingo 4:00 a 5:00 a. m. hora de Peru
  copy_tags_to_snapshot   = true
```
- En la app, un `@RestControllerAdvice` que responda **503** con `{"mensaje":"Servicio no disponible, intenta en unos minutos"}` cuando no hay conexión con la BD (`CannotCreateTransactionException` y `DataAccessResourceFailureException`). El `connection-timeout: 3000` del `application.yml` hace que falle rápido.
- **Prueba de failover (RNF 13):** requiere `db_multi_az = true`. Con un bucle de `curl` que imprima la hora y el código, el Integrante 1 ejecuta
  `aws rds reboot-db-instance --db-instance-identifier champions-gym-prod-db --force-failover`.
  Medir cuánto duran los 503 (meta: menos de 2 minutos) y comprobar que la API no se queda colgada.
- **RNF 15:** en la consola de RDS se ve la ventana de respaldo y los snapshots automáticos diarios. Para avisar por correo cuando un respaldo falla haría falta SNS, que no está en el diagrama: lo tiene que decidir el equipo (agregarlo o ajustar el texto del RNF).

### I3-2 · Límite de peticiones en API Gateway (RNF 18)

```hcl
# modules/api_gateway/main.tf, dentro de aws_apigatewayv2_stage "default"
  default_route_settings {
    throttling_rate_limit  = 500 # peticiones por segundo (RNF 14)
    throttling_burst_limit = 500 # conexiones simultaneas (RNF 18)
  }
```
- Cuando se pasa el límite, API Gateway responde 429. En `frontend/app.js`, ante un 429 mostrar "Hay muchas personas conectadas, intenta en unos segundos" (coordinar con el Integrante 2).
- **Prueba:** carga alta contra `/api/sedes` → aparecen 429 y los errores 5xx quedan por debajo de 0,1 %.

### I3-3 · WAF en CloudFront (RNF 16)

- En la raíz, el WAF usa el proveedor de us-east-1 y se conecta al frontend:
  ```hcl
  module "waf" {
    source = "./modules/waf"
    providers = {
      aws = aws.us_east_1
    }

    project = var.project
    env     = var.env
  }
  # y en module "frontend": web_acl_arn = module.waf.web_acl_arn
  ```
- Regla principal del módulo (salida: `web_acl_arn`). Además, agregar el grupo administrado `AWSManagedRulesCommonRuleSet` con `override_action { none {} }`.
  ```hcl
  resource "aws_wafv2_web_acl" "this" {
    name  = "${local.name}-waf"
    scope = "CLOUDFRONT"

    default_action {
      allow {}
    }

    rule {
      name     = "limite-por-ip"
      priority = 1

      action {
        block {}
      }

      statement {
        rate_based_statement {
          limit                 = 2000 # solicitudes por IP...
          aggregate_key_type    = "IP"
          evaluation_window_sec = 300 # ...en 5 minutos
        }
      }

      visibility_config {
        cloudwatch_metrics_enabled = true
        metric_name                = "limite-por-ip"
        sampled_requests_enabled   = true
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "${local.name}-waf"
      sampled_requests_enabled   = true
    }
  }
  ```
- Costo: unos US$7 al mes, que AWS cobra por hora (US$5 la ACL y US$1 por regla).
- **Prueba:** desde una PC, más de 2 000 peticiones en menos de 5 minutos (por ejemplo, `hey -n 3000 -c 20 https://<cloudfront>/`) → después del límite responde 403. Desde otra red (datos del celular) la página sigue funcionando.

### Cierre de la fase 2

PR `develop → main` "Release v0.2.0" → deploy en verde → tag `v0.2.0` → actualizar "Estado actual" en `AGENTS.md`.

---

## 7. Fase 3: funcionalidades

| Tarea | Quién | Rama | RNF | Qué incluye |
|---|---|---|---|---|
| Socios y suscripciones | Integrante 2 | `feature/socios-suscripciones` | 7, 8 | Tablas `socio`, `plan` y `suscripcion`; alta, consulta y renovación; páginas en `frontend/` |
| Aforo y reservas | Integrante 3 | `feature/aforo-reservas` | 14 | Ingreso y salida por sede, aforo actual (`GET /api/sedes/{id}/aforo`), reservas; prueba de carga con p95 menor a 1,5 s |
| Dominio (opcional) | Integrante 3 | `feature/dominio` | 17 | Route 53 + certificado ACM en us-east-1 + alias en CloudFront. Requiere comprar un dominio |
| Logs de actividad | Integrante 1 | `feature/logs` | 6 | Retención de 30 días, logs de acceso de API Gateway, logs de PostgreSQL y de inicios de sesión en la app |
| Correos de confirmación | Integrante 1 | `feature/correos-ses` | 5 | Identidad en SES, permiso `ses:SendEmail` en el rol de la tarea, módulo `notificaciones`. Para enviar 14 correos por segundo hay que pedir salir del sandbox de SES |
| Pagos con Mercado Pago | Integrante 1 | `feature/pagos-mercadopago` | 3 | Puerto `PasarelaPagos` + adaptador de Mercado Pago (credenciales de prueba en Parameter Store). El cobro sale por NAT → Internet Gateway. El webhook entra por `https://<cloudfront>/api/pagos/webhook`. Cada pago se registra una sola vez (el id de pago es único en la tabla) |

- Orden: socios y suscripciones va antes que correos y pagos, porque los dos los usan.
- Al terminar: release `v1.0.0`, evidencias en el README y apagar.

---

## 8. Apagar y encender (costos)

Precios de la lista de AWS para sa-east-1 (consultada el 2 de noviembre de 2026):

| Recurso | Precio | Como el diagrama | Modo ahorro (`nat_per_az = false`, `db_multi_az = false`) |
|---|---|---|---|
| NAT Gateway | US$0,093 por hora cada uno + US$0,093 por GB | 2 → US$0,186/h | 1 → US$0,093/h |
| IP pública de cada NAT | US$0,005 por hora | US$0,010/h | US$0,005/h |
| Fargate (0,5 vCPU + 1 GB) | US$0,0424 por hora cada tarea | 2 → US$0,085/h | US$0,085/h |
| RDS db.t4g.micro + 20 GB gp3 | Multi-AZ US$0,069/h · sin standby US$0,034/h | US$0,081/h | US$0,040/h |
| API Gateway, Cloud Map, ECR, S3, logs | por uso | centavos | centavos |
| **Total** | | **≈ US$0,36/h (US$8,70 por día)** | **≈ US$0,22/h (US$5,30 por día)** |

La fase 2 suma el WAF (unos US$0,01 por hora) y CloudFront (casi gratis con poco tráfico).

- **Apagar:** Actions → `apagar` → Run workflow (rama `main`) → escribir `APAGAR`. Borra todo menos el bootstrap (bucket del estado, ECR y roles, que cuestan centavos). **Se pierden los datos de la BD**; al encender, Flyway vuelve a crear las tablas y las 2 sedes.
- **Encender:** Actions → `deploy` → Run workflow (rama `main`). Como la imagen ya existe en ECR, no se vuelve a construir.
- **Mientras desarrollan** pueden poner `nat_per_az = false` y `db_multi_az = false` en `envs/prod.tfvars` (por PR) y volverlos a `true` antes de sacar las evidencias.
- **Al terminar el curso:** apagar, luego `cd infra/bootstrap && terraform destroy` desde la computadora del Integrante 1, y revisar en la consola que no quede nada (VPC, NAT, IP elásticas, RDS, CloudFront).

---

## 9. Problemas comunes

| Síntoma | Causa probable | Solución |
|---|---|---|
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | El nombre del repo en `github_repo` no coincide (mayúsculas), el workflow corrió desde otra rama que no es `main`, o se agregó `environment:` al job | Corregir `github_repo` y aplicar el bootstrap; ejecutar desde `main`; no usar `environment:` |
| `EntityAlreadyExists` en el proveedor OIDC | La cuenta ya tenía uno | `terraform import` (paso 1.3) |
| La tarea se detiene con `CannotPullContainerError` | No hay salida por NAT, o el tag no existe en ECR | Revisar rutas `app` y NAT; revisar el tag en ECR |
| La tarea muere con `exec format error` | Imagen construida en una Mac con chip M | `docker build --platform linux/amd64 ...` |
| La tarea arranca y se reinicia | Error en la app (contraseña, SSL, Flyway, ruta del health check) | Ver los logs (fila de abajo) |
| Ver logs desde Git Bash | Git Bash convierte las rutas que empiezan con `/` | `MSYS_NO_PATHCONV=1 aws logs tail /ecs/champions-gym-prod-api --follow --region sa-east-1` |
| API Gateway responde `503 Service Unavailable` | No hay tareas sanas en Cloud Map, o falta la regla `sg-vpclink → sg-api :8080` | Ver los eventos del servicio en ECS y los grupos de seguridad |
| API Gateway responde `404 Not Found` | La ruta no empieza con `/api/` | Usar `/api/...` |
| `Error acquiring the state lock` | Hay otro `apply` corriendo (el pipeline) | Esperar. `terraform force-unlock <ID>` solo si están seguros de que nadie más está aplicando |
| CI: falla `contextLoads` en `mvn verify` | El test no encuentra PostgreSQL | El job `app` ya trae el servicio `postgres:17`; revisar que el nombre de la BD y el usuario coincidan |
| `./mvnw: Permission denied` | Git en Windows perdió el permiso de ejecución | `git update-index --chmod=+x app/mvnw` y commit |
| CI: el lock de providers no coincide | El `.terraform.lock.hcl` se generó solo para Windows | `terraform providers lock -platform=linux_amd64 -platform=windows_amd64` y commit |
| El plan del PR falla con `AccessDenied` | Al rol `plan` (solo lectura) le falta algún permiso de lectura | Avisar al Integrante 1 para ajustarlo en el bootstrap |
| `deploy` termina en rojo en "Esperar a que responda la nueva version" | El circuit breaker volvió a la versión anterior | Ver los logs y eventos del servicio, corregir y volver a hacer el release |
| `apagar` falla en subredes o VPC (`DependencyViolation`) | AWS todavía no libera las interfaces del VPC Link | Esperar unos minutos y volver a ejecutar `apagar` |
| Un servicio aparece bloqueado en la cuenta | Restricción del plan gratuito | Pasar la cuenta al plan de pago (sección 1) |

---

## 10. Darle contexto a tu IA

- **Si tu IA lee el repositorio** (Claude Code, GitHub Copilot, Cursor, Codex): ya lee `AGENTS.md` (Claude Code lo lee a través de `CLAUDE.md`, que contiene `@AGENTS.md`). Solo dale tu prompt.
- **Si es un chat** (ChatGPT, Gemini, Claude.ai): pega primero `AGENTS.md` completo, luego la sección de tu tarea de esta guía y al final tu prompt.
- Después de cada merge, quien cerró la tarea marca su casilla en "Estado actual" de `AGENTS.md`, así la IA de los demás sabe qué ya existe.

**Integrante 1 (Fabrizio)**
```text
Eres mi asistente para el proyecto Champion's Gym (curso Infraestructura como Código, UPAO).
Soy el Integrante 1 y me toca la fase 1: levantar la base en AWS y GitHub Actions yo solo.
Lee AGENTS.md y la sección 5 de docs/GUIA-IMPLEMENTACION.md. Hoy voy a hacer el paso <1.x>.
Antes de escribir código, dime qué archivos vas a crear o cambiar y espera mi OK.
Respeta los nombres, versiones, CIDR y contratos de módulos que ya están en AGENTS.md:
si algo no cuadra, avísame en lugar de inventar. Uso <Windows con Git Bash / Mac / Linux>.
Al terminar, dame los commits en el orden de la guía y qué debo probar para dar el paso por terminado.
```

**Integrante 2 (Piero)**
```text
Eres mi asistente para el proyecto Champion's Gym (curso Infraestructura como Código, UPAO).
Soy el Integrante 2: mis RNF son del 7 al 12. La fase 1 ya está desplegada (revisa "Estado actual" en AGENTS.md).
Lee AGENTS.md y la sección 6 de docs/GUIA-IMPLEMENTACION.md. Mi tarea es <I2-1 / I2-2 / I2-3 / I2-4>.
No tengo permisos de escritura en AWS: pruebo en local (docker compose, terraform fmt y validate)
y mi cambio llega a AWS por PR. No ejecutes terraform apply.
Antes de escribir código, dime qué archivos vas a crear o cambiar y espera mi OK.
No cambies los contratos de módulos de otros integrantes; si necesito uno nuevo, avísame para coordinarlo.
Al terminar, dame los commits, la descripción del PR y cómo sacar la evidencia de mi RNF.
```

**Integrante 3 (Joaquín)**
```text
Eres mi asistente para el proyecto Champion's Gym (curso Infraestructura como Código, UPAO).
Soy el Integrante 3: mis RNF son del 13 al 18. La fase 1 ya está desplegada (revisa "Estado actual" en AGENTS.md).
Lee AGENTS.md y la sección 6 de docs/GUIA-IMPLEMENTACION.md. Mi tarea es <I3-1 / I3-2 / I3-3>.
No tengo permisos de escritura en AWS: pruebo en local (docker compose, terraform fmt y validate)
y mi cambio llega a AWS por PR. No ejecutes terraform apply.
Antes de escribir código, dime qué archivos vas a crear o cambiar y espera mi OK.
El WAF se crea con el proveedor aws.us_east_1 (CloudFront es global).
Al terminar, dame los commits, la descripción del PR y cómo sacar la evidencia de mi RNF.
```
