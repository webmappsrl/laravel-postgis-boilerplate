# Laravel PostGIS Boilerplate — CLAUDE.md

Stack: Laravel 12, PHP 8.4, PostgreSQL + PostGIS, Nova 5, Elasticsearch 8, Redis, Horizon.

## Comandi utili

```bash
# Formattazione codice
composer format

# Avvio ambiente locale completo (serve + horizon + pail + vite)
composer dev

# Entrare nel container PHP
docker exec -it php_${APP_NAME} bash

# Eseguire un comando artisan senza entrare nel container
docker exec -it php_${APP_NAME} php artisan <comando>
```

## Ambienti Docker

Il progetto ha tre file compose con scopi distinti:

| File | Scopo |
|------|-------|
| `compose.yml` | Base condivisa (prod). Non si usa direttamente. |
| `develop.compose.yml` | Sviluppo locale con nginx/proxy. Aggiunge minio, mailpit. |
| `local.compose.yml` | Sviluppo locale standalone con `php artisan serve`. Aggiunge scout-init, kibana, laravel server. |

```bash
# Produzione
docker compose up -d

# Sviluppo (con nginx)
docker compose -f develop.compose.yml up -d

# Sviluppo standalone (senza nginx)
docker compose -f local.compose.yml up -d
```

### Convenzioni container name

I container usano trattino come separatore: `php-${APP_NAME}`, `postgres-${APP_NAME}`, `horizon-${APP_NAME}`.
Eccezione: il `compose.yml` base usa underscore (`php_${APP_NAME}`) per compatibilità con geobox/nginx.

## Stack Elasticsearch

La sequenza di avvio è: `elasticsearch` → `elasticsearch-init` → `kibana`.

`elasticsearch-init` è un container curl one-shot che imposta la password di `kibana_system` via API (Elasticsearch non supporta variabili d'ambiente per questo utente). Si rimuove automaticamente dopo l'esecuzione.

`scout-init` (solo `local.compose.yml`) esegue `scout:import` sui modelli di wm-package dopo che Elasticsearch e il database sono pronti.

Variabili `.env` rilevanti:
```
ELASTICSEARCH_HOST=elasticsearch:9200
ELASTICSEARCH_USER=elastic
ELASTICSEARCH_PASSWORD=changeme
ELASTICSEARCH_SSL_VERIFICATION=false
DOCKER_KIBANA_PORT=5601
```

## Nova

### Gate

`NovaServiceProvider::gate()` blocca i Guest da Nova:
```php
return !$user->hasRole('Guest');
```

### Menu

Il menu è strutturato per sezioni in `NovaServiceProvider::boot()`. Le sezioni Admin e Media sono visibili solo agli Administrator. Aggiungere nuove sezioni dopo quella Media.

### Traits disponibili (`app/Nova/Traits/`)

- `FiltersUsersByRoleTrait` — filtra gli utenti relatibili per ruolo (Administrator/Validator)
- `HidesAppFromIndexTrait` — nasconde il campo `app` dalla lista index

### Footer

Il footer Nova viene renderizzato da `resources/views/nova/footer.blade.php` e mostra: nome app, versione, environment, versioni di Nova/Laravel/PHP.

## Policy

Le policy di Role e Permission (da wm-package) sono registrate in `AppServiceProvider::boot()`. Per aggiungere policy progetto-specifiche, aggiungerle nello stesso metodo seguendo lo stesso pattern:
```php
Gate::policy(MyModel::class, MyModelPolicy::class);
```

## PHPStan

```bash
vendor/bin/phpstan analyse
```

Configurazione in `phpstan.neon.dist`. La baseline è `phpstan-baseline.neon`.
