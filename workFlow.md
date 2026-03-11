# Plan Completo: App Móvil de Deportes — Sin Anuncios, Sin Suscripciones
### Arquitectura, APIs, Stack Tecnológico y Roadmap

---

## 1. Tu Situación y Decisiones Clave

**Lo que tienes:** Conocimiento de React (aprendiendo), experiencia con Docker, servidor personal, ganas de aprender.
**Lo que necesitas:** App móvil multideporte con datos en vivo, para ~10 usuarios, hosteada en tu servidor.
**Lo que NO tienes:** Experiencia en desarrollo móvil.

---

## 2. Framework Móvil: Expo (React Native) — Sin Duda

Dado que estás aprendiendo React, la elección natural es **Expo + React Native**. No Flutter, no nativo. La razón es simple y directa:

**¿Por qué Expo y no otra cosa?**

Tu conocimiento de React se traduce directamente a React Native. Los componentes funcionan igual, los hooks son los mismos, el state management es idéntico. En lugar de `<div>` usas `<View>`, en lugar de `<p>` usas `<Text>`. Eso es prácticamente el 80% de la diferencia.

Expo es un framework que envuelve React Native y elimina toda la complejidad de configuración nativa. No necesitas Android Studio, no necesitas Xcode para empezar. Con la app **Expo Go** en tu teléfono, escaneas un QR y ves tu app corriendo en tiempo real mientras programas. Es la experiencia de desarrollo más rápida que existe para alguien que viene del mundo web.

Además, Expo tiene más de 40,000 estrellas en GitHub, más de 3 millones de usuarios, y es usado por empresas grandes en producción. No es un juguete.

**La curva de aprendizaje realista:**

Si ya manejas React básico (componentes, hooks, estado), puedes tener tu primera pantalla funcional en React Native en un par de días. El ecosistema de npm completo está disponible, así que todas las librerías que ya conoces o aprenderás en web te sirven en mobile también.

**¿Y el futuro?** Expo soporta iOS, Android y Web desde el mismo codebase. Si en algún momento quieres que tu app también funcione como página web, ya está cubierto.

---

## 3. Arquitectura del Sistema Completo

```
┌─────────────────────────────────────────────────────┐
│                  TU SERVIDOR PERSONAL                │
│                   (Docker Compose)                   │
│                                                      │
│  ┌──────────────┐  ┌──────────┐  ┌───────────────┐  │
│  │   Node.js    │  │  Redis   │  │    Nginx       │  │
│  │   Express    │←→│  Cache   │  │  Reverse Proxy │  │
│  │  (API Proxy) │  │          │  │  + SSL         │  │
│  └──────┬───────┘  └──────────┘  └───────────────┘  │
│         │                                            │
└─────────┼────────────────────────────────────────────┘
          │
          │ Consulta APIs externas
          │ (solo cuando el cache expiró)
          │
    ┌─────┴──────────────────────────────┐
    │                                     │
    ▼              ▼              ▼       ▼
┌────────┐  ┌──────────┐  ┌────────┐  ┌──────────┐
│API-    │  │OpenLiga  │  │Balldont│  │TheSports │
│Sports  │  │DB        │  │lie     │  │DB        │
│(9+ dep)│  │(Fútbol EU│  │(US+EU) │  │(Imágenes)│
│100r/día│  │ilimitado)│  │(gratis)│  │(logos)   │
└────────┘  └──────────┘  └────────┘  └──────────┘

    ┌──────────────────────────────────────────┐
    │            USUARIOS (~10)                 │
    │                                           │
    │  📱 App Expo     📱 App Expo     📱 Web  │
    │  (Android)       (iOS)          (browser) │
    └──────────────────────────────────────────┘
```

### ¿Por qué un Backend Proxy y no llamar las APIs directo desde la app?

Hay tres razones importantes:

1. **Seguridad:** Las API keys no deben ir embebidas en una app móvil. Cualquiera puede decompilar un APK y extraerlas. Tu backend guarda las keys de forma segura.

2. **Optimización de requests:** Con 10 usuarios pidiendo los mismos datos (ej: resultado del partido de Chile), tu backend hace UNA sola petición a la API y sirve la respuesta cacheada a todos. Así un free tier de 100 req/día rinde mucho más.

3. **API unificada:** Tu app móvil habla con UN solo endpoint (tu servidor). El backend se encarga de orquestar entre las diferentes APIs y devolver datos en un formato unificado. Si mañana cambias una API por otra, la app no necesita actualizarse.

---

## 4. Estrategia de APIs: Máxima Cobertura Gratis

### Distribución por Deporte y API

| Deporte/Liga | API Principal | API Complemento | Tipo de Datos |
|---|---|---|---|
| **Fútbol (Ligas EU)** | OpenLigaDB | API-Sports | Live scores, fixtures, tablas |
| **Fútbol (Global)** | API-Sports | Football-Data.org | Fixtures, stats, lineups |
| **NBA / WNBA** | BALLDONTLIE | API-Sports | Stats, box scores, standings |
| **NFL** | BALLDONTLIE | API-Sports | Schedules, stats |
| **MLB** | BALLDONTLIE | — | Scores, stats |
| **NHL** | BALLDONTLIE | API-Sports | Scores, standings |
| **F1** | API-Sports | — | Carreras, rankings, pilotos |
| **MMA/UFC** | BALLDONTLIE | API-Sports | Peleas, fighters |
| **Imágenes/Logos** | TheSportsDB | — | Logos, fotos, badges |

### Presupuesto Total de Requests Diarios (Gratis)

```
API-Sports:      100 req/día  (por cada sub-API)
                 → ~100 fútbol + 100 NBA + 100 F1 + ... = ~900 req/día total
BALLDONTLIE:     ~300 req/día (free tier, rate limited por minuto)
OpenLigaDB:      Ilimitado
TheSportsDB:     ~4,320 req/día (30 req/min)
Football-Data:   ~14,400 req/día (10 req/min)
────────────────────────────────────────
TOTAL:           ~20,000+ req/día disponibles
```

Con cache de Redis, 10 usuarios no deberían consumir más de 500-1000 requests reales al día.

### Estrategia de Cache por Tipo de Dato

| Tipo de Dato | TTL del Cache | Razón |
|---|---|---|
| Resultados en vivo | 30-60 segundos | Necesitan estar actualizados |
| Fixtures/calendario | 6-12 horas | Cambian raramente |
| Tablas de posiciones | 1-2 horas | Se actualizan post-partido |
| Estadísticas de jugador | 2-4 horas | Se actualizan post-partido |
| Logos/imágenes | 7 días | Casi nunca cambian |
| Info de equipos/ligas | 24 horas | Muy estables |

---

## 5. Stack Tecnológico Detallado

### Backend (En tu Docker)

```
Node.js + Express      → API proxy y lógica de negocio
Redis                   → Cache de respuestas de APIs
Nginx                   → Reverse proxy, SSL, compresión
Docker Compose          → Orquestación de todos los servicios
```

### Frontend Móvil

```
Expo (React Native)     → Framework de la app
React Navigation        → Navegación entre pantallas
Axios o fetch           → HTTP client para tu API
AsyncStorage            → Cache local en el dispositivo
Expo Notifications      → Push notifications (goles, etc.)
```

### Estructura del Docker Compose

```yaml
# docker-compose.yml (concepto)
services:
  api:
    build: ./backend
    ports:
      - "3000:3000"
    environment:
      - REDIS_URL=redis://redis:6379
      - API_SPORTS_KEY=tu_key_aqui
      - BALLDONTLIE_KEY=tu_key_aqui
    depends_on:
      - redis

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - api

volumes:
  redis_data:
```

---

## 6. Diseño de la API Unificada (Tu Backend)

Tu backend expone endpoints simples y unificados. La app no necesita saber de dónde vienen los datos:

```
GET /api/sports                          → Lista de deportes disponibles
GET /api/sports/:sport/leagues           → Ligas por deporte
GET /api/leagues/:leagueId/fixtures      → Calendario de partidos
GET /api/leagues/:leagueId/standings     → Tabla de posiciones
GET /api/matches/live                    → Partidos en vivo (todos los deportes)
GET /api/matches/:matchId                → Detalle de un partido
GET /api/matches/:matchId/stats          → Estadísticas del partido
GET /api/teams/:teamId                   → Info del equipo + logo
GET /api/players/:playerId/stats         → Estadísticas del jugador
```

Internamente, cada endpoint sabe a cuál API externa consultar según el deporte. Por ejemplo, `/api/matches/live` consulta en paralelo a OpenLigaDB (fútbol EU), API-Sports (otros deportes) y BALLDONTLIE (ligas US), combina todo y devuelve una lista unificada.

---

## 7. Roadmap de Desarrollo (Orden Sugerido)

### Fase 1 — Backend Proxy (Semana 1-2)
Lo que ya sabes hacer, pero enfocado en deportes.

```
□ Crear proyecto Node.js + Express
□ Configurar Docker Compose (API + Redis + Nginx)
□ Implementar conexión con API-Sports (registrarse, obtener key)
□ Implementar conexión con OpenLigaDB (sin key necesario)
□ Implementar conexión con BALLDONTLIE (registrarse, obtener key)
□ Implementar conexión con TheSportsDB (key de test)
□ Crear middleware de cache con Redis
□ Exponer endpoints unificados
□ Testear con Postman/Insomnia desde tu red local
```

### Fase 2 — Primera App Móvil (Semana 3-4)
Tu primer contacto con Expo.

```
□ Instalar Expo CLI: npx create-expo-app sports-app
□ Instalar Expo Go en tu teléfono
□ Crear pantalla principal con lista de deportes
□ Crear pantalla de partidos en vivo (la más importante)
□ Implementar navegación básica con React Navigation
□ Conectar con tu backend usando fetch/axios
□ Probar en tu teléfono vía Expo Go
```

### Fase 3 — UX y Funcionalidad (Semana 5-6)
Donde la app empieza a sentirse real.

```
□ Pantalla de detalle de partido (lineups, estadísticas)
□ Pantalla de tabla de posiciones por liga
□ Pantalla de calendario/fixtures
□ Favoritos (guardar equipos/ligas favoritas localmente)
□ Pull-to-refresh en pantallas de datos en vivo
□ Indicadores de carga y estados vacíos
□ Cache local con AsyncStorage para uso offline
```

### Fase 4 — Pulido (Semana 7-8)
La diferencia entre "funciona" y "se siente bien".

```
□ Tema oscuro/claro
□ Animaciones de transición entre pantallas
□ Push notifications para partidos de equipos favoritos
□ Auto-refresh de scores en vivo (polling cada 30-60s)
□ Búsqueda de equipos/ligas
□ Build del APK con EAS Build para instalar sin Expo Go
```

### Fase 5 — Futuro (Opcional)
```
□ Versión web usando el mismo codebase de Expo
□ Widgets en pantalla de inicio (Android)
□ Comparador de estadísticas entre jugadores
□ Historial de partidos
```

---

## 8. Primeros Pasos Concretos

### Paso 1: Registrarte en las APIs (10 minutos)

```
1. API-Sports:    https://api-sports.io       → Registrar → Obtener API Key
2. BALLDONTLIE:   https://app.balldontlie.io   → Crear cuenta → Copiar API Key
3. TheSportsDB:   https://www.thesportsdb.com  → Key de test: "3" (sí, literal)
4. OpenLigaDB:    Sin registro, acceso directo
5. Football-Data: https://www.football-data.org → Registrar → API Key por email
```

### Paso 2: Tu Primer Endpoint de Prueba (30 minutos)

Antes de todo lo demás, prueba que las APIs responden. Desde tu terminal:

```bash
# OpenLigaDB - Partidos actuales de la Bundesliga (sin key)
curl https://api.openligadb.de/getmatchdata/bl1/2025

# API-Sports - Partidos en vivo (con tu key)
curl -H "x-apisports-key: TU_KEY" \
  "https://v3.football.api-sports.io/fixtures?live=all"

# BALLDONTLIE - Partidos NBA de hoy (con tu key)
curl -H "Authorization: TU_KEY" \
  "https://api.balldontlie.io/v1/games?dates[]=2026-03-11"

# TheSportsDB - Info del equipo (key de test)
curl "https://www.thesportsdb.com/api/v1/json/3/searchteams.php?t=Arsenal"
```

### Paso 3: Tu Primera App Expo (20 minutos)

```bash
# Instalar y crear proyecto
npx create-expo-app sports-app
cd sports-app

# Iniciar el servidor de desarrollo
npx expo start

# Escanear QR con Expo Go en tu teléfono
# → Ya tienes una app corriendo
```

---

## 9. Tips de UX para que se Sienta Profesional

Dado que quieres una buena experiencia de usuario, estas son las cosas que marcan la diferencia:

**Navegación tipo tabs** en la parte inferior (como 365Scores): usa `@react-navigation/bottom-tabs`. Las tabs principales serían: En Vivo, Calendario, Favoritos, Más.

**Indicadores de estado en vivo:** Un punto verde pulsante al lado de partidos que están en juego. Hace que la app se sienta activa y en tiempo real.

**Pull-to-refresh:** En cada lista de datos, implementar el gesto de "tirar hacia abajo para actualizar". React Native lo soporta nativamente con `RefreshControl`.

**Skeleton loading:** En lugar de un spinner genérico mientras cargan los datos, mostrar "esqueletos" grises de las tarjetas de partidos. Se siente más rápido y profesional.

**Haptic feedback:** Expo tiene `expo-haptics` para dar una vibración sutil al tocar elementos. Es un detalle pequeño que hace que la app se sienta nativa.

**Tipografía y espaciado:** Usar tamaños consistentes. Scores grandes y prominentes (24-32pt), nombres de equipos medianos (16-18pt), info secundaria pequeña (12-14pt).

---

## 10. Consideraciones sobre iOS

Un punto importante: Expo Go funciona en iOS directamente para desarrollo. Pero para distribuir la app a tus ~10 usuarios con iPhone, necesitarás una cuenta de Apple Developer ($99 USD/año) o distribuir por TestFlight. Para Android, puedes generar el APK directamente y compartirlo sin costo.

Si todos tus usuarios son Android, no necesitas preocuparte por esto. Si hay usuarios iOS en tu grupo, una alternativa es que la versión web de Expo (que sale del mismo código) funcione como PWA en Safari sin necesidad de cuenta de developer.

---

*Plan creado: Marzo 2026*
*Stack: Expo (React Native) + Node.js/Express + Redis + Docker + Nginx*
*APIs: API-Sports + BALLDONTLIE + OpenLigaDB + TheSportsDB + Football-Data.org*