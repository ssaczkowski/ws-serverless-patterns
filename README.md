# Serverless Users API

API REST serverless para gestión de usuarios, construida con AWS SAM, Lambda, DynamoDB y autenticación JWT mediante Amazon Cognito.

---

## Arquitectura de Infraestructura

```mermaid
graph TB
    Client(["👤 Cliente"])

    subgraph Cognito["Amazon Cognito"]
        UserPool["User Pool"]
        UserPoolClient["App Client"]
        AdminGroup["Grupo: apiAdmins"]
    end

    subgraph APIGateway["API Gateway (REST)"]
        API["RestAPI /Prod"]
        Authorizer["Lambda Token Authorizer"]
    end

    subgraph Lambda["AWS Lambda"]
        AuthFn["AuthorizerFunction\nauthorizer.py"]
        UsersFn["UsersFunction\nusers.py"]
    end

    subgraph Storage["Amazon DynamoDB"]
        Table[("UsersTable\nPK: userid")]
    end

    Client -->|"1. Obtiene token JWT"| UserPool
    UserPool -->|"JWT Token"| Client
    Client -->|"2. Request + Authorization header"| API
    API -->|"3. Valida token"| Authorizer
    Authorizer -->|"Llama"| AuthFn
    AuthFn -->|"Verifica JWT con JWKS"| UserPool
    AuthFn -->|"4. IAM Policy (Allow/Deny)"| API
    API -->|"5. Invoca"| UsersFn
    UsersFn -->|"6. CRUD"| Table
```

---

## Flujo de Autenticación y Autorización

```mermaid
sequenceDiagram
    actor User as 👤 Usuario
    participant Cognito as Amazon Cognito
    participant APIGW as API Gateway
    participant Auth as AuthorizerFunction
    participant Users as UsersFunction
    participant DDB as DynamoDB

    User->>Cognito: initiate-auth (email + password)
    Cognito-->>User: JWT Token (IdToken)

    User->>APIGW: GET /users/{userid}<br/>Authorization: <JWT>
    APIGW->>Auth: Token + methodArn
    Auth->>Cognito: GET /.well-known/jwks.json (cold start)
    Cognito-->>Auth: Public Keys (JWKS)
    Auth->>Auth: Verifica firma, expiración y audience
    alt Token válido
        Auth-->>APIGW: IAM Policy - Allow
        APIGW->>Users: Invoca con evento
        Users->>DDB: get_item(userid)
        DDB-->>Users: Item
        Users-->>APIGW: 200 OK + datos usuario
        APIGW-->>User: 200 OK + datos usuario
    else Token inválido o expirado
        Auth-->>APIGW: 401 Unauthorized
        APIGW-->>User: 401 Unauthorized
    end
```

---

## Flujo de Autorización por Roles

```mermaid
flowchart TD
    A["Token JWT recibido"] --> B{"¿Firma válida?"}
    B -->|No| Z["❌ Unauthorized"]
    B -->|Sí| C{"¿Token expirado?"}
    C -->|Sí| Z
    C -->|No| D{"¿Audience correcto?"}
    D -->|No| Z
    D -->|Sí| E["Extraer sub (principalId)"]
    E --> F["Permitir acceso a\n/users/{sub} (GET, PUT, DELETE)"]
    F --> G{"¿Usuario en grupo\napiAdmins?"}
    G -->|Sí| H["✅ Permitir acceso total\n(GET, POST, PUT, DELETE /users/*)"]
    G -->|No| I["✅ Acceso solo a\nrecursos propios"]
```

---

## Endpoints de la API

| Método | Ruta | Descripción | Requiere Admin |
|--------|------|-------------|----------------|
| `GET` | `/users` | Lista todos los usuarios | ✅ |
| `POST` | `/users` | Crea un nuevo usuario | ✅ |
| `GET` | `/users/{userid}` | Obtiene un usuario por ID | Solo propio |
| `PUT` | `/users/{userid}` | Actualiza un usuario por ID | Solo propio |
| `DELETE` | `/users/{userid}` | Elimina un usuario por ID | Solo propio |

---

## Estructura del Proyecto

```
users/
├── src/
│   └── api/
│       ├── users.py          # Lambda handler CRUD de usuarios
│       └── authorizer.py     # Lambda authorizer JWT + IAM Policy
├── tests/
│   ├── unit/
│   │   └── test_handler.py
│   └── integration/
│       ├── conftest.py
│       └── test_api.py
├── events/                   # Eventos de prueba para SAM local
├── template.yaml             # SAM template (infraestructura)
├── samconfig.toml            # Configuración de despliegue
└── requirements.txt          # Dependencias Python
```

---

## Requisitos Previos

- [AWS CLI](https://aws.amazon.com/cli/) configurado
- [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html)
- Python 3.14
- Permisos IAM para desplegar Lambda, DynamoDB, API Gateway y Cognito

---

## Despliegue

```bash
# 1. Instalar dependencias
pip install -r requirements.txt

# 2. Build
sam build

# 3. Deploy (primera vez, modo guiado)
sam deploy --guided

# 4. Deploys siguientes (usa samconfig.toml)
sam deploy
```

El stack se desplegará en `us-east-2` con el nombre `ws-serverless-patterns-users`.

---

## Prueba Local

```bash
# Iniciar API local
sam local start-api --env-vars env.json

# Ejemplo: obtener todos los usuarios
curl -H "Authorization: <JWT_TOKEN>" http://localhost:3000/users
```

### Obtener un token JWT desde Cognito

```bash
aws cognito-idp initiate-auth \
  --auth-flow USER_PASSWORD_AUTH \
  --client-id <UserPoolClientId> \
  --auth-parameters USERNAME=<email>,PASSWORD=<password> \
  --query 'AuthenticationResult.IdToken' \
  --output text
```

---

## Variables de Entorno

| Variable | Descripción |
|----------|-------------|
| `USERS_TABLE` | Nombre de la tabla DynamoDB |
| `USER_POOL_ID` | ID del Cognito User Pool |
| `APPLICATION_CLIENT_ID` | ID del App Client de Cognito |
| `ADMIN_GROUP_NAME` | Nombre del grupo de administradores (default: `apiAdmins`) |

---

## Recursos AWS Creados

| Recurso | Tipo | Descripción |
|---------|------|-------------|
| `UsersTable` | DynamoDB Table | Almacena datos de usuarios (PK: `userid`) |
| `UsersFunction` | Lambda Function | Maneja operaciones CRUD |
| `AuthorizerFunction` | Lambda Function | Valida JWT y genera políticas IAM |
| `RestAPI` | API Gateway | REST API con authorizer Lambda |
| `UserPool` | Cognito User Pool | Gestión de identidades |
| `UserPoolClient` | Cognito App Client | Cliente con flujo `USER_PASSWORD_AUTH` |
| `UserPoolDomain` | Cognito Domain | Hosted UI para login |
| `ApiAdministratorsUserPoolGroup` | Cognito Group | Grupo con privilegios de administrador |

---

## Dependencias

```
python-jose[cryptography]  # Validación de tokens JWT
```

---

## Licencia

MIT-0 — Ver [LICENSE](https://github.com/aws/mit-0) para más detalles.
