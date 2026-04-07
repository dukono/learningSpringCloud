# 🐍 Python Backend para Desarrolladores Java/Spring

Guía completa orientada a quien ya sabe Java y Spring Boot.
Cada concepto se explica con su **equivalente Java** al lado.

---

## Índice

1. [Python vs Java — Diferencias clave](#1-python-vs-java--diferencias-clave)
2. [Estructura de un proyecto Python](#2-estructura-de-un-proyecto-python)
3. [Dependencias — pip y pyproject.toml](#3-dependencias--pip-y-pyprojecttoml)
4. [Entornos virtuales — el equivalente a Maven wrapper](#4-entornos-virtuales--el-equivalente-a-maven-wrapper)
5. [El framework web — FastAPI (equivalente a Spring Boot)](#5-el-framework-web--fastapi-equivalente-a-spring-boot)
6. [Controladores y rutas](#6-controladores-y-rutas)
7. [Modelos de datos — Pydantic (equivalente a Records / DTOs)](#7-modelos-de-datos--pydantic-equivalente-a-records--dtos)
8. [Inyección de dependencias](#8-inyección-de-dependencias)
9. [Servicios y capas de negocio](#9-servicios-y-capas-de-negocio)
10. [Acceso a datos — SQLAlchemy (equivalente a JPA/Hibernate)](#10-acceso-a-datos--sqlalchemy-equivalente-a-jpahiberante)
11. [Repositorios y queries](#11-repositorios-y-queries)
12. [Migraciones de BD — Alembic (equivalente a Flyway/Liquibase)](#12-migraciones-de-bd--alembic-equivalente-a-flywayliquibase)
13. [Seguridad y autenticación — JWT](#13-seguridad-y-autenticación--jwt)
14. [Variables de entorno y configuración](#14-variables-de-entorno-y-configuración)
15. [Manejo de errores y excepciones](#15-manejo-de-errores-y-excepciones)
16. [Middleware](#16-middleware)
17. [Testing](#17-testing)
18. [Estructura completa de proyecto real](#18-estructura-completa-de-proyecto-real)
19. [Cheatsheet Java → Python](#19-cheatsheet-java--python)
20. [Seguridad avanzada — RBAC, rate limiting, CORS, SQL injection, secrets](#20-seguridad-avanzada)
21. [pip vs poetry vs uv — dependencias en detalle](#21-pip-vs-poetry-vs-uv--gestión-de-dependencias-en-detalle)
22. [Calidad de código — ruff, mypy y pre-commit](#22-calidad-de-código--ruff-mypy-y-pre-commit)
23. [CI/CD con GitHub Actions para Python](#23-cicd-con-github-actions-para-python)
24. [Patrones de diseño — Repository, Service Layer, Singleton, Factory](#24-patrones-de-diseño-en-python)
25. [Estructura de proyecto — árbol completo y convenciones de nombres](#25-estructura-de-proyecto--árbol-completo-con-descripción-de-cada-archivo)

---

## 1. Python vs Java — Diferencias clave

### Lo que cambia radicalmente

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     JAVA  vs  PYTHON                                     │
├─────────────────────────────┬────────────────────────────────────────────┤
│  JAVA                       │  PYTHON                                    │
├─────────────────────────────┼────────────────────────────────────────────┤
│  Tipado estático             │  Tipado dinámico (pero hay type hints)    │
│  Compilado (.class, .jar)   │  Interpretado (se ejecuta directo)         │
│  Llaves {}  para bloques    │  Indentación para bloques (4 espacios)     │
│  Punto y coma ; obligatorio │  Sin punto y coma                          │
│  Verboso (getters/setters)  │  Conciso (menos código)                    │
│  1 clase por archivo        │  Muchas clases por archivo (normal)        │
│  Compilación antes de run   │  python app.py  y ya funciona              │
│  JVM                        │  Intérprete CPython                        │
│  Maven / Gradle             │  pip / poetry / uv                         │
│  pom.xml / build.gradle     │  pyproject.toml / requirements.txt         │
└─────────────────────────────┴────────────────────────────────────────────┘
```

### Sintaxis básica — comparativa

```java
// JAVA
public class Ejemplo {
    private String nombre;
    private int edad;

    public Ejemplo(String nombre, int edad) {
        this.nombre = nombre;
        this.edad = edad;
    }

    public String saludar() {
        if (this.edad >= 18) {
            return "Hola, " + this.nombre;
        } else {
            return "Hola, menor";
        }
    }
}
```

```python
# PYTHON — equivalente exacto
class Ejemplo:
    def __init__(self, nombre: str, edad: int):  # constructor = __init__
        self.nombre = nombre                      # no hay private/public
        self.edad = edad

    def saludar(self) -> str:                    # self = this
        if self.edad >= 18:                      # sin llaves, con :
            return f"Hola, {self.nombre}"        # f-string = String.format
        else:
            return "Hola, menor"
```

### Type hints — Python SÍ tiene tipos (opcionales)

```python
# Sin type hints (válido pero no recomendado en proyectos serios)
def sumar(a, b):
    return a + b

# Con type hints (recomendado — igual que Java pero opcional)
def sumar(a: int, b: int) -> int:
    return a + b

# Tipos comunes
nombre: str = "Juan"
edad: int = 25
precio: float = 9.99
activo: bool = True
lista: list[str] = ["a", "b", "c"]          # List<String>
mapa: dict[str, int] = {"uno": 1, "dos": 2}  # Map<String, Integer>
opcional: str | None = None                  # Optional<String>
```

### Indentación — la regla más importante

```python
# ✅ CORRECTO — 4 espacios de indentación
def procesar(items: list[str]) -> list[str]:
    resultado = []
    for item in items:
        if len(item) > 0:
            resultado.append(item.upper())
    return resultado

# ❌ ERROR — indentación incorrecta (Python lanza IndentationError)
def procesar(items):
  resultado = []      # 2 espacios — inconsistente
    for item in items:  # 4 espacios — error
        pass
```

---

## 2. Estructura de un proyecto Python

### Comparativa de estructura

```
JAVA / SPRING BOOT                    PYTHON / FASTAPI
──────────────────────────────────    ──────────────────────────────────
my-app/                               my_app/
├── pom.xml                           ├── pyproject.toml
├── src/
│   └── main/
│       ├── java/
│       │   └── com/empresa/app/      ├── app/
│       │       ├── Application.java  │   ├── main.py
│       │       ├── controller/       │   ├── routers/
│       │       ├── service/          │   ├── services/
│       │       ├── repository/       │   ├── repositories/
│       │       ├── model/            │   ├── models/
│       │       └── config/           │   ├── core/
│       └── resources/                │   └── schemas/
│           └── application.yml       ├── .env
└── src/
    └── test/                         └── tests/
```

### Estructura real de un proyecto FastAPI profesional

```
my_app/
│
├── pyproject.toml          ← dependencias (= pom.xml)
├── .env                    ← variables de entorno (NO commitear)
├── .env.example            ← plantilla del .env (SÍ commitear)
├── alembic.ini             ← configuración de migraciones (= Flyway config)
├── README.md
│
├── app/                    ← código fuente principal
│   ├── main.py             ← punto de entrada (= Application.java)
│   ├── dependencies.py     ← inyección de dependencias global
│   │
│   ├── core/               ← configuración central (= config/ en Spring)
│   │   ├── config.py       ← settings (= application.yml cargado en Java)
│   │   ├── security.py     ← JWT, password hashing
│   │   └── database.py     ← conexión a BD (= DataSource config)
│   │
│   ├── routers/            ← controladores (= @RestController)
│   │   ├── users.py
│   │   ├── products.py
│   │   └── auth.py
│   │
│   ├── services/           ← lógica de negocio (= @Service)
│   │   ├── user_service.py
│   │   └── product_service.py
│   │
│   ├── repositories/       ← acceso a datos (= @Repository / JPA)
│   │   ├── user_repository.py
│   │   └── product_repository.py
│   │
│   ├── models/             ← entidades de BD (= @Entity)
│   │   ├── user.py
│   │   └── product.py
│   │
│   └── schemas/            ← DTOs / request-response (= @RequestBody / @ResponseBody)
│       ├── user.py
│       └── product.py
│
├── migrations/             ← scripts de BD (= resources/db/migration en Flyway)
│   └── versions/
│       └── 001_create_users.py
│
└── tests/                  ← tests (= src/test/java)
    ├── conftest.py         ← fixtures globales (= @BeforeEach / TestConfig)
    ├── test_users.py
    └── test_products.py
```

---

## 3. Dependencias — pip y pyproject.toml

### Comparativa con Maven

```
MAVEN (pom.xml)                       PYTHON (pyproject.toml)
──────────────────────────────────    ──────────────────────────────────
<dependency>                          fastapi>=0.110.0
  <groupId>org.springframework.boot
  <artifactId>spring-boot-starter-web
</dependency>

mvn install                           pip install -e .
mvn clean install                     pip install -r requirements.txt
mvn test                              pytest
```

### pyproject.toml — el pom.xml de Python

```toml
# pyproject.toml

[project]
name = "my-app"
version = "0.1.0"
description = "Mi API con FastAPI"
requires-python = ">=3.11"           # como <java.version> en Maven

# Dependencias de producción (= <dependencies> en pom.xml)
dependencies = [
    "fastapi>=0.110.0",              # framework web (= spring-boot-starter-web)
    "uvicorn[standard]>=0.29.0",     # servidor HTTP (= Tomcat embebido)
    "sqlalchemy>=2.0.0",             # ORM (= Hibernate / JPA)
    "alembic>=1.13.0",               # migraciones BD (= Flyway)
    "pydantic>=2.6.0",               # validación de datos (= Bean Validation)
    "pydantic-settings>=2.2.0",      # configuración tipada (= @ConfigurationProperties)
    "python-jose[cryptography]",     # JWT tokens (= spring-security-jwt)
    "passlib[bcrypt]",               # hashing de passwords (= BCryptPasswordEncoder)
    "python-dotenv>=1.0.0",          # leer .env files
    "psycopg2-binary>=2.9.0",        # driver PostgreSQL (= postgresql JDBC driver)
    "httpx>=0.27.0",                 # cliente HTTP (= RestTemplate / WebClient)
]

# Dependencias de desarrollo/test (= <scope>test</scope>)
[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",                 # testing (= JUnit)
    "pytest-asyncio>=0.23.0",        # tests async
    "httpx>=0.27.0",                 # cliente HTTP para tests
    "ruff>=0.3.0",                   # linter + formatter (= Checkstyle + Spotless)
]
```

### Comandos básicos de pip

```bash
# Instalar todas las dependencias del proyecto
pip install -e .
pip install -e ".[dev]"        # incluir dependencias de desarrollo

# Añadir una dependencia (después editar pyproject.toml manualmente)
pip install fastapi             # instala en el entorno activo

# Ver dependencias instaladas
pip list
pip show fastapi                # detalle de un paquete

# Generar requirements.txt (para Docker, CI/CD)
pip freeze > requirements.txt

# Instalar desde requirements.txt
pip install -r requirements.txt
```

### requirements.txt — alternativa simple

Si no quieres usar pyproject.toml puedes usar requirements.txt (más simple, menos potente):

```txt
# requirements.txt
fastapi==0.110.0
uvicorn[standard]==0.29.0
sqlalchemy==2.0.29
alembic==1.13.1
pydantic==2.6.4
pydantic-settings==2.2.1
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
python-dotenv==1.0.1
psycopg2-binary==2.9.9
httpx==0.27.0
```

---

## 4. Entornos virtuales — el equivalente a Maven wrapper

### ¿Por qué existen?

```
Sin entorno virtual (problemático):
┌──────────────────────────────────────────────────────────┐
│  Sistema operativo                                       │
│  └── Python global                                       │
│       ├── proyecto-A necesita  fastapi==0.100.0          │
│       └── proyecto-B necesita  fastapi==0.110.0  ← CONFLICTO
└──────────────────────────────────────────────────────────┘

Con entorno virtual (correcto):
┌──────────────────────────────────────────────────────────┐
│  Sistema operativo                                       │
│  ├── Python global (sin librerías de proyecto)           │
│  ├── proyecto-A/                                         │
│  │   └── .venv/  ← entorno aislado con fastapi==0.100.0  │
│  └── proyecto-B/                                         │
│      └── .venv/  ← entorno aislado con fastapi==0.110.0  │
└──────────────────────────────────────────────────────────┘
```

### Crear y usar un entorno virtual

```bash
# 1. Crear el entorno virtual (una sola vez por proyecto)
python -m venv .venv

# 2. Activarlo (cada vez que abres una terminal nueva)
source .venv/bin/activate          # Linux / Mac
.venv\Scripts\activate             # Windows

# Sabrás que está activo porque el prompt cambia:
# (.venv) usuario@maquina:~/my_app$

# 3. Instalar dependencias (dentro del entorno activo)
pip install -e ".[dev]"

# 4. Desactivar cuando termines
deactivate
```

> **En IntelliJ / PyCharm:** File → Settings → Project → Python Interpreter → Add → Virtualenv → selecciona `.venv`
> Después el IDE activa el entorno automáticamente.

### uv — el gestor moderno (más rápido, recomendado)

```bash
# Instalar uv (una sola vez)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Crear proyecto nuevo
uv init my_app
cd my_app

# Añadir dependencias (edita pyproject.toml automáticamente)
uv add fastapi uvicorn sqlalchemy alembic pydantic pydantic-settings
uv add --dev pytest ruff

# Ejecutar comandos dentro del entorno (sin activarlo manualmente)
uv run python app/main.py
uv run pytest
```

---

## 5. El framework web — FastAPI (equivalente a Spring Boot)

### ¿Por qué FastAPI?

```
┌─────────────────────────────────────────────────────────────┐
│  Frameworks web en Python                                   │
├──────────────┬──────────────────────────────────────────────┤
│  FastAPI     │  ★ Recomendado. Moderno, rápido, tipado,    │
│              │  genera Swagger automático, async nativo.    │
│              │  Equivalente a Spring Boot moderno.          │
├──────────────┼──────────────────────────────────────────────┤
│  Django      │  "Baterías incluidas". ORM propio, admin UI, │
│              │  más opinado. Equivalente a Spring MVC full. │
├──────────────┼──────────────────────────────────────────────┤
│  Flask       │  Minimalista. Sin ORM, sin validación.       │
│              │  Equivalente a JAX-RS básico.                │
└──────────────┴──────────────────────────────────────────────┘
```

### app/main.py — punto de entrada

```python
# app/main.py
# Equivalente a Application.java con @SpringBootApplication

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.core.config import settings
from app.routers import users, products, auth

# Crear la aplicación (= new SpringApplication())
app = FastAPI(
    title=settings.PROJECT_NAME,
    version="1.0.0",
    description="Mi API REST",
    # Swagger UI disponible en /docs
    # OpenAPI JSON en /openapi.json
)

# CORS (= @CrossOrigin en Spring o CorsConfigurationSource)
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Registrar rutas (= @ComponentScan / @Import en Spring)
app.include_router(auth.router,     prefix="/api/v1/auth",     tags=["auth"])
app.include_router(users.router,    prefix="/api/v1/users",    tags=["users"])
app.include_router(products.router, prefix="/api/v1/products", tags=["products"])

# Endpoint de health check
@app.get("/health")
def health():
    return {"status": "ok"}
```

### Arrancar el servidor

```bash
# Desarrollo (con auto-reload = DevTools en Spring Boot)
uvicorn app.main:app --reload --port 8080

# Producción
uvicorn app.main:app --host 0.0.0.0 --port 8080 --workers 4

# Con uv
uv run uvicorn app.main:app --reload
```

```
Servidor arrancado:
  http://localhost:8080        ← tu API
  http://localhost:8080/docs   ← Swagger UI (automático, sin configurar nada)
  http://localhost:8080/redoc  ← ReDoc (alternativa)
```

---

## 6. Controladores y rutas

### Comparativa con Spring MVC

```java
// JAVA — Spring Boot
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping
    public List<UserResponse> getAll() {
        return userService.findAll();
    }

    @GetMapping("/{id}")
    public UserResponse getById(@PathVariable Long id) {
        return userService.findById(id);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserResponse create(@Valid @RequestBody UserCreateRequest request) {
        return userService.create(request);
    }

    @PutMapping("/{id}")
    public UserResponse update(@PathVariable Long id,
                               @Valid @RequestBody UserUpdateRequest request) {
        return userService.update(id, request);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

```python
# PYTHON — FastAPI equivalente exacto
# app/routers/users.py

from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from app.dependencies import get_db, get_current_user
from app.services.user_service import UserService
from app.schemas.user import UserResponse, UserCreateRequest, UserUpdateRequest

# APIRouter = @RestController + @RequestMapping
router = APIRouter()

@router.get("/", response_model=list[UserResponse])
def get_all(
    db: Session = Depends(get_db)           # inyección de dependencias
):
    service = UserService(db)
    return service.find_all()

@router.get("/{user_id}", response_model=UserResponse)
def get_by_id(
    user_id: int,                            # @PathVariable — se infiere del nombre
    db: Session = Depends(get_db)
):
    service = UserService(db)
    user = service.find_by_id(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="Usuario no encontrado")
    return user

@router.post("/", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
def create(
    request: UserCreateRequest,             # @RequestBody — se infiere del tipo Pydantic
    db: Session = Depends(get_db)
):
    service = UserService(db)
    return service.create(request)

@router.put("/{user_id}", response_model=UserResponse)
def update(
    user_id: int,
    request: UserUpdateRequest,
    db: Session = Depends(get_db)
):
    service = UserService(db)
    return service.update(user_id, request)

@router.delete("/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete(
    user_id: int,
    db: Session = Depends(get_db)
):
    service = UserService(db)
    service.delete(user_id)
```

### Query params, path params y body — cómo los distingue FastAPI

```python
# FastAPI infiere el origen del parámetro por su tipo:

@router.get("/search")
def search(
    name: str,               # → Query param  (?name=Juan)     porque es tipo simple
    age: int = 0,            # → Query param  (?age=25)        con valor por defecto
    user_id: int,            # → Path param   (/search/{user_id}) si está en la ruta
    body: UserSchema,        # → Request body (JSON)           porque es modelo Pydantic
    db: Session = Depends(), # → Inyectado    (no viene del request)
):
    pass

# Ejemplo de URL: GET /search?name=Juan&age=25
```

---

## 7. Modelos de datos — Pydantic (equivalente a Records / DTOs)

### Comparativa

```java
// JAVA — DTO con Bean Validation
public record UserCreateRequest(
    @NotBlank String name,
    @Email String email,
    @Size(min = 8) String password,
    @Min(0) Integer age
) {}

public record UserResponse(
    Long id,
    String name,
    String email,
    LocalDateTime createdAt
) {}
```

```python
# PYTHON — Pydantic (validación automática, igual que Bean Validation)
# app/schemas/user.py

from pydantic import BaseModel, EmailStr, Field
from datetime import datetime

# @RequestBody para crear usuario
class UserCreateRequest(BaseModel):
    name: str = Field(min_length=1, max_length=100)    # @NotBlank + @Size
    email: EmailStr                                     # @Email
    password: str = Field(min_length=8)                # @Size(min=8)
    age: int = Field(ge=0, le=150)                     # @Min(0) @Max(150)

# @ResponseBody — lo que devuelve la API
class UserResponse(BaseModel):
    id: int
    name: str
    email: str
    created_at: datetime

    class Config:
        from_attributes = True    # permite crear desde un objeto SQLAlchemy (= @Mapper en MapStruct)

# Para actualizaciones parciales (campos opcionales)
class UserUpdateRequest(BaseModel):
    name: str | None = None        # Optional<String> — si no viene, no se actualiza
    email: EmailStr | None = None
    age: int | None = None
```

### Validación automática

```python
# FastAPI valida automáticamente al recibir el request.
# Si falla, devuelve 422 con detalle de los errores — sin código extra.

# Ejemplo de respuesta de error automática:
# POST /users  con body: {"name": "", "email": "no-es-email", "password": "123"}
# Respuesta 422:
{
    "detail": [
        {"loc": ["body", "name"],     "msg": "String should have at least 1 character"},
        {"loc": ["body", "email"],    "msg": "value is not a valid email address"},
        {"loc": ["body", "password"], "msg": "String should have at least 8 characters"},
        {"loc": ["body", "age"],      "msg": "Field required"}
    ]
}
```

### Pydantic Field — equivalencias con Bean Validation

```python
from pydantic import BaseModel, Field

class Ejemplo(BaseModel):
    # Bean Validation → Pydantic Field
    nombre:   str   = Field(min_length=1)          # @NotBlank
    apellido: str   = Field(min_length=1, max_length=50)  # @Size(min=1,max=50)
    email:    EmailStr                              # @Email
    edad:     int   = Field(ge=0, le=150)          # @Min(0) @Max(150)
    precio:   float = Field(gt=0)                  # @DecimalMin(exclusive=true)
    codigo:   str   = Field(pattern=r"^[A-Z]{3}$") # @Pattern
    lista:    list[str] = Field(min_length=1)       # @NotEmpty
    opcional: str | None = None                    # @Nullable / Optional
```

---

## 8. Inyección de dependencias

### Comparativa con Spring

```
SPRING                                FASTAPI
──────────────────────────────────    ──────────────────────────────────
@Autowired                            Depends()
@Bean                                 función que retorna el recurso
@Component / @Service                 clase normal (sin anotación)
@Scope("request")                     función en Depends (se ejecuta por request)
ApplicationContext                    no existe como tal (más simple)
```

### Cómo funciona Depends()

```python
# app/dependencies.py
# Equivalente a @Configuration en Spring

from sqlalchemy.orm import Session
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

from app.core.database import SessionLocal
from app.core.security import verify_token

# ── Dependencia de base de datos ────────────────────────────────────────────
# Equivalente a @PersistenceContext EntityManager o @Autowired JpaRepository

def get_db():
    """
    Crea una sesión de BD por request y la cierra al terminar.
    = @Transactional en Spring (abre y cierra automáticamente)
    """
    db = SessionLocal()
    try:
        yield db          # "yield" = el código de abajo se ejecuta AL TERMINAR
    finally:
        db.close()        # siempre se cierra, haya error o no

# ── Dependencia de usuario autenticado ──────────────────────────────────────
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login")

def get_current_user(
    token: str = Depends(oauth2_scheme),   # extrae el Bearer token del header
    db: Session = Depends(get_db)
):
    """
    Valida el JWT y devuelve el usuario.
    = @PreAuthorize("isAuthenticated()") en Spring Security
    """
    payload = verify_token(token)
    if payload is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token inválido o expirado"
        )
    user = db.query(User).filter(User.id == payload["sub"]).first()
    if not user:
        raise HTTPException(status_code=404, detail="Usuario no encontrado")
    return user

def get_current_admin(
    current_user = Depends(get_current_user)  # composición de dependencias
):
    """
    = @PreAuthorize("hasRole('ADMIN')") en Spring Security
    """
    if current_user.role != "admin":
        raise HTTPException(status_code=403, detail="No autorizado")
    return current_user
```

### Usar las dependencias en los routers

```python
# app/routers/users.py

@router.get("/me", response_model=UserResponse)
def get_my_profile(
    current_user = Depends(get_current_user)   # protege el endpoint
):
    return current_user

@router.delete("/admin/{user_id}", status_code=204)
def admin_delete_user(
    user_id: int,
    db: Session = Depends(get_db),
    admin = Depends(get_current_admin)          # solo admins
):
    # ...
    pass
```

---

## 9. Servicios y capas de negocio

### Equivalente a @Service en Spring

```java
// JAVA
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
    @Autowired
    private PasswordEncoder passwordEncoder;

    public UserResponse create(UserCreateRequest request) {
        if (userRepository.existsByEmail(request.email())) {
            throw new ConflictException("Email ya registrado");
        }
        User user = new User();
        user.setName(request.name());
        user.setEmail(request.email());
        user.setPasswordHash(passwordEncoder.encode(request.password()));
        User saved = userRepository.save(user);
        return UserMapper.toResponse(saved);
    }
}
```

```python
# PYTHON — equivalente exacto
# app/services/user_service.py

from sqlalchemy.orm import Session
from fastapi import HTTPException, status
from passlib.context import CryptContext

from app.models.user import User
from app.schemas.user import UserCreateRequest, UserResponse

pwd_context = CryptContext(schemes=["bcrypt"])  # = BCryptPasswordEncoder

class UserService:
    def __init__(self, db: Session):             # = @Autowired en el constructor
        self.db = db

    def find_all(self) -> list[User]:
        return self.db.query(User).all()

    def find_by_id(self, user_id: int) -> User | None:
        return self.db.query(User).filter(User.id == user_id).first()

    def create(self, request: UserCreateRequest) -> User:
        # Verificar que el email no existe
        existing = self.db.query(User).filter(User.email == request.email).first()
        if existing:
            raise HTTPException(
                status_code=status.HTTP_409_CONFLICT,
                detail="El email ya está registrado"
            )

        # Crear el usuario (= new User() + setters)
        user = User(
            name=request.name,
            email=request.email,
            password_hash=pwd_context.hash(request.password),  # hash del password
        )

        self.db.add(user)       # = entityManager.persist() / repository.save()
        self.db.commit()        # = @Transactional commit
        self.db.refresh(user)   # recarga el objeto con el ID generado por la BD
        return user

    def update(self, user_id: int, request) -> User:
        user = self.find_by_id(user_id)
        if not user:
            raise HTTPException(status_code=404, detail="Usuario no encontrado")

        # Actualiza solo los campos que vinieron en el request (no None)
        if request.name is not None:
            user.name = request.name
        if request.email is not None:
            user.email = request.email

        self.db.commit()
        self.db.refresh(user)
        return user

    def delete(self, user_id: int) -> None:
        user = self.find_by_id(user_id)
        if not user:
            raise HTTPException(status_code=404, detail="Usuario no encontrado")
        self.db.delete(user)
        self.db.commit()
```

---

## 10. Acceso a datos — SQLAlchemy (equivalente a JPA/Hibernate)

### Configuración de la conexión

```python
# app/core/database.py
# Equivalente a application.yml spring.datasource + JPA config

from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, DeclarativeBase
from app.core.config import settings

# URL de conexión (= spring.datasource.url)
# postgresql://usuario:password@host:5432/nombre_bd
engine = create_engine(
    settings.DATABASE_URL,
    pool_size=10,              # = spring.datasource.hikari.maximum-pool-size
    max_overflow=20,
    echo=False,                # echo=True imprime las SQL (= spring.jpa.show-sql=true)
)

# Fábrica de sesiones (= EntityManagerFactory)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# Clase base para los modelos (= @Entity base)
class Base(DeclarativeBase):
    pass
```

### Modelos / Entidades

```java
// JAVA — @Entity
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(unique = true, nullable = false)
    private String email;

    @Column(name = "password_hash")
    private String passwordHash;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @OneToMany(mappedBy = "user")
    private List<Order> orders;
}
```

```python
# PYTHON — SQLAlchemy (equivalente exacto)
# app/models/user.py

from sqlalchemy import String, Integer, DateTime, ForeignKey, func
from sqlalchemy.orm import Mapped, mapped_column, relationship
from datetime import datetime
from app.core.database import Base

class User(Base):
    __tablename__ = "users"       # = @Table(name = "users")

    # = @Id @GeneratedValue
    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)

    # = @Column(nullable = false)
    name: Mapped[str] = mapped_column(String(100), nullable=False)

    # = @Column(unique = true, nullable = false)
    email: Mapped[str] = mapped_column(String(255), unique=True, nullable=False, index=True)

    # = @Column(name = "password_hash")
    password_hash: Mapped[str] = mapped_column(String(255), nullable=False)

    # = @Column con valor por defecto del servidor
    created_at: Mapped[datetime] = mapped_column(DateTime, server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime, server_default=func.now(), onupdate=func.now())

    # = @OneToMany(mappedBy = "user")
    orders: Mapped[list["Order"]] = relationship("Order", back_populates="user")

    def __repr__(self):    # = toString()
        return f"<User id={self.id} email={self.email}>"


# Otro modelo con relación
class Order(Base):
    __tablename__ = "orders"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    total: Mapped[float] = mapped_column(nullable=False)

    # = @ManyToOne @JoinColumn(name="user_id")
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), nullable=False)
    user: Mapped["User"] = relationship("User", back_populates="orders")
```

---

## 11. Repositorios y queries

### Queries básicas

```python
# app/repositories/user_repository.py
# En Python no hay interfaz mágica como JpaRepository.
# Se escriben las queries directamente — más explícito.

from sqlalchemy.orm import Session
from sqlalchemy import select, and_, or_
from app.models.user import User

class UserRepository:
    def __init__(self, db: Session):
        self.db = db

    # findAll()
    def find_all(self) -> list[User]:
        return self.db.query(User).all()

    # findById()
    def find_by_id(self, user_id: int) -> User | None:
        return self.db.query(User).filter(User.id == user_id).first()

    # findByEmail()
    def find_by_email(self, email: str) -> User | None:
        return self.db.query(User).filter(User.email == email).first()

    # existsByEmail()
    def exists_by_email(self, email: str) -> bool:
        return self.db.query(User).filter(User.email == email).first() is not None

    # findAll con paginación (= Pageable en Spring)
    def find_all_paginated(self, page: int = 0, size: int = 10) -> list[User]:
        return (
            self.db.query(User)
            .offset(page * size)   # = PageRequest.of(page, size)
            .limit(size)
            .all()
        )

    # Query con múltiples condiciones
    def search(self, name: str | None, min_age: int | None) -> list[User]:
        query = self.db.query(User)
        if name:
            query = query.filter(User.name.ilike(f"%{name}%"))  # LIKE insensible a mayúsculas
        if min_age:
            query = query.filter(User.age >= min_age)
        return query.order_by(User.name).all()

    # SQL nativo (= @Query(nativeQuery=true) en Spring)
    def find_active_users_raw(self) -> list:
        result = self.db.execute(
            "SELECT id, name, email FROM users WHERE active = true"
        )
        return result.fetchall()
```

### ORM estilo moderno (recomendado)

```python
# Forma moderna con select() — más expresiva
from sqlalchemy import select

def find_users_with_orders(self, db: Session) -> list[User]:
    stmt = (
        select(User)
        .join(User.orders)              # JOIN
        .where(User.active == True)     # WHERE
        .order_by(User.name)            # ORDER BY
        .limit(10)                      # LIMIT
    )
    return db.scalars(stmt).all()
```

---

## 12. Migraciones de BD — Alembic (equivalente a Flyway/Liquibase)

### Configuración inicial

```bash
# Inicializar Alembic en el proyecto (una sola vez)
alembic init migrations

# Estructura generada:
# alembic.ini
# migrations/
#   env.py          ← configuración (modificar para apuntar a tus modelos)
#   versions/       ← aquí van los scripts de migración
```

### Configurar migrations/env.py

```python
# migrations/env.py — conectar Alembic con tus modelos

from app.core.database import Base   # tu Base con todos los modelos
from app.models import user, product  # importar todos los modelos

target_metadata = Base.metadata      # Alembic lee los modelos para generar SQL
```

### Crear y ejecutar migraciones

```bash
# Generar migración automática al detectar cambios en modelos
# (= Flyway genera el SQL tú, aquí lo genera automáticamente)
alembic revision --autogenerate -m "crear tabla users"

# Ver qué SQL va a ejecutar (sin ejecutarlo)
alembic upgrade head --sql

# Aplicar todas las migraciones pendientes (= flyway migrate)
alembic upgrade head

# Ver estado actual (= flyway info)
alembic current
alembic history

# Revertir la última migración (= flyway undo)
alembic downgrade -1

# Revertir todo
alembic downgrade base
```

### Migración generada automáticamente

```python
# migrations/versions/001_crear_tabla_users.py
# (generado por alembic revision --autogenerate)

def upgrade() -> None:
    op.create_table('users',
        sa.Column('id',            sa.Integer(),     nullable=False),
        sa.Column('name',          sa.String(100),   nullable=False),
        sa.Column('email',         sa.String(255),   nullable=False),
        sa.Column('password_hash', sa.String(255),   nullable=False),
        sa.Column('created_at',    sa.DateTime(),    server_default=sa.text('now()')),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('email')
    )

def downgrade() -> None:
    op.drop_table('users')
```

---

## 13. Seguridad y autenticación — JWT

### Comparativa con Spring Security

```
SPRING SECURITY                       FASTAPI + python-jose
──────────────────────────────────    ──────────────────────────────────
SecurityFilterChain                   Middleware / Depends()
UsernamePasswordAuthenticationFilter  endpoint POST /auth/login
JwtAuthenticationFilter               Depends(oauth2_scheme) + verify_token()
@PreAuthorize("isAuthenticated()")    Depends(get_current_user)
@PreAuthorize("hasRole('ADMIN')")     Depends(get_current_admin)
BCryptPasswordEncoder                 passlib CryptContext(["bcrypt"])
UserDetailsService                    función que busca usuario en BD
```

### app/core/security.py

```python
# app/core/security.py

from datetime import datetime, timedelta, timezone
from jose import JWTError, jwt
from passlib.context import CryptContext
from app.core.config import settings

# Hashing de passwords (= BCryptPasswordEncoder)
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)   # = passwordEncoder.matches()

# JWT
def create_access_token(data: dict) -> str:
    """
    Crea un JWT. Equivalente a Jwts.builder() en jjwt (Java).
    """
    payload = data.copy()
    expire = datetime.now(timezone.utc) + timedelta(minutes=settings.JWT_EXPIRE_MINUTES)
    payload["exp"] = expire
    return jwt.encode(payload, settings.JWT_SECRET, algorithm=settings.JWT_ALGORITHM)

def verify_token(token: str) -> dict | None:
    """
    Verifica y decodifica el JWT. Devuelve el payload o None si es inválido.
    """
    try:
        payload = jwt.decode(token, settings.JWT_SECRET, algorithms=[settings.JWT_ALGORITHM])
        return payload
    except JWTError:
        return None
```

### app/routers/auth.py — login y registro

```python
# app/routers/auth.py

from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy.orm import Session

from app.dependencies import get_db
from app.core.security import verify_password, create_access_token, hash_password
from app.models.user import User
from app.schemas.auth import TokenResponse, RegisterRequest

router = APIRouter()

@router.post("/login", response_model=TokenResponse)
def login(
    form: OAuth2PasswordRequestForm = Depends(),  # lee username + password del form
    db: Session = Depends(get_db)
):
    # Buscar usuario
    user = db.query(User).filter(User.email == form.username).first()
    if not user or not verify_password(form.password, user.password_hash):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Credenciales incorrectas"
        )

    # Crear token JWT
    token = create_access_token({"sub": str(user.id), "email": user.email})
    return {"access_token": token, "token_type": "bearer"}

@router.post("/register", status_code=status.HTTP_201_CREATED)
def register(
    request: RegisterRequest,
    db: Session = Depends(get_db)
):
    if db.query(User).filter(User.email == request.email).first():
        raise HTTPException(status_code=409, detail="Email ya registrado")

    user = User(
        name=request.name,
        email=request.email,
        password_hash=hash_password(request.password)
    )
    db.add(user)
    db.commit()
    return {"message": "Usuario creado correctamente"}
```

### Schemas de autenticación

```python
# app/schemas/auth.py

from pydantic import BaseModel, EmailStr

class RegisterRequest(BaseModel):
    name: str
    email: EmailStr
    password: str

class TokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"
```

---

## 14. Variables de entorno y configuración

### Comparativa con Spring

```
SPRING                                PYTHON
──────────────────────────────────    ──────────────────────────────────
application.yml / .properties         .env  (cargado con pydantic-settings)
@ConfigurationProperties              clase Settings(BaseSettings)
@Value("${prop.name}")                settings.PROP_NAME
spring.profiles.active                APP_ENV=production en el .env
```

### .env — nunca lo commitees

```bash
# .env  (en .gitignore)
DATABASE_URL=postgresql://user:pass@localhost:5432/mydb
JWT_SECRET=mi-secreto-super-largo-y-random-aqui
JWT_ALGORITHM=HS256
JWT_EXPIRE_MINUTES=30
PROJECT_NAME=MyApp
APP_ENV=development
ALLOWED_ORIGINS=["http://localhost:3000","http://localhost:8080"]
```

### .env.example — sí lo commitees

```bash
# .env.example  (plantilla sin valores reales)
DATABASE_URL=postgresql://user:password@localhost:5432/dbname
JWT_SECRET=change-this-to-a-random-secret
JWT_ALGORITHM=HS256
JWT_EXPIRE_MINUTES=30
PROJECT_NAME=MyApp
APP_ENV=development
ALLOWED_ORIGINS=["http://localhost:3000"]
```

### app/core/config.py — @ConfigurationProperties en Python

```python
# app/core/config.py
# Equivalente a @ConfigurationProperties + application.yml

from pydantic_settings import BaseSettings
from pydantic import AnyUrl

class Settings(BaseSettings):
    # Lee automáticamente desde .env o variables de entorno del sistema
    DATABASE_URL: str
    JWT_SECRET: str
    JWT_ALGORITHM: str = "HS256"        # valor por defecto
    JWT_EXPIRE_MINUTES: int = 30
    PROJECT_NAME: str = "MyApp"
    APP_ENV: str = "development"
    ALLOWED_ORIGINS: list[str] = ["*"]

    class Config:
        env_file = ".env"               # leer desde el archivo .env
        case_sensitive = True           # DATABASE_URL ≠ database_url

# Instancia global (= @Bean singleton en Spring)
settings = Settings()

# Uso en cualquier archivo:
# from app.core.config import settings
# print(settings.DATABASE_URL)
```

---

## 15. Manejo de errores y excepciones

### Comparativa con Spring

```
SPRING                                FASTAPI
──────────────────────────────────    ──────────────────────────────────
@ControllerAdvice                     @app.exception_handler()
@ExceptionHandler                     HTTPException / custom exception
ResponseEntityExceptionHandler        Automático para 422, 404, etc.
throw new NotFoundException(...)      raise HTTPException(status_code=404)
```

### Errores HTTP con HTTPException

```python
# Directamente en el código (más simple)
from fastapi import HTTPException, status

raise HTTPException(status_code=404, detail="Usuario no encontrado")
raise HTTPException(status_code=409, detail="Email ya registrado")
raise HTTPException(
    status_code=status.HTTP_403_FORBIDDEN,
    detail="No tienes permiso para esta acción"
)
```

### Excepciones personalizadas + handler global (@ControllerAdvice)

```python
# app/core/exceptions.py

# Excepción personalizada
class AppException(Exception):
    def __init__(self, status_code: int, detail: str):
        self.status_code = status_code
        self.detail = detail

class NotFoundException(AppException):
    def __init__(self, resource: str, id: int):
        super().__init__(404, f"{resource} con id {id} no encontrado")

class ConflictException(AppException):
    def __init__(self, detail: str):
        super().__init__(409, detail)
```

```python
# app/main.py — registrar el handler global (= @ControllerAdvice)

from fastapi import Request
from fastapi.responses import JSONResponse
from app.core.exceptions import AppException

@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"detail": exc.detail, "path": str(request.url)}
    )

# Ahora en los servicios puedes lanzar:
# raise NotFoundException("Usuario", user_id)
# raise ConflictException("El email ya existe")
```

---

## 16. Middleware

### Equivalente a Spring Filters / Interceptors

```python
# app/main.py

import time
import logging
from fastapi import Request

logger = logging.getLogger(__name__)

# Middleware de logging (= HandlerInterceptor en Spring MVC)
@app.middleware("http")
async def log_requests(request: Request, call_next):
    start = time.time()
    response = await call_next(request)    # = chain.doFilter() / proceed()
    duration = time.time() - start
    logger.info(f"{request.method} {request.url.path} → {response.status_code} ({duration:.3f}s)")
    return response

# Middleware para añadir headers a todas las respuestas
@app.middleware("http")
async def add_headers(request: Request, call_next):
    response = await call_next(request)
    response.headers["X-API-Version"] = "1.0"
    return response
```

---

## 17. Testing

### Comparativa con JUnit + MockMvc

```
SPRING (JUnit 5 + MockMvc)            PYTHON (pytest + httpx)
──────────────────────────────────    ──────────────────────────────────
@SpringBootTest                       TestClient / AsyncClient
MockMvc                               client.get("/users")
@MockBean                             monkeypatch / MagicMock
@BeforeEach                           fixture con yield
@Transactional (rollback)             BD en memoria / fixture que limpia
assert que ...isEqualTo(...)          assert response.json() == {...}
```

### tests/conftest.py — configuración global de tests

```python
# tests/conftest.py
# Equivalente a @BeforeEach, TestConfig, @SpringBootTest

import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from app.main import app
from app.core.database import Base
from app.dependencies import get_db

# BD en memoria para tests (= H2 en Spring)
SQLALCHEMY_TEST_URL = "sqlite:///./test.db"
engine = create_engine(SQLALCHEMY_TEST_URL, connect_args={"check_same_thread": False})
TestingSessionLocal = sessionmaker(bind=engine)

@pytest.fixture(scope="session", autouse=True)
def setup_db():
    Base.metadata.create_all(bind=engine)   # crear tablas
    yield
    Base.metadata.drop_all(bind=engine)     # limpiar después de todos los tests

@pytest.fixture
def db():
    session = TestingSessionLocal()
    try:
        yield session
    finally:
        session.rollback()    # ← revertir cambios entre tests (= @Transactional)
        session.close()

@pytest.fixture
def client(db):
    # Reemplazar la dependencia de BD por la de test
    def override_get_db():
        yield db

    app.dependency_overrides[get_db] = override_get_db
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()
```

### tests/test_users.py — tests de integración

```python
# tests/test_users.py

def test_create_user(client):
    response = client.post("/api/v1/users/", json={
        "name": "Juan",
        "email": "juan@test.com",
        "password": "password123",
        "age": 25
    })
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Juan"
    assert data["email"] == "juan@test.com"
    assert "id" in data
    assert "password" not in data   # el password nunca debe salir en la respuesta

def test_create_user_duplicate_email(client):
    # Crear usuario
    client.post("/api/v1/users/", json={
        "name": "Juan", "email": "dup@test.com", "password": "pass12345", "age": 20
    })
    # Intentar crear con mismo email
    response = client.post("/api/v1/users/", json={
        "name": "Pedro", "email": "dup@test.com", "password": "pass12345", "age": 22
    })
    assert response.status_code == 409

def test_get_user_not_found(client):
    response = client.get("/api/v1/users/99999")
    assert response.status_code == 404

def test_login(client):
    # Registrar usuario
    client.post("/api/v1/auth/register", json={
        "name": "Test", "email": "test@test.com", "password": "password123"
    })
    # Login
    response = client.post("/api/v1/auth/login", data={
        "username": "test@test.com",
        "password": "password123"
    })
    assert response.status_code == 200
    assert "access_token" in response.json()
```

### Ejecutar los tests

```bash
pytest                          # ejecutar todos
pytest tests/test_users.py      # solo un archivo
pytest -v                       # verbose (= más detalle)
pytest -k "test_login"          # solo tests cuyo nombre contiene "test_login"
pytest --cov=app                # con cobertura de código (= JaCoCo)
```

---

## 18. Estructura completa de proyecto real

### Todos los archivos

```python
# ─────────────────────────────────────────────────────────
# app/core/config.py
# ─────────────────────────────────────────────────────────
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    DATABASE_URL: str
    JWT_SECRET: str
    JWT_ALGORITHM: str = "HS256"
    JWT_EXPIRE_MINUTES: int = 30
    PROJECT_NAME: str = "MyApp"
    ALLOWED_ORIGINS: list[str] = ["*"]

    class Config:
        env_file = ".env"

settings = Settings()
```

```python
# ─────────────────────────────────────────────────────────
# app/core/database.py
# ─────────────────────────────────────────────────────────
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, DeclarativeBase
from app.core.config import settings

engine = create_engine(settings.DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

class Base(DeclarativeBase):
    pass
```

```python
# ─────────────────────────────────────────────────────────
# app/models/user.py
# ─────────────────────────────────────────────────────────
from sqlalchemy import String, Integer, DateTime, func
from sqlalchemy.orm import Mapped, mapped_column
from datetime import datetime
from app.core.database import Base

class User(Base):
    __tablename__ = "users"
    id:            Mapped[int]      = mapped_column(primary_key=True, autoincrement=True)
    name:          Mapped[str]      = mapped_column(String(100), nullable=False)
    email:         Mapped[str]      = mapped_column(String(255), unique=True, nullable=False)
    password_hash: Mapped[str]      = mapped_column(String(255), nullable=False)
    role:          Mapped[str]      = mapped_column(String(20), default="user")
    created_at:    Mapped[datetime] = mapped_column(DateTime, server_default=func.now())
```

```python
# ─────────────────────────────────────────────────────────
# app/schemas/user.py
# ─────────────────────────────────────────────────────────
from pydantic import BaseModel, EmailStr, Field
from datetime import datetime

class UserCreateRequest(BaseModel):
    name:     str      = Field(min_length=1, max_length=100)
    email:    EmailStr
    password: str      = Field(min_length=8)

class UserResponse(BaseModel):
    id:         int
    name:       str
    email:      str
    role:       str
    created_at: datetime

    class Config:
        from_attributes = True
```

```python
# ─────────────────────────────────────────────────────────
# app/dependencies.py
# ─────────────────────────────────────────────────────────
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.orm import Session
from app.core.database import SessionLocal
from app.core.security import verify_token
from app.models.user import User

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login")

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

def get_current_user(token: str = Depends(oauth2_scheme), db: Session = Depends(get_db)):
    payload = verify_token(token)
    if not payload:
        raise HTTPException(status_code=401, detail="Token inválido")
    user = db.query(User).filter(User.id == int(payload["sub"])).first()
    if not user:
        raise HTTPException(status_code=404, detail="Usuario no encontrado")
    return user
```

```python
# ─────────────────────────────────────────────────────────
# app/main.py
# ─────────────────────────────────────────────────────────
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.core.config import settings
from app.routers import users, auth

app = FastAPI(title=settings.PROJECT_NAME)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(auth.router,  prefix="/api/v1/auth",  tags=["auth"])
app.include_router(users.router, prefix="/api/v1/users", tags=["users"])

@app.get("/health")
def health():
    return {"status": "ok"}
```

### Iniciar el proyecto desde cero (comandos en orden)

```bash
# 1. Crear el proyecto
mkdir my_app && cd my_app
python -m venv .venv
source .venv/bin/activate

# 2. Instalar dependencias
pip install fastapi uvicorn sqlalchemy alembic pydantic pydantic-settings \
            python-jose passlib python-dotenv psycopg2-binary httpx \
            pytest pytest-asyncio ruff

# 3. Crear .env
cat > .env << EOF
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/mydb
JWT_SECRET=mi-secreto-muy-largo-y-random
JWT_EXPIRE_MINUTES=30
EOF

# 4. Inicializar Alembic
alembic init migrations

# 5. Arrancar la BD (con Docker)
docker run -d --name postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 postgres:16

# 6. Crear primera migración y aplicarla
alembic revision --autogenerate -m "initial"
alembic upgrade head

# 7. Arrancar la API
uvicorn app.main:app --reload --port 8080

# 8. Abrir Swagger
# http://localhost:8080/docs
```

---

## 19. Cheatsheet Java → Python

### Sintaxis

```
JAVA                              PYTHON
──────────────────────────────    ──────────────────────────────
String s = "hola";                s: str = "hola"
int n = 42;                       n: int = 42
boolean b = true;                 b: bool = True
null                              None
System.out.println("hola");       print("hola")
String.format("Hi %s", name)      f"Hi {name}"
// comentario                     # comentario
/* bloque */                      no existe (usa # en cada línea)
```

### Clases y métodos

```
JAVA                              PYTHON
──────────────────────────────    ──────────────────────────────
public class Foo { }              class Foo:
this.campo                        self.campo
public Foo() { }                  def __init__(self):
@Override toString()              def __repr__(self):
instanceof                        isinstance(obj, Clase)
abstract class                    from abc import ABC, abstractmethod
interface                         Protocol (typing) o ABC
```

### Colecciones

```
JAVA                              PYTHON
──────────────────────────────    ──────────────────────────────
List<String> list = new ArrayList<>()    lista: list[str] = []
list.add("a")                            lista.append("a")
list.get(0)                              lista[0]
list.size()                              len(lista)
Map<String, Integer> map = new HashMap<>()   mapa: dict[str, int] = {}
map.put("k", 1)                          mapa["k"] = 1
map.get("k")                             mapa["k"]  o  mapa.get("k")
map.containsKey("k")                     "k" in mapa
Set<String> set = new HashSet<>()        conjunto: set[str] = set()
```

### Streams → List comprehensions

```
JAVA (Streams)                    PYTHON (comprehensions)
──────────────────────────────    ──────────────────────────────
list.stream()
  .filter(x -> x > 0)             [x for x in lista if x > 0]
  .collect(toList())

list.stream()
  .map(x -> x * 2)                [x * 2 for x in lista]
  .collect(toList())

list.stream()
  .filter(x -> x > 0)
  .map(x -> x * 2)                [x * 2 for x in lista if x > 0]
  .collect(toList())

list.stream()
  .reduce(0, Integer::sum)        sum(lista)

list.stream().count()             len(lista)
```

### Manejo de errores

```
JAVA                              PYTHON
──────────────────────────────    ──────────────────────────────
try { }                           try:
catch (Exception e) { }           except Exception as e:
catch (IOException e) { }         except IOException as e:
finally { }                       finally:
throw new RuntimeException("msg") raise RuntimeException("msg")
throws IOException                (no existe — no hay checked exceptions)
```

### Spring → FastAPI

```
SPRING BOOT                       FASTAPI
──────────────────────────────    ──────────────────────────────
@SpringBootApplication            FastAPI()
@RestController                   APIRouter()
@GetMapping("/path")              @router.get("/path")
@PostMapping("/path")             @router.post("/path")
@PathVariable                     parámetro en ruta def f(id: int)
@RequestBody                      parámetro tipo Pydantic
@RequestParam                     parámetro tipo simple
@ResponseStatus(CREATED)          status_code=201
@Valid                             automático con Pydantic
@Autowired                        Depends()
@Service                          clase normal
@Repository                       clase normal
@Entity                           clase que hereda de Base (SQLAlchemy)
@Column                           mapped_column()
@Id @GeneratedValue               mapped_column(primary_key=True, autoincrement=True)
@OneToMany                        relationship("Modelo", back_populates=...)
@ControllerAdvice                 @app.exception_handler()
application.yml                   .env + Settings(BaseSettings)
```

---

## 20. Seguridad avanzada

### 20.1 RBAC — Control de acceso por roles

```
SPRING SECURITY                       FASTAPI
──────────────────────────────────    ──────────────────────────────────
@PreAuthorize("hasRole('ADMIN')")     Depends(require_role("admin"))
@PreAuthorize("isAuthenticated()")    Depends(get_current_user)
hasAnyRole("ADMIN","MOD")             require_role("admin", "moderator")
SecurityContextHolder                 current_user en Depends()
```

```python
# app/core/roles.py

from enum import Enum
from fastapi import HTTPException, status, Depends
from app.dependencies import get_current_user

class Role(str, Enum):
    USER      = "user"
    MODERATOR = "moderator"
    ADMIN     = "admin"

# Matriz de permisos:
# ┌───────────────────┬──────┬───────────┬───────┐
# │  Acción           │ user │ moderator │ admin │
# ├───────────────────┼──────┼───────────┼───────┤
# │  Ver su perfil    │  ✅  │    ✅     │  ✅   │
# │  Editar su perfil │  ✅  │    ✅     │  ✅   │
# │  Ver otros users  │  ❌  │    ✅     │  ✅   │
# │  Banear users     │  ❌  │    ✅     │  ✅   │
# │  Eliminar users   │  ❌  │    ❌     │  ✅   │
# │  Ver métricas     │  ❌  │    ❌     │  ✅   │
# └───────────────────┴──────┴───────────┴───────┘

# Jerarquía de roles (de menor a mayor)
ROLE_HIERARCHY = {
    Role.USER:      0,
    Role.MODERATOR: 1,
    Role.ADMIN:     2,
}

def require_role(*roles: Role):
    """
    Dependencia que exige que el usuario tenga al menos uno de los roles indicados.
    Equivale a @PreAuthorize("hasAnyRole(...)") en Spring Security.
    """
    def checker(current_user=Depends(get_current_user)):
        if current_user.role not in [r.value for r in roles]:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Se requiere uno de los roles: {[r.value for r in roles]}"
            )
        return current_user
    return checker

def require_min_role(min_role: Role):
    """
    Exige que el usuario tenga un rol igual o superior al mínimo indicado.
    require_min_role(Role.MODERATOR) → acepta moderator y admin.
    """
    def checker(current_user=Depends(get_current_user)):
        user_level = ROLE_HIERARCHY.get(Role(current_user.role), -1)
        min_level  = ROLE_HIERARCHY[min_role]
        if user_level < min_level:
            raise HTTPException(status_code=403, detail="Permiso insuficiente")
        return current_user
    return checker
```

```python
# Uso en los routers

@router.get("/users",       dependencies=[Depends(require_min_role(Role.MODERATOR))])
def list_all_users(): ...

@router.delete("/users/{id}", dependencies=[Depends(require_role(Role.ADMIN))])
def delete_user(id: int): ...

# Ownership: solo el propio usuario puede editar su perfil (o un admin)
@router.put("/users/{id}/profile")
def update_profile(
    id: int,
    current_user=Depends(get_current_user)
):
    if current_user.id != id and current_user.role != Role.ADMIN:
        raise HTTPException(status_code=403, detail="Solo puedes editar tu propio perfil")
    ...
```

---

### 20.2 Rate limiting — protección contra abuso

Equivalente a **Bucket4j** en Java.

```bash
pip install slowapi
```

```python
# app/main.py

from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

# Limiter identifica al usuario por su IP (o puedes usar get_current_user)
limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
```

```python
# app/routers/auth.py — limitar el endpoint de login para evitar fuerza bruta

from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@router.post("/login")
@limiter.limit("5/minute")           # máximo 5 intentos de login por minuto por IP
def login(request: Request, form: OAuth2PasswordRequestForm = Depends(), ...):
    ...

@router.post("/register")
@limiter.limit("3/hour")             # máximo 3 registros por hora por IP
def register(request: Request, ...):
    ...
```

```
Respuesta cuando se supera el límite (HTTP 429 Too Many Requests):

HTTP/1.1 429 Too Many Requests
Retry-After: 47
X-RateLimit-Limit: 5
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1711234614

{
  "error": "Rate limit exceeded: 5 per 1 minute"
}
```

---

### 20.3 CORS en detalle

**¿Qué es CORS y por qué existe?**

```
Sin CORS (mismo origen — OK):
┌─────────────────┐         ┌─────────────────┐
│  frontend       │ fetch() │  backend        │
│  localhost:3000 │────────▶│  localhost:3000 │  ✅ mismo origen, sin restricción
└─────────────────┘         └─────────────────┘

Con CORS (origen diferente — el navegador bloquea):
┌─────────────────┐         ┌─────────────────┐
│  frontend       │ fetch() │  backend        │
│  localhost:3000 │────────▶│  localhost:8080 │  ❌ orígenes distintos
└─────────────────┘    ✋   └─────────────────┘
                    BLOQUEADO
                    por el navegador

Con CORS configurado correctamente:
┌─────────────────┐  OPTIONS  ┌─────────────────┐
│  frontend       │──────────▶│  backend        │
│  localhost:3000 │◀──────────│  localhost:8080 │  1️⃣ preflight: "¿me permites?"
│                 │  200 + headers CORS          │
│                 │  fetch()  │                 │
│                 │──────────▶│                 │  2️⃣ petición real
│                 │◀──────────│                 │  ✅ permitido
└─────────────────┘           └─────────────────┘
```

```python
# app/main.py — configuración correcta de CORS

from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,

    # ✅ BIEN — lista exacta de orígenes permitidos
    allow_origins=["https://mi-app.com", "https://admin.mi-app.com"],

    # ❌ MAL en producción — permite CUALQUIER origen
    # allow_origins=["*"],

    # ⚠️ IMPORTANTE: si allow_credentials=True, NO puedes usar "*" en allow_origins
    # Las cookies y el header Authorization solo se envían con orígenes exactos
    allow_credentials=True,

    allow_methods=["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"],
    allow_headers=["Authorization", "Content-Type", "X-Requested-With"],

    # Cuánto tiempo puede el navegador cachear la respuesta del preflight (en segundos)
    max_age=600,
)
```

```
Errores comunes de CORS y cómo diagnosticarlos:

ERROR en el navegador:
  "Access to fetch at 'http://localhost:8080/api' from origin
   'http://localhost:3000' has been blocked by CORS policy"

CAUSA 1: allow_origins no incluye tu origen
  Solución: añadir "http://localhost:3000" a allow_origins

CAUSA 2: allow_credentials=True con allow_origins=["*"]
  Solución: especificar el origen exacto, nunca "*" con credentials

CAUSA 3: el método HTTP no está en allow_methods
  Solución: añadir "PUT" o "DELETE" a allow_methods

CAUSA 4: el header personalizado no está en allow_headers
  Solución: añadir "X-Custom-Header" a allow_headers

CÓMO DEPURAR: en las DevTools del navegador → Network → buscar la petición
  OPTIONS (preflight) → inspeccionar los headers de respuesta:
    Access-Control-Allow-Origin: http://localhost:3000  ← debe estar
    Access-Control-Allow-Methods: GET, POST, ...
    Access-Control-Allow-Headers: Authorization, ...
```

---

### 20.4 Protección contra inyección SQL

```python
# ❌ VULNERABLE — nunca hagas esto
def get_user_vulnerable(username: str, db: Session):
    # Si username = "admin' OR '1'='1" → devuelve todos los usuarios
    result = db.execute(f"SELECT * FROM users WHERE name = '{username}'")
    return result.fetchall()

# ✅ SEGURO — SQLAlchemy ORM (parametriza automáticamente)
def get_user_orm(username: str, db: Session):
    return db.query(User).filter(User.name == username).first()
    # Genera internamente: SELECT * FROM users WHERE name = ?  (parámetro vinculado)

# ✅ SEGURO — SQL nativo con parámetros explícitos
from sqlalchemy import text

def get_user_native(username: str, db: Session):
    result = db.execute(
        text("SELECT * FROM users WHERE name = :name"),
        {"name": username}   # parámetro vinculado — nunca interpolado en el string
    )
    return result.fetchall()

# ❌ MAL — SQL nativo SIN parámetros (vulnerable)
def get_user_unsafe(username: str, db: Session):
    result = db.execute(text(f"SELECT * FROM users WHERE name = '{username}'"))
    return result.fetchall()
```

```
Regla de oro:
  ORM SQLAlchemy   → seguro por defecto ✅
  text() con :param → seguro ✅
  f-string en SQL  → NUNCA ❌

Primera línea de defensa: Pydantic valida la entrada ANTES de que llegue a la BD.
  Si el campo "username" está definido como  str  con  max_length=50,
  un payload de 1000 caracteres con SQL inyectado falla en la validación del schema.
```

---

### 20.5 Gestión de secretos — qué nunca debe ir en el repo

```
Jerarquía de secretos según el entorno:

┌─────────────────────────────────────────────────────────────────────┐
│  LOCAL (desarrollo)                                                 │
│  └── .env  (en .gitignore — NUNCA commitear)                        │
│       DATABASE_URL=postgresql://user:pass@localhost:5432/dev        │
│       JWT_SECRET=secreto-local-no-importa-si-es-simple             │
├─────────────────────────────────────────────────────────────────────┤
│  CI/CD (GitHub Actions, GitLab CI...)                               │
│  └── Variables de entorno del runner (GitHub Secrets)               │
│       Settings → Secrets → DATABASE_URL, JWT_SECRET                │
│       Se inyectan como env vars durante el pipeline                 │
├─────────────────────────────────────────────────────────────────────┤
│  PRODUCCIÓN                                                         │
│  └── Secrets Manager (AWS / GCP / Azure / Vault)                   │
│       La app los lee al arrancar mediante SDK                       │
│       Rotación automática sin redespliegue                          │
└─────────────────────────────────────────────────────────────────────┘
```

```
Lo que NUNCA debe estar en el repositorio:
  ❌  passwords de base de datos
  ❌  JWT_SECRET / API keys
  ❌  tokens de servicios externos (AWS, Stripe, SendGrid...)
  ❌  certificados privados (.pem, .key)
  ❌  archivos .env con valores reales

Lo que SÍ debe estar:
  ✅  .env.example  (plantilla con keys pero sin valores)
  ✅  pyproject.toml, requirements.txt
  ✅  código fuente, tests, migrations
```

```bash
# .gitignore — asegúrate de incluir:
.env
.env.local
.env.production
*.pem
*.key
secrets/
```

---

## 21. pip vs poetry vs uv — gestión de dependencias en detalle

### Comparativa

```
┌─────────────────┬────────────────────────────────────────────────────────────┐
│  Herramienta    │  Características                                           │
├─────────────────┼────────────────────────────────────────────────────────────┤
│  pip            │  El estándar. Simple. Sin lockfile nativo. Lento.          │
│                 │  Usa requirements.txt (pin manual) o pyproject.toml        │
├─────────────────┼────────────────────────────────────────────────────────────┤
│  poetry         │  Gestión completa: deps + venv + publish a PyPI.           │
│                 │  Genera poetry.lock (lockfile). Más lento que uv.          │
│                 │  pyproject.toml propio con [tool.poetry].                  │
├─────────────────┼────────────────────────────────────────────────────────────┤
│  uv             │  ★ Recomendado 2024+. Ultra rápido (escrito en Rust).     │
│                 │  Lockfile nativo (uv.lock). Compatible con pyproject.toml  │
│                 │  estándar. Reemplaza pip + venv + pip-tools.               │
└─────────────────┴────────────────────────────────────────────────────────────┘
```

### uv — el gestor moderno (recomendado)

```bash
# Instalar uv una sola vez
curl -LsSf https://astral.sh/uv/install.sh | sh

# Crear proyecto nuevo con uv
uv init my_app
cd my_app

# Añadir dependencias (edita pyproject.toml + genera uv.lock)
uv add fastapi uvicorn sqlalchemy pydantic pydantic-settings
uv add --dev pytest ruff mypy

# Instalar todas las dependencias (lee uv.lock)
uv sync

# Ejecutar sin activar el venv manualmente
uv run uvicorn app.main:app --reload
uv run pytest

# Ver el árbol de dependencias (incluyendo las transitivas)
uv tree

# Actualizar una dependencia
uv add fastapi@latest

# Exportar a requirements.txt (para Docker)
uv export --format requirements-txt > requirements.txt
```

### Diferencia entre `==`, `>=` y `~=` en versiones

```
pyproject.toml — cómo especificar versiones:

fastapi==0.110.0      # pin exacto — siempre esta versión (máxima reproducibilidad)
fastapi>=0.110.0      # mínimo 0.110.0 — puede subir a 0.200, 1.0, etc.
fastapi~=0.110.0      # compatible release — permite 0.110.x pero no 0.111.0
                      # equivalente a >=0.110.0, <0.111.0
fastapi>=0.110.0,<1.0 # rango explícito — recomendado en librerías

Recomendación para aplicaciones (no librerías):
  - En pyproject.toml usa  >=versión  (flexible al instalar)
  - El lockfile (uv.lock / poetry.lock) fija las versiones exactas
  - Commitea el lockfile → reproducibilidad garantizada
```

### Lockfile — qué es y por qué importa

```
Sin lockfile (problemático):
  Desarrollador A instala hoy:  fastapi 0.110.0 (la última)
  Desarrollador B instala mañana: fastapi 0.111.0 (salió una nueva versión)
  CI instala pasado mañana: fastapi 0.111.1
  → Tres versiones distintas → "en mi máquina funciona"

Con lockfile (correcto):
  uv.lock / poetry.lock registra las versiones EXACTAS de TODAS las dependencias
  (incluyendo las transitivas — las dependencias de tus dependencias)

  Todos los desarrolladores y CI instalan exactamente lo mismo:
  uv sync  →  lee uv.lock  →  instala fastapi==0.110.0 siempre ✅

¿Qué commitear?
  ✅ pyproject.toml  (los rangos de versión que acepta tu app)
  ✅ uv.lock / poetry.lock  (las versiones exactas para reproducibilidad)
  ❌ requirements.txt generado  (redundante si tienes lockfile)
```

### Auditoría de seguridad de dependencias

```bash
# pip-audit — escanea vulnerabilidades (CVEs) en tus dependencias
pip install pip-audit
pip-audit

# Salida típica:
# Found 2 known vulnerabilities in 1 package
# Name    Version  ID             Fix Versions
# ------  -------  -------------  ------------
# pillow  9.0.0    GHSA-56pw-mpj4  9.0.1

# Integración en CI (falla el pipeline si hay vulnerabilidades)
pip-audit --strict   # sale con código 1 si hay vulnerabilidades

# Con uv
uv run pip-audit

# safety — alternativa popular
pip install safety
safety check
```

---

## 22. Calidad de código — ruff, mypy y pre-commit

### 22.1 ruff — linter y formatter (reemplaza flake8 + black + isort)

```
JAVA                              PYTHON
──────────────────────────────    ──────────────────────────────
Checkstyle (linting)              ruff check
Spotless / google-java-format     ruff format
```

```bash
# Instalar
pip install ruff

# Comprobar errores de estilo
ruff check app/

# Corregir automáticamente lo que pueda
ruff check --fix app/

# Formatear el código (equivalente a black)
ruff format app/

# Comprobar sin modificar (para CI)
ruff format --check app/
```

```toml
# pyproject.toml — configuración de ruff

[tool.ruff]
line-length = 100           # longitud máxima de línea

[tool.ruff.lint]
select = [
    "E",   # pycodestyle errors
    "W",   # pycodestyle warnings
    "F",   # pyflakes (imports no usados, variables no definidas...)
    "I",   # isort (orden de imports)
    "B",   # bugbear (errores comunes de lógica)
    "UP",  # pyupgrade (modernizar sintaxis a Python 3.10+)
]
ignore = ["E501"]           # ignorar líneas largas (ya lo controla line-length)

[tool.ruff.lint.isort]
known-first-party = ["app"] # tus módulos propios van en la sección "first party"
```

---

### 22.2 mypy — verificación de tipos (equivalente al compilador Java)

```
Java: el compilador detecta errores de tipos antes de ejecutar.
Python: mypy hace lo mismo — analiza los type hints en estático.
```

```bash
pip install mypy

# Verificar tipos
mypy app/

# Con configuración estricta (recomendado)
mypy app/ --strict
```

```toml
# pyproject.toml

[tool.mypy]
python_version = "3.12"
strict = true               # activa todas las comprobaciones
ignore_missing_imports = true  # no fallar si una librería no tiene stubs
```

```python
# Ejemplo: mypy detecta errores antes de ejecutar

def get_user(user_id: int) -> User:
    return None   # ❌ mypy error: Incompatible return value type (got "None", expected "User")
                  # En Java el compilador no compilaría esto

def process(users: list[User]) -> None:
    for u in users:
        print(u.nonexistent_field)  # ❌ mypy error: "User" has no attribute "nonexistent_field"
```

---

### 22.3 pre-commit — hooks automáticos antes de cada commit

```
Sin pre-commit:                         Con pre-commit:
  git commit -m "fix"           →         git commit -m "fix"
  → se sube código con errores            → ruff check (¿hay errores de estilo?)
                                          → ruff format (¿está formateado?)
                                          → mypy (¿hay errores de tipos?)
                                          → Si alguno falla: commit bloqueado ❌
                                          → El dev corrige y vuelve a commitear ✅
```

```bash
# Instalar pre-commit
pip install pre-commit

# Activar los hooks en el repositorio (una sola vez por desarrollador)
pre-commit install
```

```yaml
# .pre-commit-config.yaml — en la raíz del proyecto

repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.0
    hooks:
      - id: ruff           # linting
        args: [--fix]
      - id: ruff-format    # formatting

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.9.0
    hooks:
      - id: mypy
        additional_dependencies: [pydantic, sqlalchemy]

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace    # elimina espacios al final de línea
      - id: end-of-file-fixer      # asegura que el fichero termina en newline
      - id: check-merge-conflict   # detecta marcadores de conflicto de merge
      - id: detect-private-key     # bloquea si commiteas una clave privada ❌
```

---

## 23. CI/CD con GitHub Actions para Python

```yaml
# .github/workflows/ci.yml

name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      # Base de datos para los tests de integración (= H2 en Spring pero real)
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      # Instalar uv (mucho más rápido que pip)
      - name: Install uv
        uses: astral-sh/setup-uv@v3
        with:
          version: "latest"
          enable-cache: true          # cachea el entorno virtual entre runs

      - name: Set up Python
        run: uv python install 3.12

      - name: Install dependencies
        run: uv sync --all-extras     # instala deps de producción + dev

      # Equivalente a Checkstyle en Maven
      - name: Lint with ruff
        run: |
          uv run ruff check app/ tests/
          uv run ruff format --check app/ tests/

      # Equivalente al compilador Java (verifica tipos)
      - name: Type check with mypy
        run: uv run mypy app/ --ignore-missing-imports

      # Equivalente a mvn test con JaCoCo
      - name: Run tests with coverage
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/testdb
          JWT_SECRET: test-secret-para-ci
        run: |
          uv run pytest tests/ -v \
            --cov=app \
            --cov-report=xml \
            --cov-report=term-missing \
            --cov-fail-under=80       # falla si la cobertura baja del 80%

      # Auditoría de seguridad de dependencias
      - name: Security audit
        run: uv run pip-audit --strict

      # Subir reporte de cobertura (opcional)
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.xml
```

```
Flujo del pipeline comparado con Java/Maven:
┌──────────────────────────────────────────────────────────────────────────┐
│  JAVA / MAVEN                     PYTHON / UV                            │
├───────────────────────────────────┬──────────────────────────────────────┤
│  actions/setup-java               │  astral-sh/setup-uv                  │
│  mvn dependency:resolve           │  uv sync                             │
│  mvn checkstyle:check             │  ruff check + ruff format --check    │
│  (no hay equivalente nativo)      │  mypy                                │
│  mvn test                         │  pytest                              │
│  mvn jacoco:report                │  pytest --cov                        │
│  (no incluido por defecto)        │  pip-audit                           │
└───────────────────────────────────┴──────────────────────────────────────┘
```

---

## 24. Patrones de diseño en Python

### 24.1 Repository Pattern — abstracción del acceso a datos

```
JAVA                              PYTHON
──────────────────────────────    ──────────────────────────────
JpaRepository<User, Long>         clase con ABC (interfaz abstracta)
interface UserRepository          class UserRepository(ABC)
@Repository                       clase concreta normal
```

```
Diagrama de capas:
┌──────────────────────────────────────────────────────────────────┐
│  Router (HTTP)                                                   │
│  └── llama a → UserService                                       │
│                └── llama a → UserRepository (interfaz ABC)       │
│                              ├── SqlAlchemyUserRepository (real) │
│                              └── InMemoryUserRepository (tests)  │
└──────────────────────────────────────────────────────────────────┘
```

```python
# app/repositories/base.py

from abc import ABC, abstractmethod
from app.models.user import User

class UserRepositoryABC(ABC):
    """Interfaz — define el contrato, no la implementación."""

    @abstractmethod
    def find_by_id(self, user_id: int) -> User | None: ...

    @abstractmethod
    def find_by_email(self, email: str) -> User | None: ...

    @abstractmethod
    def save(self, user: User) -> User: ...

    @abstractmethod
    def delete(self, user_id: int) -> None: ...

    @abstractmethod
    def find_all(self) -> list[User]: ...
```

```python
# app/repositories/user_repository.py — implementación real con SQLAlchemy

from sqlalchemy.orm import Session
from app.repositories.base import UserRepositoryABC
from app.models.user import User

class SqlAlchemyUserRepository(UserRepositoryABC):

    def __init__(self, db: Session):
        self.db = db

    def find_by_id(self, user_id: int) -> User | None:
        return self.db.query(User).filter(User.id == user_id).first()

    def find_by_email(self, email: str) -> User | None:
        return self.db.query(User).filter(User.email == email).first()

    def save(self, user: User) -> User:
        self.db.add(user)
        self.db.commit()
        self.db.refresh(user)
        return user

    def delete(self, user_id: int) -> None:
        user = self.find_by_id(user_id)
        if user:
            self.db.delete(user)
            self.db.commit()

    def find_all(self) -> list[User]:
        return self.db.query(User).all()
```

```python
# tests/repositories/in_memory_user_repository.py — implementación para tests (sin BD)

from app.repositories.base import UserRepositoryABC
from app.models.user import User

class InMemoryUserRepository(UserRepositoryABC):
    """Repositorio en memoria para tests — sin BD real, sin I/O."""

    def __init__(self):
        self._store: dict[int, User] = {}
        self._next_id = 1

    def find_by_id(self, user_id: int) -> User | None:
        return self._store.get(user_id)

    def find_by_email(self, email: str) -> User | None:
        return next((u for u in self._store.values() if u.email == email), None)

    def save(self, user: User) -> User:
        if not user.id:
            user.id = self._next_id
            self._next_id += 1
        self._store[user.id] = user
        return user

    def delete(self, user_id: int) -> None:
        self._store.pop(user_id, None)

    def find_all(self) -> list[User]:
        return list(self._store.values())
```

```python
# tests/test_user_service.py — test unitario del service SIN BD (usando InMemory)

from app.services.user_service import UserService
from app.schemas.user import UserCreateRequest
from tests.repositories.in_memory_user_repository import InMemoryUserRepository

def test_create_user_sets_hashed_password():
    repo = InMemoryUserRepository()
    service = UserService(repo)                          # inyectamos el repo fake

    request = UserCreateRequest(name="Ana", email="ana@test.com", password="pass1234")
    user = service.create(request)

    assert user.id is not None
    assert user.email == "ana@test.com"
    assert user.password_hash != "pass1234"   # debe estar hasheado

def test_create_duplicate_email_raises():
    repo = InMemoryUserRepository()
    service = UserService(repo)

    service.create(UserCreateRequest(name="A", email="dup@test.com", password="12345678"))

    import pytest
    with pytest.raises(Exception):            # debe lanzar excepción
        service.create(UserCreateRequest(name="B", email="dup@test.com", password="12345678"))
```

---

### 24.2 Service Layer — orquestación de lógica de negocio

```python
# app/services/user_service.py
# Recibe el repositorio por constructor → fácil de testear con InMemory

from app.repositories.base import UserRepositoryABC
from app.schemas.user import UserCreateRequest
from app.models.user import User
from passlib.context import CryptContext
from fastapi import HTTPException, status

pwd_context = CryptContext(schemes=["bcrypt"])

class UserService:
    def __init__(self, repo: UserRepositoryABC):   # depende de la INTERFAZ, no de SQLAlchemy
        self.repo = repo

    def create(self, request: UserCreateRequest) -> User:
        if self.repo.find_by_email(request.email):
            raise HTTPException(status_code=status.HTTP_409_CONFLICT,
                                detail="Email ya registrado")
        user = User(
            name=request.name,
            email=request.email,
            password_hash=pwd_context.hash(request.password),
        )
        return self.repo.save(user)

    def find_by_id(self, user_id: int) -> User:
        user = self.repo.find_by_id(user_id)
        if not user:
            raise HTTPException(status_code=404, detail="Usuario no encontrado")
        return user
```

```python
# app/dependencies.py — inyección del repositorio real en producción

from fastapi import Depends
from sqlalchemy.orm import Session
from app.core.database import SessionLocal
from app.repositories.user_repository import SqlAlchemyUserRepository
from app.services.user_service import UserService

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

def get_user_service(db: Session = Depends(get_db)) -> UserService:
    repo = SqlAlchemyUserRepository(db)   # implementación real
    return UserService(repo)
```

---

### 24.3 Singleton con módulos — cómo Python lo resuelve sin la clase Singleton

```
Java necesita la clase Singleton porque el estado es por instancia.
Python no la necesita: los módulos son singletons por defecto.
```

```python
# JAVA — patrón Singleton clásico
public class DatabasePool {
    private static DatabasePool instance;
    private DatabasePool() { }
    public static synchronized DatabasePool getInstance() {
        if (instance == null) instance = new DatabasePool();
        return instance;
    }
}
```

```python
# PYTHON — no necesitas la clase Singleton

# app/core/database.py
# Este módulo se importa una sola vez — Python lo cachea en sys.modules
# engine y SessionLocal son creados UNA SOLA VEZ aunque muchos módulos los importen

from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.core.config import settings

engine = create_engine(settings.DATABASE_URL, pool_size=10)  # ← singleton natural
SessionLocal = sessionmaker(bind=engine)                     # ← singleton natural

# Cualquier módulo que haga  from app.core.database import engine
# recibe EL MISMO objeto, no uno nuevo.
```

```python
# El settings también es singleton natural
# app/core/config.py
settings = Settings()   # se crea una sola vez cuando el módulo se importa por primera vez

# Uso en cualquier archivo:
from app.core.config import settings   # siempre el mismo objeto
```

---

### 24.4 Factory Pattern

```python
# app/factories/user_factory.py
# Útil para crear objetos complejos, especialmente en tests

from app.models.user import User
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"])

class UserFactory:
    """Crea instancias de User con valores por defecto sensibles para tests."""

    @staticmethod
    def create(
        name: str = "Test User",
        email: str = "test@example.com",
        password: str = "password123",
        role: str = "user",
    ) -> User:
        return User(
            name=name,
            email=email,
            password_hash=pwd_context.hash(password),
            role=role,
        )

    @staticmethod
    def create_admin(email: str = "admin@example.com") -> User:
        return UserFactory.create(name="Admin", email=email, role="admin")

    @staticmethod
    def create_batch(count: int) -> list[User]:
        return [
            UserFactory.create(
                name=f"User {i}",
                email=f"user{i}@example.com"
            )
            for i in range(count)
        ]
```

```python
# Uso en tests — más limpio que construir User manualmente

def test_admin_can_delete_user(client, db):
    admin = UserFactory.create_admin()
    user  = UserFactory.create(email="victim@test.com")
    db.add_all([admin, user])
    db.commit()

    # Login como admin, obtener token...
    response = client.delete(f"/api/v1/users/{user.id}", headers=admin_headers)
    assert response.status_code == 204
```

---

## 25. Estructura de proyecto — árbol completo con descripción de cada archivo

```
my_app/
│
├── pyproject.toml              ← metadatos, deps, config de ruff/mypy/pytest
├── uv.lock                     ← versiones exactas de TODAS las deps (commitear)
├── .env                        ← secretos locales (en .gitignore, NO commitear)
├── .env.example                ← plantilla del .env (SÍ commitear)
├── .pre-commit-config.yaml     ← hooks de calidad de código
├── alembic.ini                 ← configuración de migraciones
├── README.md
│
├── .github/
│   └── workflows/
│       └── ci.yml              ← pipeline CI: lint + typecheck + tests + audit
│
├── app/                        ← código fuente principal (= src/main/java)
│   ├── __init__.py             ← hace de "app" un paquete Python
│   ├── main.py                 ← FastAPI app + middlewares + routers
│   ├── dependencies.py         ← get_db, get_current_user, get_user_service
│   │
│   ├── core/                   ← configuración transversal (= config/ en Spring)
│   │   ├── __init__.py
│   │   ├── config.py           ← Settings(BaseSettings) — lee el .env
│   │   ├── security.py         ← JWT: create_token, verify_token, hash_password
│   │   └── database.py         ← engine, SessionLocal, Base — singleton natural
│   │
│   ├── routers/                ← controladores HTTP (= @RestController)
│   │   ├── __init__.py
│   │   ├── auth.py             ← POST /auth/login, POST /auth/register
│   │   ├── users.py            ← CRUD /users
│   │   └── products.py
│   │
│   ├── services/               ← lógica de negocio (= @Service)
│   │   ├── __init__.py
│   │   ├── user_service.py     ← UserService(repo: UserRepositoryABC)
│   │   └── product_service.py
│   │
│   ├── repositories/           ← acceso a datos (= @Repository)
│   │   ├── __init__.py
│   │   ├── base.py             ← ABCs (interfaces)
│   │   └── user_repository.py  ← SqlAlchemyUserRepository
│   │
│   ├── models/                 ← entidades de BD (= @Entity)
│   │   ├── __init__.py
│   │   └── user.py             ← class User(Base)
│   │
│   ├── schemas/                ← DTOs request/response (= @RequestBody / Records)
│   │   ├── __init__.py
│   │   ├── user.py             ← UserCreateRequest, UserResponse
│   │   └── auth.py             ← TokenResponse, RegisterRequest
│   │
│   └── factories/              ← factories para tests (= @Builder en Lombok)
│       ├── __init__.py
│       └── user_factory.py
│
├── migrations/                 ← scripts de BD (= resources/db/migration en Flyway)
│   ├── env.py                  ← conecta Alembic con los modelos de SQLAlchemy
│   └── versions/
│       └── 001_create_users.py
│
└── tests/                      ← tests (= src/test/java)
    ├── __init__.py
    ├── conftest.py             ← fixtures globales: client, db, TestingSessionLocal
    ├── test_users.py           ← tests de integración de /users
    ├── test_auth.py            ← tests de login/register
    └── repositories/
        └── in_memory_user_repository.py  ← repositorio fake para unit tests
```

### Convenciones de nombres en Python

```
┌──────────────────────────────────────────────────────────────────────────┐
│  JAVA                         PYTHON                                     │
├──────────────────────────────┬───────────────────────────────────────────┤
│  UserService.java            │  user_service.py   (snake_case ficheros)  │
│  class UserService           │  class UserService  (PascalCase clases)   │
│  getUserById()               │  get_user_by_id()  (snake_case métodos)   │
│  MAX_RETRIES = 3             │  MAX_RETRIES = 3    (UPPER_CASE constantes)│
│  private int userId          │  self._user_id      (_prefijo = "privado")│
│  interface UserRepository    │  class UserRepositoryABC(ABC)             │
│  TestUserService.java        │  test_user_service.py                     │
│  package com.empresa.app     │  app/  (directorio = paquete)             │
└──────────────────────────────┴───────────────────────────────────────────┘

Regla para ficheros de test: el nombre siempre empieza con  test_
  → pytest los descubre automáticamente (sin anotación @Test)
```

