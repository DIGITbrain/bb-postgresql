# PostreSQL Docker-compose

For configuration see: [docker/README](../docker/README.md)

To access PostgreSQL in the container (without a forwarded port) use a command like this:

```
docker exec -it docker-compose_postgres_1 psql default
```