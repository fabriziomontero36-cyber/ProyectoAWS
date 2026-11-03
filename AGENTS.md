# AGENTS.md · Contexto del proyecto para asistentes de IA

Lee este archivo completo antes de proponer código. Si lo que te piden contradice algo de aquí, avisa antes de hacerlo. La guía paso a paso está en `docs/GUIA-IMPLEMENTACION.md`.

## 1. El proyecto

- **Curso:** Infraestructura como Código, UPAO 2026-2. Trabajo de 3 integrantes:

  | Integrante | Nombre | RNF | Papel |
  |---|---|---|---|
  | 1 | Fabrizio Montero (GitHub `fabriziomontero36-cyber`) | 1 a 6 | Base: fase 1, cuenta AWS, bootstrap, releases |
  | 2 | Piero Aguirre (GitHub `PieroAguirreDC`) | 7 a 12 | Web, CloudFront, autoescalado, rollback |
  | 3 | Joaquín Portella | 13 a 18 | Base de datos, WAF, límites de API, aforo y reservas |

- **Sistema:** suscripciones, pagos y control de aforo de Champion's Gym (sedes de Trujillo y Chiclayo). Más de 1 500 socios y picos de 200 usuarios concurrentes (lunes a miércoles de 6 a 9 p. m. y campañas de verano).
- **Solución:** monolito modular en Spring Boot (una sola API REST JSON), desplegado en AWS con Terraform y actualizado con GitHub Actions. La web es HTML/CSS/JS estático en S3 detrás de CloudFront.
- **QA:** es local, con `docker-compose.yml` (PostgreSQL + API). En AWS solo existe el entorno `prod`.
- Trabajan en Windows con Git Bash.

## 2. Reglas que no se pueden romper

1. Responder y documentar en **español**. Los commits van en **inglés** con Conventional Commits: `feat(ecs): ...`, `fix(app): ...`.
2. Usar **solo los servicios del diagrama:** Route 53, CloudFront, WAF, ACM, IAM, S3, API Gateway (HTTP API), VPC Link, Cloud Map, ECS Fargate, Application Auto Scaling, ECR, RDS PostgreSQL, SES, CloudWatch, la VPC (subredes, Internet Gateway, NAT Gateway), GitHub Actions y Mercado Pago.
   - Recursos de apoyo permitidos: bucket del estado de Terraform, Parameter Store, grupos de seguridad, tablas de rutas, roles IAM y grupos de logs.
   - **No agregar** ALB/NLB, ElastiCache, réplicas de lectura, SNS, SQS, Lambda, Secrets Manager, EC2, VPC endpoints ni NAT extra sin que el equipo lo apruebe.
3. En los documentos solo hay **2 atributos de calidad: Disponibilidad y Escalabilidad**. **Nunca escribir las siglas en inglés de las metas de recuperación ante desastres** (tiempo y punto de recuperación): el profesor lo penaliza. Los tiempos que ya dicen los RNF (por ejemplo, el 4 y el 13) sí se mantienen.
4. **Versiones fijas, sin `~>`:** Terraform `1.14.8`, `hashicorp/aws` `6.67.0`, `hashicorp/random` `3.9.1`, Java 21, Spring Boot 4.1.x. Los `versions.tf` de los módulos no fijan versión.
5. **Nombres** con prefijo `champions-gym-prod-`. Los roles IAM de la app **tienen que** empezar con `champions-gym-prod-`: el rol de GitHub solo puede crear esos.
6. **Nada de secretos en el repo.** Contraseñas con `random_password` + Parameter Store (`SecureString`), y en ECS con `secrets` y `valueFrom`.
7. **Después de la fase 1, nadie ejecuta `terraform apply` en local.** En local solo `terraform fmt -recursive`, `terraform init -backend=false` y `terraform validate`. El plan se ve en el PR y el apply lo hace `deploy.yml` al llegar a `main`.
8. Una rama por tarea, que sale de `develop` y vuelve a `develop` por PR. Commits pequeños.
9. No cambiar los **contratos de los módulos** (sección 6) sin avisar al dueño del módulo.
10. Spring Boot 4: **no cambiar los starters que generó Initializr** (Flyway necesita `spring-boot-starter-flyway` y `flyway-database-postgresql`).
11. Explicar los cambios en palabras sencillas: el equipo tiene que poder sustentarlos.

## 3. Arquitectura (con las correcciones del 2 de noviembre de 2026)

| Alcance | Servicios |
|---|---|
| Global | IAM, CloudFront + WAF, Route 53, ACM |
| Regional (sa-east-1) | VPC 10.0.0.0/16, Internet Gateway, VPC Link, API Gateway, Cloud Map, ECR, cluster ECS, S3, SES, CloudWatch |
| Zona (sa-east-1a y sa-east-1b) | Subredes, un NAT Gateway por zona, las tareas Fargate, RDS principal (1a) y standby (1b) |
| Externo | Usuarios, GitHub Actions, Mercado Pago |

- **Entrada:** usuario → CloudFront + WAF (fase 2) → `/api/*` → API Gateway HTTP API (ruta `ANY /api/{proxy+}`, stage `$default`) → VPC Link → Cloud Map (registros SRV) → tareas Fargate :8080 → RDS PostgreSQL :5432 (con SSL).
- **Salida a internet:** tarea (subred privada) → NAT Gateway de su misma zona → Internet Gateway. Por ahí se llega a ECR, SES y Mercado Pago. CloudFront **no** es salida: solo recibe tráfico, como el webhook de Mercado Pago.
- **Subredes:** públicas 10.0.1.0/24 (1a) y 10.0.2.0/24 (1b) con NAT · app privadas 10.0.3.0/24 (1a) y 10.0.4.0/24 (1b) · BD privadas 10.0.11.0/24 (1a) y 10.0.12.0/24 (1b), sin salida a internet.
- **Grupos de seguridad:** `sg-vpclink` → `sg-api` por 8080 · `sg-api` → `sg-db` por 5432.
- **Despliegue:** rolling (100 % mínimo, 200 % máximo) con circuit breaker y rollback automático.
- **Números del informe:** de 2 a 10 tareas, escalar con CPU > 70 % durante 5 minutos, disponibilidad del 99,5 %, respaldo diario a las 3:00 a. m. y mantenimiento de 4:00 a 5:00 a. m. (hora de Perú, UTC-5).

## 4. Estado actual (actualizar en cada PR que cierre un paso)

Fase 1 · Integrante 1
- [ ] 1.1 Repositorio, ramas y estructura
- [ ] 1.2 API mínima (`/api/sedes`, `/api/version`) + docker-compose
- [ ] 1.3 Bootstrap: bucket del estado, ECR, OIDC y roles de GitHub
- [ ] 1.4 Infra base: red, RDS, ECS Fargate, Cloud Map, API Gateway
- [ ] 1.5 GitHub Actions: `ci`, `deploy`, `apagar`
- [ ] 1.6 Primer deploy automático, tag `v0.1.0`, accesos de solo lectura del equipo

Fase 2 · los 3
- [ ] I2-1 Web en S3 + CloudFront
- [ ] I2-2 Entrar solo por CloudFront (X-Origin-Verify)
- [ ] I2-3 Autoescalado de 2 a 10
- [ ] I2-4 Rollback y circuit breaker
- [ ] I3-1 Respaldos y failover de la BD
- [ ] I3-2 Límite de peticiones en API Gateway
- [ ] I3-3 WAF en CloudFront
- [ ] I1-1 Evidencias de despliegue sin corte y caída de zona

Fase 3 · los 3
- [ ] Socios y suscripciones (I2)
- [ ] Aforo y reservas (I3)
- [ ] Dominio con Route 53, opcional (I3)
- [ ] Logs de actividad en CloudWatch (I1)
- [ ] Correos con SES (I1)
- [ ] Pagos con Mercado Pago (I1)

## 5. Valores fijos

| Qué | Valor |
|---|---|
| Región | `sa-east-1`; el WAF y el certificado de CloudFront van en `us-east-1` (provider `aws.us_east_1`) |
| Repo | `fabriziomontero36-cyber/champions-gym` · ramas `main` (despliega), `develop`, `feature/*`, `fix/*` |
| Estado de Terraform | bucket `champions-gym-tfstate-<ACCOUNT_ID>`, key `champions-gym/prod/terraform.tfstate`, `use_lockfile = true` (sin DynamoDB) |
| ECR | `champions-gym-api`, tags inmutables, tag = SHA del commit |
| Roles de GitHub | `champions-gym-github-deploy` (solo `main`) y `champions-gym-github-plan` (solo PR, lectura) |
| Variables de GitHub | `AWS_REGION`, `AWS_DEPLOY_ROLE_ARN`, `AWS_PLAN_ROLE_ARN`, `ECR_REPOSITORY` (no hay claves de AWS en GitHub) |
| Workflows | `ci.yml` (PR: tests, docker build, fmt, validate, plan) · `deploy.yml` (push a `main` o manual) · `apagar.yml` (manual, escribir `APAGAR`) · fase 2: `rollback.yml` |
| ECS | cluster `champions-gym-prod-cluster`, servicio `champions-gym-prod-api`, contenedor `api`, 0,5 vCPU / 1 GB, `X86_64` |
| Cloud Map | namespace privado `champions-gym.local`, servicio `api`, registros SRV |
| RDS | `champions-gym-prod-db`, PostgreSQL 17, `db.t4g.micro`, BD `gym`, usuario `gym_admin`, contraseña en `/champions-gym/prod/db/password` |
| Logs | `/ecs/champions-gym-prod-api` (7 días; en la fase 3 pasa a 30) |
| Variables del contenedor | `SPRING_DATASOURCE_URL` (con `?sslmode=require`), `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD` (secreto), `APP_VERSION` (= tag), fase 2: `ORIGIN_VERIFY_SECRET` |

## 6. Contratos de los módulos (`infra/modules/`)

| Módulo | Dueño | Entradas | Salidas |
|---|---|---|---|
| `network` | I1 | `project`, `env`, `azs`, `vpc_cidr`, `public_subnet_cidrs`, `app_subnet_cidrs`, `db_subnet_cidrs`, `nat_per_az`, `app_port` | `vpc_id`, `public_subnet_ids`, `app_subnet_ids` (espera a NAT y rutas), `db_subnet_ids`, `sg_vpclink_id`, `sg_api_id`, `sg_db_id`, `nat_public_ips` |
| `database` | I1 | `project`, `env`, `db_subnet_ids`, `sg_db_id`, `multi_az`; opcionales: `instance_class`, `engine_version`, `db_name`, `db_username` | `db_address`, `db_port`, `db_name`, `db_username`, `db_password_ssm_arn`, `db_instance_id` |
| `ecs_api` | I1 | `project`, `env`, `aws_region`, `vpc_id`, `app_subnet_ids`, `sg_api_id`, `app_port`, `ecr_repository_url`, `image_tag`, `db_address`, `db_port`, `db_name`, `db_username`, `db_password_ssm_arn`, `log_retention_days`; opcionales: `cpu`, `memory`, `desired_count`; fase 2: `origin_verify_secret` | `cluster_name`, `service_name`, `cloudmap_service_arn`, `task_role_name`, `log_group_name` |
| `api_gateway` | I1 | `project`, `env`, `app_subnet_ids`, `sg_vpclink_id`, `cloudmap_service_arn` | `api_endpoint`, `api_id`, `stage_name` |
| `frontend` (fase 2) | I2 | `project`, `env`, `api_endpoint`, `web_acl_arn` (default `null`) | `cloudfront_domain`, `cloudfront_distribution_id`, `frontend_bucket`, `origin_verify_secret` (sensitive) |
| `autoscaling` (fase 2) | I2 | `project`, `env`, `cluster_name`, `service_name` | (ninguna) |
| `waf` (fase 2) | I3 | `project`, `env`; se llama con `providers = { aws = aws.us_east_1 }` | `web_acl_arn` |

- **Raíz (`infra/`):** variables `project`, `env` (solo `"prod"`), `aws_region`, `image_tag` (sin default), los 4 CIDR, `nat_per_az`, `db_multi_az` y `log_retention_days`. Salidas: `api_endpoint`, `ecs_cluster_name`, `ecs_service_name`, `db_address` y `nat_public_ips` (fase 2: `cloudfront_domain`, `frontend_bucket`, `cloudfront_distribution_id`).
- **Bootstrap (`infra/bootstrap/`):** salidas `account_id`, `tfstate_bucket`, `ecr_repository_url`, `github_deploy_role_arn`, `github_plan_role_arn`, `consola_url` y `passwords_iniciales`.
- **Detalles que no se tocan:**
  - El servicio ECS tiene `lifecycle { ignore_changes = [desired_count] }` porque el número de tareas lo maneja el autoescalado.
  - El servicio de Cloud Map lleva `force_destroy = true` y `health_check_custom_config { failure_threshold = 1 }`. El aviso de "deprecated" es esperado.

## 7. Convenciones de la API (Spring Boot)

- Paquete base `pe.championsgym.api`, con un paquete por módulo:
  - Fase 1: `comun` y `sedes`.
  - Fase 3: `socios`, `suscripciones`, `aforo`, `reservas`, `notificaciones` y `pagos`.
- **Solo `pagos` usa arquitectura hexagonal:** `pagos.dominio`, `pagos.puerto` (interfaz `PasarelaPagos`) y `pagos.adaptador.mercadopago`. Los demás módulos usan controlador → servicio → repositorio.
- Todo está bajo `/api` (`server.servlet.context-path=/api`), así que los controladores mapean `/sedes`, `/version`, etc.
- **Endpoints fijos:**
  - `GET /api/version` → `{"version": "..."}` (el pipeline lo usa para saber si ya salió la versión nueva).
  - `GET /api/actuator/health/liveness` → health check de ECS. No revisa la BD.
- **Base de datos:**
  - Las tablas solo las crea Flyway (`ddl-auto: validate`).
  - Las migraciones van en `src/main/resources/db/migration/V<AAAAMMDDHHMM>__<descripcion>.sql`.
  - Nunca editar una migración que ya está en `develop`.
- La configuración llega por variables de entorno; los valores de `application.yml` son los de local.
- **Errores** en JSON: `{"mensaje": "..."}`.
  - BD caída → 503 "Servicio no disponible, intenta en unos minutos".
  - Exceso de peticiones → 429, y la web muestra "intenta en unos segundos".
- **Sin estado en memoria:** cuando haya inicio de sesión, la sesión se guarda en la BD (Spring Session JDBC), así el autoescalado no saca a nadie.
- **Fase 2:** el filtro `X-Origin-Verify` rechaza con 403 lo que no venga de CloudFront, excepto `/api/actuator/**`.

## 8. Comandos

```bash
# QA local
docker compose up --build
curl http://localhost:8080/api/sedes

# Tests (con el compose levantado)
cd app && ./mvnw verify

# Terraform sin credenciales
cd infra && terraform fmt -recursive && terraform init -backend=false && terraform validate

# Logs de las tareas en AWS desde Git Bash (MSYS_NO_PATHCONV evita que cambie la ruta /ecs/...)
MSYS_NO_PATHCONV=1 aws logs tail /ecs/champions-gym-prod-api --follow --region sa-east-1
```

## 9. RNF (resumen), dueño y fase

| Nº | Qué exige (resumen) | Componentes | Dueño | Fase |
|---|---|---|---|---|
| 1 | Permisos automáticos para cada nueva instancia de la API; rechazar y registrar los cambios de quien no es administrador | IAM | I1 | 1 |
| 2 | Publicar una versión nueva en menos de 15 min, sin caída; si falla, no reemplaza a la que funciona | GitHub Actions, ECR, ECS Fargate | I1 | 1 |
| 3 | Cada pago aprobado se registra exactamente una vez, aunque la API haya estado caída | CloudFront, API Gateway, ECS, Mercado Pago | I1 | 3 |
| 4 | API en 2 zonas con al menos 1 instancia en cada una; si cae una zona, la otra atiende el 100 % y se recupera en menos de 5 min | ECS Fargate, subredes privadas 1a y 1b | I1 | 1 |
| 5 | Hasta 1 500 correos por hora, 14 por segundo, sin perder ninguno | SES | I1 | 3 |
| 6 | Registrar el 100 % de la actividad con menos de 1 min de retraso, durante 30 días | CloudWatch | I1 | 3 |
| 7 | Disponibilidad mensual mínima de 99,5 % (web, API y BD), sin contar mantenimientos registrados | S3, CloudFront, ECS, RDS | I2 | 2 |
| 8 | Disponibilidad de 99,5 % para la API de consultas | ECS, API Gateway | I2 | 2 |
| 9 | Volver a la versión estable anterior en menos de 10 min | ECS, ECR | I2 | 2 |
| 10 | Más instancias cuando la CPU supera el 70 % durante 5 min; menos cuando baja | ECS, Application Auto Scaling | I2 | 2 |
| 11 | Entre 2 y 10 instancias, nunca menos de 2 | ECS, Application Auto Scaling | I2 | 2 |
| 12 | 1 000 solicitudes por minuto de contenido estático durante 15 min, con 99 % o más de éxito | S3, CloudFront | I2 | 2 |
| 13 | La BD pasa al nodo secundario en menos de 2 min; mientras tanto, la API responde "Servicio no disponible, intenta en unos minutos" | RDS Multi-AZ | I3 | 2 |
| 14 | Consultas de aforo y reservas en menos de 1,5 s, hasta 500 solicitudes por segundo | ECS, API Gateway, VPC Link | I3 | 3 |
| 15 | Respaldo completo diario a las 3:00 a. m., en menos de 45 min; avisar si falla (el aviso necesitaría SNS, pendiente de decidir) | Backups de RDS, S3 | I3 | 2 |
| 16 | La entrada recibe 300 peticiones por minuto en campañas, en menos de 200 ms; bloquear la IP que haga más de 2 000 solicitudes en 5 min | WAF, CloudFront | I3 | 2 |
| 17 | El dominio siempre lleva a la entrada del sistema | Route 53 | I3 | 3 (opcional) |
| 18 | 200 usuarios activos y 500 conexiones simultáneas con menos de 0,1 % de errores; si se supera, "intenta en unos segundos" | API Gateway, VPC Link | I3 | 2 |
