# General

## Deployment type

Docker

## Image

Based on official image on Docker Hub: https://hub.docker.com/_/postgres

## Licence

PostgreSQL License (https://www.postgresql.org/about/licence)

## Version

13.3

## Description

PostgreSQL is a powerful, open source object-relational database system with over 30 years of active development that has earned it a strong reputation for reliability, feature robustness, and performance.

# Deployment

General example:

```sh
docker run -d --rm \
        --name postgres \
        -e POSTGRES_DB=mydatabase \
        -e POSTGRES_USER=admin \
        -e POSTGRES_PASSWORD=p4ss \
        -p 5432:5432 \
        postgres:13.3
```

See [1] for more details.


Available environment variables:
- `POSTGRES_DB`
- `POSTGRES_USER`
- `POSTGRES_PASSWORD`

## Parameters

|Name|Value|Description|
|-|-|-|
|Ports|`-p 5432:5432`| PostgreSQL port |

For additional configuration edit `/etc/postgresql/postgresql.conf` check [2].

## Testing

```sh
docker exec -it postgres \
        psql -U admin \
        -d mydatabase \
        -h localhost \
        -p 5432

psql (13.3 (Debian 13.3-1.pgdg100+1))
Type "help" for help.
mydatabase=#
```

Commands:
- \l -- show databases
- \dt -- show tables
- \du -- show users
- CREATE TABLE users ( user_id serial PRIMARY KEY );

# Authentication

Use:
```sh
-e POSTGRES_USER
-e POSTGRES_PASSWORD
```

__Notes__:
- no password required from `localhost`
- `POSTGRES_USER` will be superuser (not only in the created database)

## Testing

```sh
docker exec -it postgres \
        psql -U admin \
        -d mydatabase \
        -h host \
        -p 5432

Password for user admin:
```

# TLS

__Notes__:
- Requires that `OpenSSL` is installed on both client and server.
- expected `server.crt` and `server.key` respectively, in the data directory

Create server certificates (`ca.crt`, `server.key`, `server.crt`):

```sh
mkdir $HOME/certs

openssl genrsa -out certs/ca.key 4096
openssl req -x509 -new -nodes -sha256 \
        -key certs/ca.key \
        -days 3650 \
        -subj '/O=PostgreSQL Test/CN=Certificate Authority' \
        -out certs/ca.crt

openssl genrsa -out certs/server.key 2048
chmod og-rwx certs/server.key
chown 999:999 certs/server.key
openssl req -new -sha256 \
        -key certs/server.key \
        -subj '/O=PostgreSQL Test/CN=Server' | openssl x509 -req -sha256 \
        -CA certs/ca.crt \
        -CAkey certs/ca.key \
        -CAserial certs/ca.txt \
        -CAcreateserial \
        -days 365 \
        -out certs/server.crt
```

Run with option: ssl=on:

```sh
docker run -d --rm \
        --name postgres \
        -e POSTGRES_DB=mydatabase \
        -e POSTGRES_USER=admin \
        -e POSTGRES_PASSWORD=p4ss \
        -p 5432:5432 \
        -v $PWD/certs/server.crt:/var/lib/postgresql/server.crt \
        -v $PWD/certs/server.key:/var/lib/postgresql/server.key \
        postgres:13.2 \
        -c ssl=on \
        -c ssl_cert_file=/var/lib/postgresql/server.crt \
        -c ssl_key_file=/var/lib/postgresql/server.key
```

## Testing

```sh
psql host=<IP> \
        port=5432 \
        user=admin \
        dbname=mydatabase \
        sslmode=verify-full \
        sslcert=client.crt \
        sslkey=/var/lib/postgresql/client.key \
        sslrootcert=/var/lib/postgresql/ca.crt \
        sslmode=require
```

```sh
Password for user admin:

psql (13.3 (Debian 13.3-1.pgdg100+1))

SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
```

Available sslmode:
| sslmode 	| Eavesdropping protection 	| MITM protection 	| Statement 	|
|-	|-	|-	|-	|
| `disable` 	| No 	| No 	| I don't care about security, and I don't want to pay the overhead of encryption. 	|
| `allow` 	| Maybe 	| No 	| I don't care about security, but I will pay the overhead of encryption if the server insists on it. 	|
| `prefer` 	| Maybe 	| No 	| I don't care about encryption, but I wish to pay the overhead of encryption if the server supports it. 	|
| `require` 	| Yes 	| No 	| I want my data to be encrypted, and I accept the overhead. I trust that the network will make sure I always connect to the server I want. 	|
| `verify-ca` 	| Yes 	| Depends on CA policy 	| I want my data encrypted, and I accept the overhead. I want to be sure that I connect to a server that I trust. 	|
| `verify-full` 	| Yes 	| Yes 	| I want my data encrypted, and I accept the overhead. I want to be sure that I connect to a server I trust, and that it's the one I specify. 	|

<br/>
See [3] Table 31-3. for more details.

With server certificate verification - but without hotname verification:

```sh
docker exec -it postgres
        psql host=IP \
        port=5432 \
        user=admin \
        dbname=mydatabase \
        sslmode=verify-ca \
        sslrootcert=/var/lib/postgresql/ca.crt
```


# TLS mutual authentication [4]

__NOT POSSIBLE VIA DOCKER__

Init script does not allow hostssl=cert setting.

The manual solution would be to add line to /var/lib/postgresql/data/pg_hba.conf:

```sh
# to use only TLS authentication, remove md5
hostssl all myuser 0.0.0.0/0               md5 clientcert=1
```
__Note__:
- it is not working: -e `POSTGRES_INITDB_ARGS`="--auth-hostssl=cert"
- it is not working: -e `POSTGRES_INITDB_ARGS`="--auth-host=cert"
- it is not working: -e `POSTGRES_HOST_AUTH_METHOD`=cert
- it is not working: -v `$PWD/pg_hba.conf:/var/lib/postgresql/data/pg_hba.conf`


The initdb location is under /usr/lib/postgresql/13/bin/initdb .

# References

[1] https://www.postgresql.org/docs/9.3/config-setting.html

[2] https://www.postgresql.org/docs/current/app-postgres.html

[3] https://www.postgresql.org/docs/current/libpq-ssl.html

[4] https://smallstep.com/hello-mtls/doc/server/postgresql