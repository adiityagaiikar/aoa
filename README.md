# Zabbix Docker Setup (Windows)

This project runs Zabbix Server + Zabbix Web + MySQL using Docker Compose.

## Requirements

- Docker Desktop installed and running
- Windows CMD or PowerShell

## 1) First-time startup (clean and reliable)

Run these commands from this folder:

```bat
docker compose -f docker-compose.yml down -v
docker compose -f docker-compose.yml up -d db
```

Wait until MySQL is healthy:

```bat
docker inspect -f "{{.State.Status}} health={{if .State.Health}}{{.State.Health.Status}}{{else}}n/a{{end}}" zabbix-db
```

When it shows `health=healthy`, initialize the Zabbix database:

```bat
docker exec zabbix-db mysql -uroot -prootpassword -e "DROP DATABASE IF EXISTS zabbix; CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin; GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'%'; FLUSH PRIVILEGES;"
docker run --rm --network zabbix_default zabbix/zabbix-server-mysql:ubuntu-7.0-latest sh -c "zcat /usr/share/doc/zabbix-server-mysql/create.sql.gz" | docker exec -i zabbix-db mysql -uroot -prootpassword zabbix
```

Optional verification:

```bat
docker exec zabbix-db mysql -uroot -prootpassword -e "USE zabbix; SELECT COUNT(*) AS users_count FROM users; SELECT mandatory,optional FROM dbversion;"
```

Expected:
- `users_count` should be `2`
- `mandatory` should be `7000000`

Start the remaining services:

```bat
docker compose -f docker-compose.yml up -d zabbix-server zabbix-web
```

Open UI:
- http://localhost:8081

## 2) Normal start (after first-time setup)

```bat
docker compose -f docker-compose.yml up -d
```

## 3) Stop

```bat
docker compose -f docker-compose.yml down
```

## 4) Full reset (delete all Zabbix data)

```bat
docker compose -f docker-compose.yml down -v
```

Then run the first-time startup steps again.

## 5) Health checks

```bat
docker compose -f docker-compose.yml ps -a
docker inspect -f "{{.Name}} restartCount={{.RestartCount}} status={{.State.Status}} health={{if .State.Health}}{{.State.Health.Status}}{{else}}n/a{{end}}" zabbix-db zabbix-server zabbix-web
```

Healthy output should look like:
- `zabbix-db`: running, healthy, restartCount=0
- `zabbix-server`: running, restartCount=0
- `zabbix-web`: running, healthy, restartCount=0

## 6) If the browser still shows a DB error

- Hard refresh with Ctrl+F5
- Or open http://localhost:8081 in an incognito/private window
- Re-check container state using the health checks above
