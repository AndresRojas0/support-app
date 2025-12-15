https://www.youtube.com/watch?v=z1Q8dcP5jWI&list=PLxooeC3-xaNdSeTjr-lgWvKSGKDWIt9Tu

# FastAPI y Next.js: Crea una App Full Stack Real (con WebSockets) 💻

# Introducción 

## Objetivo
Vamos a desarrollar un proyecto: crear una aplicación profesional de soporte técnico para resolver problemas que tienen con sus clientes. Vamos a tener una autenticación segura, crear tickets, crear conversaciones en tiempo real e incorporar notificaciones automáticas sin recargar la página, todo con FastAPI + Next.js. Hacia el final vamos a desplegar en producción en Vercel y Render, y vamos a desarrollar todo esto localmente en contenedores de Podman. 

## Entendiendo el problema
Cuál es el problema que tenemos, y cómo lo vamos a resolver: Tenemos a un usuario que se comunica con la empresa y envía un mensaje que tiene un asunto, que dice que no puede iniciar sesión. Entonces en la descripción del problema escribe de que desde ayer "me da error 403 al loguearme" y al momento que el cliente crea este mensaje, el estado de este ticket para a tener un estado "abierto". En ese momento el backend recibe una notificación de que hay alguien que tiene un problema. Entonces el técnico que recibe el problema le envía un mensaje en donde le dice "te reseteé la contraseña" y en ese preciso instante en que el usuario técnico le envía el mensaje, el cliente va a recibir una notificación sin haber recargado la página. 

## Tecnologías a usar
* Frontend vamos a desarrollar en Next.js y desplegar en Vercel. 
* Backend vamos a desarrollar en FastAPI y desplegar en Render. 
* Base de datos PostgreSQL vamos a desplegar en Render. 
* HTTP/HTTPS para comunicar frontend con backend: enviar formularios, login, cargar datos y renderizar datos en la página.
* Websockets para comunicación bidireccional en tiempo real entre el frontend y backend: recibir actualizaciones en tiempo real, nuevos mensajes, cambios de estado y notificaciones.
* Autenticación con JWT en cookies, no localStorage. Cliente --->  ↓POST /auth/login {email, password} 

## Modelos
### User
* id: int
* username: str
* role: str
* hashed-password: str
### Ticket
* id: int
* title: str
* description: str
* status: str
* user-id: int
* assigned_technician_id: int
* created_at: datetime
* updated_at: datetime
### Message
* id: int
* content: str
* ticket-id: int
* user-id: int
* created_at: datetime 
### Estructura visual del Ticket
* Ticket #...
* TÍTULO
* DETALLE
* Mensaje #1
* ...
* Mensaje #n

## Flujo de usuario
* Cliente abre el ticket.
* Soporte recibe la notificación.
* Soporte responde el mensaje.
* Cliente responde con detalles.
* Soporte responde el mensaje.
* Cliente confirma solución.
* Soporte cierra el caso.

# Estructura del backend

## Creando directorio del backend

Crear y editar la siguiente estructura de  archivos con tu editor de preferencia. 
* \support-app\backend\app\
* \support-app\backend\app\main.py
* \support-app\backend\
* \support-app\backend\Containerfile
* \support-app\backend\requirements.txt
* \support-app\infra-backend\
* \support-app\infra-backend\podman-compose.yml

## /backend/app/main.py
```
# /backend/app/main.py

from fastapi import FastAPI
app = FastAPI(title="Soporte Técnico API", version="0.1.0")

@app.get('/')
def home():
    return {"message": "Backend funcionando con FastAPI"}

@app.get('/health')
def health():
    return {"status": "Todo OK"}
```

## /backend/Containerfile 
```
# /backend/Containerfile
FROM public.ecr.aws/docker/library/python:3.12-slim

ENV PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir --upgrade pip
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

## /backend/requirements.txt 
```
# /backend/requirements.txt

fastapi==0.115.0
uvicorn[standard]==0.32.0
sqlalchemy==2.0.31
asyncpg==0.29.0
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
alembic==1.13.1
python-dotenv==1.0.1
pydantic-settings==2.3.4
```

## /backend/infra-backend/podman-compose.yml 
```
# /backend/infra-backend/podman-compose.yml
# RUTA:

services:
  db:
    image: public.ecr.aws/docker/library/postgres:16
    environment:
      POSTGRES_DB: supportdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresl/postgres_data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  api:
    build:
      context: ../backend
      dockerfile: Containerfile
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql+asyncpg://postgres:secret@db:5432/supportdb
      SECRET_KEY: change-this-in-production
    depends_on:
      - db
    restart: unless-stopped
    network:
      - support-net

volumes:
  postgres_data:

networks:
  support-net:
    name: support-net
    external: true
```


## Ejecución desde consola de trabajo (en Windows)
Dirigir a https://github.com/containers/podman/blob/main/docs/tutorials/podman-for-windows.md

Descargar .exe desde https://github.com/containers/podman/releases 

Instalar. 

Abrir CMD o Powershell:

> podman machine init

> podman machine start

> podman network create support-net

> cd ..\support-app\infra-backend\

> pip install podman-compose

> podman-compose up --build


# Estructura del frontend

## Creando directorio del frontend
Crear y editar la siguiente estructura de  archivos con tu editor de preferencia. 
* \support-app\frontend\
* \support-app\infra-frontend\

## Iniciando frontend Next.js
> cd ..\support-app\frontend\

>  npx create-next-app@latest . --typescript --tailwind --eslint --app --src-dir --use-npm --yes

## /frontend/Containerfile 
```
# /frontend/Containerfile

FROM public.ecr.aws/docker/library/node:20-slim

WORKDIR /app

COPY package*.json .

RUN npm ci --only=production
RUN npm cache clean --force
RUN npm rebuild lightningcss
RUN npm install lightningcss-linux-x64-gnu

COPY . .

EXPOSE 3000

CMD ["npm", "run", "dev", "--", "-H", "0.0.0.0"]
```

## Ejecución desde consola de trabajo (en Windows) 

> podman machine init

> podman machine start

> podman network create support-net

> cd ..\support-app\infra-frontend\

> podman-compose up --build

Si ocurre error: "error evaluating Node.js code Error: Cannot find module..." agregar en Containerfile: 
```
RUN npm rebuild lightningcss
RUN npm install lightningcss-linux-x64-gnu
```

## /backend/app/main.py 
```
# /backend/app/main.py

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(title="Soporte Técnico API", version="0.1.0")

# Configurar CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins = ["http://localhost:3000"],
    allow_credentials = True,
    allow_methods = ["*"],
    allow_headers = ["*"],
)

@app.get('/')
def home():
    return {"message": "Backend funcionando con FastAPI"}

@app.get('/health')
def health():
    return {"status": "Todo OK"}
```

## /frontend/.env
```
NEXT_PUBLIC_API_URL=http://localhost:8000
```

## /src/app/page.tsx
```
// /src/app/page.tsx

"use client"

import { useEffect, useState } from "react";

export default function HomePage() {
  const [message, setMessage] = useState<string | null>(null);
  const [health, setHealth] = useState<string | null>(null);

  useEffect(()=> {
    fetch(`${process.env.NEXT_PUBLIC_API_URL}/`)
      .then((res)=> res.json())
      .then((data)=> setMessage(data.message))
      .catch(()=> setMessage("Error al conectar con el backend"))

    fetch(`${process.env.NEXT_PUBLIC_API_URL}/health`)
      .then((res)=> res.json())
      .then((data)=> setHealth(data.status))
      .catch(()=> setHealth("Error en el healthcheck"))
  }, []);
  return (
    <main className="flex flex-col items-center justify-center min-h-screen p-8">
      <h1 className="text-3xl font-bold mb-4">Conexión del frontend con el backend</h1>
      <div className="space-y-3">
        <p>
          Mensaje del backend: <strong>{ message || "Cargando..."}</strong>
        </p>
        <p>
          Estado de salud: <strong>{ health || "Cargando..."}</strong>
        </p>
      </div>
    </main>
  )
}
```

## Ejecución desde consola de trabajo (en Windows)

> podman-compose down -v

> podman-compose up --build


# Estructura final del backend

## /backend/...
* /backend/.env
* /backend/.gitignore
* /backend/app/__init__.py
* /backend/app/dependencies.py
* /backend/app/core/__init__.py
* /backend/app/core/config.py 
* /backend/app/models/__init__.py
* /backend/app/models/base.py
* /backend/app/models/message.py
* /backend/app/models/ticket.py
* /backend/app/models/user.py
* /backend/app/routers/__init__.py
* /backend/app/schemas/__init__.py

## /backend/.env
```
DATABASE_URL= postgresql+asyncpg://postgres:secret@db:5432/supportdb
SECRET_KEY= ...clave creada por consola.
```

## /backend/.gitignore
```
.env
__pycache__/
*.pyc
alembic/versions/
```