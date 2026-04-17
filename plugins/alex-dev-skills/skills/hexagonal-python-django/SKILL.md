---
name: hexagonal-python-django
description: Usa questa skill quando lavori su backend Python con Django + Django Ninja in architettura hexagonal (ports & adapters). Copre il layout preciso di core/ (domain/adapters/api), ORM Django come adapter separato da entities (dataclass puri), Container singleton con build order, Pydantic Settings, Service-Deps pattern, schemi con from_entity(), NinjaAPI exception handlers mappati uno-a-uno su domain errors, Logger+Audit service sempre presenti, Poetry e requirements.txt workflow. Stack-specific — se lavori in Go o altro linguaggio usa hexagonal-architecture (generica).
---

# Hexagonal Python — Django + Django Ninja

Skill stack-specific. La base teorica (ports, adapters, dependency rule, error flow) è in `hexagonal-architecture`. Qui trovi il **come** concreto per il tuo stack.

## Stack

- **Framework**: Django 5 + Django Ninja
- **Config**: Pydantic Settings (`core/config.py`) con `.env`
- **DI**: Container singleton manuale (`core/container.py`), no framework esterni
- **Package manager**: Poetry primario; `requirements.txt` auto-generato da pre-commit (`poetry export`) per Docker CI
- **DB adapter preferito**: Django ORM (ma il dominio non lo sa)
- **Task queue**: Celery (`core/tasks.py`)
- **Test**: pytest, suddivisi in `unit/` (domain), `integration/` (adapters), `e2e/` (API)

## Project structure

```
├── core/
│   ├── apps.py
│   ├── config.py                          # Pydantic Settings
│   ├── container.py                       # Singleton DI
│   ├── tasks.py                           # Celery tasks
│   ├── domain/
│   │   ├── errors.py                      # DomainException hierarchy
│   │   ├── entities/
│   │   │   └── <name>.py                  # @dataclass puri
│   │   ├── ports/
│   │   │   ├── drivers/
│   │   │   │   └── <port>/
│   │   │   │       ├── port.py            # ABC
│   │   │   │       └── errors.py          # Port-specific domain errors
│   │   │   └── repositories/
│   │   │       └── <port>/
│   │   │           ├── port.py
│   │   │           └── errors.py
│   │   ├── services/
│   │   │   └── <service>/
│   │   │       └── service.py
│   │   └── utils/                         # Solo utility pure
│   ├── adapters/
│   │   ├── drivers/
│   │   │   └── <port>/<impl>/
│   │   │       ├── driver.py
│   │   │       └── errors.py              # Adapter-internal, mappati a domain
│   │   └── repositories/
│   │       └── <port>/<impl>/
│   │           ├── repo.py
│   │           └── errors.py
│   ├── infrastructure/                    # Wrapper per librerie esterne riusate
│   └── api/
│       ├── v1.py                          # NinjaAPI + exception handlers
│       ├── auth/
│       │   └── v1/auth.py
│       ├── routes/
│       │   └── v1/
│       │       ├── routers.py             # Aggrega i router v1
│       │       └── <resource>/router.py
│       └── schemas/
│           └── v1/<resource>.py
├── <project_name>/                        # Django project config
│   ├── settings.py
│   ├── urls.py
│   ├── asgi.py, wsgi.py
│   └── utils/models_abc.py
├── <django_app>/                          # Django app SOLO per ORM models
│   ├── models.py                          # Django Models, NON entities
│   ├── migrations/
│   └── admin.py
├── tests/
│   ├── conftest.py
│   ├── fixtures/
│   ├── unit/core/domain/services/
│   ├── integration/core/adapters/
│   └── e2e/
├── pyproject.toml                         # Poetry
├── poetry.lock, poetry.toml
├── requirements.txt                       # Auto-generato (pre-commit)
├── pytest.ini, Makefile, run.sh
```

**Regola chiave**: il dominio vive in `core/domain/`. Non importa **mai** nulla da `core/adapters/`, `<django_app>/`, o librerie di framework. Se ti serve un `from django...` nel dominio, stai sbagliando layer.

## Entities

`@dataclass` puri. Zero dipendenze da framework.

```python
# core/domain/entities/user.py
from dataclasses import dataclass
from datetime import datetime
from typing import Optional


@dataclass
class User:
    id: str
    email: str
    username: str
    created_at: datetime
    updated_at: Optional[datetime] = None
```

## Domain errors

Gerarchia unica in `core/domain/errors.py`. Ogni port può avere i propri errori in `<port>/errors.py`, ma ereditano da `DomainException`.

```python
# core/domain/errors.py
class DomainException(Exception):
    """Base per tutti gli errori di dominio."""
    pass


class NotFoundException(DomainException): pass
class UnauthorizedException(DomainException): pass
class ForbiddenException(DomainException): pass
class ConflictException(DomainException): pass
class InvalidValueException(DomainException): pass
```

## Ports

`ABC` con metodi astratti. Vivono nel dominio. Separati tra `drivers/` (servizi esterni: email, audit, lock, storage, API esterne) e `repositories/` (persistenza).

```python
# core/domain/ports/repositories/user/port.py
from abc import ABC, abstractmethod
from typing import Optional

from core.domain.entities.user import User


class UserRepositoryPort(ABC):
    @abstractmethod
    def get_by_id(self, user_id: str) -> User: ...

    @abstractmethod
    def get_by_email(self, email: str) -> Optional[User]: ...

    @abstractmethod
    def save(self, user: User) -> User: ...

    @abstractmethod
    def delete(self, user_id: str) -> None: ...
```

```python
# core/domain/ports/drivers/email/port.py
from abc import ABC, abstractmethod


class EmailDriverPort(ABC):
    @abstractmethod
    def send(self, to: str, subject: str, body: str) -> None: ...
```

## Services

Business logic. Dipendono **solo da port**, iniettati via costruttore. Quando le dep sono più di 2, raggruppale in `@dataclass` chiamato `<Service>Deps`.

```python
# core/domain/services/user/service.py
from dataclasses import dataclass

from core.domain.entities.user import User
from core.domain.errors import ConflictException
from core.domain.ports.drivers.email.port import EmailDriverPort
from core.domain.ports.repositories.user.port import UserRepositoryPort
from core.domain.services.logger.service import LoggerService


@dataclass
class UserServiceDeps:
    user_repository: UserRepositoryPort
    email_driver: EmailDriverPort


class UserService:
    def __init__(self, deps: UserServiceDeps, logger: LoggerService):
        self.deps = deps
        self.logger = logger

    def create_user(self, email: str, username: str) -> User:
        if self.deps.user_repository.get_by_email(email):
            raise ConflictException(f"User with email {email} already exists")

        user = User(id=generate_id(), email=email, username=username, created_at=now())
        saved = self.deps.user_repository.save(user)
        self.deps.email_driver.send(email, "Welcome", f"Hello {username}")
        self.logger.info(f"User created: {saved.id}")
        return saved
```

## Adapters

### Repository adapter (Django ORM)

Implementa il port. **Mappa sempre** le eccezioni Django in domain errors. Conversione esplicita model ↔ entity via `_to_entity` / `_from_entity`.

```python
# core/adapters/repositories/user/django/repo.py
from core.domain.entities.user import User
from core.domain.errors import NotFoundException
from core.domain.ports.repositories.user.port import UserRepositoryPort
from user_app.models import UserModel  # Django model, NON importato nel dominio


class DjangoUserRepository(UserRepositoryPort):
    def get_by_id(self, user_id: str) -> User:
        try:
            model = UserModel.objects.get(id=user_id)
            return self._to_entity(model)
        except UserModel.DoesNotExist:
            raise NotFoundException(f"User {user_id} not found")

    def get_by_email(self, email: str):
        try:
            return self._to_entity(UserModel.objects.get(email=email))
        except UserModel.DoesNotExist:
            return None

    def save(self, user: User) -> User:
        model, _ = UserModel.objects.update_or_create(
            id=user.id,
            defaults={
                "email": user.email,
                "username": user.username,
                "created_at": user.created_at,
                "updated_at": user.updated_at,
            },
        )
        return self._to_entity(model)

    def delete(self, user_id: str) -> None:
        deleted, _ = UserModel.objects.filter(id=user_id).delete()
        if not deleted:
            raise NotFoundException(f"User {user_id} not found")

    @staticmethod
    def _to_entity(model: UserModel) -> User:
        return User(
            id=str(model.id),
            email=model.email,
            username=model.username,
            created_at=model.created_at,
            updated_at=model.updated_at,
        )
```

### Driver adapter (SMTP)

```python
# core/adapters/drivers/email/smtp/driver.py
import smtplib

from core.domain.errors import DomainException
from core.domain.ports.drivers.email.port import EmailDriverPort


class SmtpEmailDriver(EmailDriverPort):
    def __init__(self, host: str, port: int):
        self.host, self.port = host, port

    def send(self, to: str, subject: str, body: str) -> None:
        try:
            with smtplib.SMTP(self.host, self.port) as server:
                server.sendmail("noreply@app.com", to, f"Subject: {subject}\n\n{body}")
        except smtplib.SMTPException as e:
            raise DomainException(f"Failed to send email to {to}: {e}")
```

## Django app per ORM models

Quando usi l'ORM come repository adapter, i `Model` vivono **fuori** da `core/` in una Django app dedicata. Questi sono adapter internals, **non** entities.

```python
# user_app/models.py
from django.db import models


class UserModel(models.Model):
    id = models.UUIDField(primary_key=True)
    email = models.EmailField(unique=True)
    username = models.CharField(max_length=150)
    created_at = models.DateTimeField()
    updated_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        db_table = "users"
```

Imported **solo** da repository adapter. Mai da services, entities, schemi API.

## Config — Pydantic Settings

```python
# core/config.py
import os
from pathlib import Path
from typing import Optional, Literal

from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict

BASE_DIR = Path(__file__).resolve().parent.parent
ENV_FILE = os.getenv("ENV_FILE", ".env")


class Config(BaseSettings):
    model_config = SettingsConfigDict(env_file=ENV_FILE, env_file_encoding="utf-8")

    # Django core
    secret_key: str = Field(..., alias="SECRET_KEY")
    debug: bool = Field(False, alias="DEBUG")
    database_url: str = Field(..., alias="DATABASE_URL")
    allowed_hosts: list[str] = Field([], alias="ALLOWED_HOSTS")
    csrf_allowed_origins: list[str] = Field([], alias="CSRF_ALLOWED_ORIGINS")
    cors_allowed_origins: list[str] = Field([], alias="CORS_ALLOWED_ORIGINS")

    # App
    app_name: str = Field("my-app", alias="APP_NAME")
    environment: str = Field("production", alias="ENVIRONMENT")

    # Logging & audit (dynamic backends)
    log_backends: Optional[list[Literal["sentry", "django"]]] = Field(
        default_factory=list, alias="LOG_BACKENDS"
    )
    audit_backends: Optional[list[Literal["django"]]] = Field(
        default_factory=list, alias="AUDIT_BACKENDS"
    )
    log_level: Optional[Literal["INFO", "DEBUG", "ERROR"]] = Field("INFO", alias="LOG_LEVEL")


config = Config()
```

`settings.py` delega **tutto** a `config`: `SECRET_KEY = config.secret_key`, `DATABASES` via `dj-database-url`, logging configurato dinamicamente da `config.log_backends`.

## Container — Singleton DI

```python
# core/container.py
from typing import Any, Dict

from core.config import config


class Container:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._init()
        return cls._instance

    def _init(self):
        self.config = config
        self._instances: Dict[str, Any] = {}
        self._build()

    def _build(self):
        # 1. External clients (redis, s3, ...)
        # 2. Repositories (Django ORM instances)
        # 3. Drivers statici (smtp, stripe, keycloak)
        # 4. Drivers dinamici (logger/audit backends da config)
        # 5. Logger service (sempre presente)
        # 6. Audit service (sempre presente)
        # 7. Domain services (usano 1–6 via port)
        pass

    def __getitem__(self, key: str) -> Any:
        return self._instances[key]

    # Typed property per ogni service
    # @property
    # def user_service(self) -> UserService:
    #     return self["user_service"]

    def override(self, key: str, instance: Any):
        """Override per test."""
        self._instances[key] = instance


container = Container()
```

**Build order obbligatorio**: external clients → repositories → static drivers → dynamic drivers → logger service → audit service → domain services. Mai invertire: un service non può essere istanziato prima dei suoi port.

## Logger & Audit — sempre presenti

Ogni progetto **deve** avere `LoggerService` e `AuditService` nel container. Il logger è un fanout su una lista di `LogDriver` (console, Django logging, Sentry). L'audit scrive su tabella Django dedicata.

```python
# core/domain/services/logger/service.py
from typing import List

from core.domain.entities.log_context import LogContext
from core.domain.ports.drivers.audit.port import AuditDriver
from core.domain.ports.drivers.log.port import LogDriver


class LoggerService:
    def __init__(self, loggers: List[LogDriver], auditors: List[AuditDriver]):
        self.loggers, self.auditors = loggers, auditors

    def info(self, msg: str, ctx: LogContext = None):
        for l in self.loggers: l.log_info(msg, ctx)

    def error(self, msg: str, ctx: LogContext = None):
        for l in self.loggers: l.log_error(msg, ctx)

    def warning(self, msg: str, ctx: LogContext = None):
        for l in self.loggers: l.log_warning(msg, ctx)

    def audit(self, action: str, ctx: LogContext = None):
        for a in self.auditors: a.journal(action, ctx)
```

Config `log_backends: ["sentry", "django"]` → driver caricati condizionalmente nel `_build()`.

## API Layer — Django Ninja

### `core/api/v1.py` — API instance + exception handlers

Mappatura 1:1 domain error → HTTP status. Non inventare nuovi codici.

```python
# core/api/v1.py
from django.core.exceptions import ValidationError
from ninja import NinjaAPI

from core.api.routes.v1.routers import v1_router
from core.config import config
from core.container import container
from core.domain import errors

api = NinjaAPI(version="0.1.0", title="my app api")
api.add_router("v1/", v1_router)


@api.exception_handler(errors.NotFoundException)
def _not_found(request, exc):
    return api.create_response(request, {"detail": str(exc), "status": "not_found"}, status=404)


@api.exception_handler(errors.UnauthorizedException)
def _unauthorized(request, exc):
    return api.create_response(request, {"detail": str(exc), "status": "unauthorized"}, status=401)


@api.exception_handler(errors.ForbiddenException)
def _forbidden(request, exc):
    return api.create_response(request, {"detail": str(exc), "status": "forbidden"}, status=403)


@api.exception_handler(errors.ConflictException)
def _conflict(request, exc):
    return api.create_response(request, {"detail": str(exc), "status": "conflict"}, status=409)


@api.exception_handler(errors.InvalidValueException)
def _invalid(request, exc):
    return api.create_response(request, {"detail": str(exc), "status": "invalid_value"}, status=422)


@api.exception_handler(ValidationError)
def _validation(request, exc):
    return api.create_response(request, {"detail": str(exc), "status": "invalid_value"}, status=422)


@api.exception_handler(Exception)
def _default(request, exc):
    container.logger_service.error(f"unhandled exception: {exc}")
    if config.debug:
        raise exc
    return api.create_response(request, {"detail": str(exc)}, status=500)
```

### Schemas — `from_entity()`

```python
# core/api/schemas/v1/user.py
from ninja import Schema


class UserOut(Schema):
    id: str
    email: str
    username: str

    @classmethod
    def from_entity(cls, entity) -> "UserOut":
        return cls(id=entity.id, email=entity.email, username=entity.username)


class UserIn(Schema):
    email: str
    username: str
```

### Routes — sottili, zero logica

```python
# core/api/routes/v1/user/router.py
from ninja import Router

from core.api.schemas.v1.user import UserIn, UserOut
from core.container import container

router = Router(tags=["users"])


@router.post("/", response={201: UserOut})
def create_user(request, payload: UserIn):
    user = container.user_service.create_user(email=payload.email, username=payload.username)
    return 201, UserOut.from_entity(user)


@router.get("/{user_id}", response=UserOut)
def get_user(request, user_id: str):
    return UserOut.from_entity(container.user_service.get_by_id(user_id))
```

## CORS

Sempre `django-cors-headers`:

```python
# settings.py
INSTALLED_APPS = [..., "corsheaders", ...]
MIDDLEWARE = [
    "corsheaders.middleware.CorsMiddleware",  # PRIMA di CommonMiddleware
    ...,
]
CORS_ALLOWED_ORIGINS = config.cors_allowed_origins
```

## Package management

- **Poetry** primario: `pyproject.toml` + `poetry.lock` committati
- `requirements.txt` auto-generato da **pre-commit hook** (`poetry export --format=requirements.txt --without-hashes`) — per Docker CI/CD, così il container non installa Poetry
- `poetry.toml` committato per lockare la config locale (es. virtualenv in `.venv`)

## Convenzioni (riassunto)

| Tipo | Path | Forma |
|---|---|---|
| Entity | `core/domain/entities/<name>.py` | `@dataclass` |
| Domain error | `core/domain/errors.py` (o `<port>/errors.py`) | classe che estende `DomainException` |
| Port (driver) | `core/domain/ports/drivers/<name>/port.py` | `ABC` |
| Port (repo) | `core/domain/ports/repositories/<name>/port.py` | `ABC` |
| Service | `core/domain/services/<name>/service.py` | classe + `<Service>Deps` dataclass |
| Driver adapter | `core/adapters/drivers/<port>/<impl>/driver.py` | implementa port |
| Repo adapter | `core/adapters/repositories/<port>/<impl>/repo.py` | implementa port, `_to_entity`/`_from_entity` |
| Route | `core/api/routes/v1/<resource>/router.py` | Ninja `Router`, zero logica |
| Schema | `core/api/schemas/v1/<resource>.py` | Ninja `Schema` + `from_entity()` |
| Django app | `<app>/` a root | solo `models.py` + migrations, usato da repo adapter |

## Anti-pattern specifici di questo stack

- `from django.db.models import ...` dentro `core/domain/` → **vietato**
- Entity che eredita da `models.Model` → i Model sono **adapter**, non entity
- Route che fa `UserModel.objects.filter(...)` direttamente → bypassa il port, rompe il layering
- Container che istanzia services prima dei port → refactor l'ordine del `_build()`
- Aggiungere un exception handler ad hoc per caso speciale → se non è mappabile su un `DomainException` esistente, crea un nuovo tipo di errore nel dominio, non un handler ad hoc

## Exit criteria

- [ ] Layout `core/` rispettato (domain/adapters/api separati, niente import cross-layer illeciti)
- [ ] Entities sono `@dataclass` puri, zero import framework
- [ ] Ogni port ha `port.py` (ABC) e `errors.py` quando serve
- [ ] Services usano solo port, `<Service>Deps` quando >2 dep
- [ ] Adapters mappano **tutti** gli errori esterni in `DomainException`
- [ ] Container `_build()` rispetta build order: external → repo → driver → dynamic → logger → audit → services
- [ ] `LoggerService` e `AuditService` presenti nel container
- [ ] NinjaAPI ha exception handler per ogni `DomainException` usato
- [ ] Schemi hanno `from_entity()`; route sono sottili
- [ ] Pydantic Settings in `core/config.py`, `settings.py` delega a `config`
- [ ] Poetry + `requirements.txt` auto-generato via pre-commit
- [ ] Django app per ORM models esiste solo se si usa ORM adapter, separata da `core/`
