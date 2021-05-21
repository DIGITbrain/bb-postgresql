# IMAGE SOURCE

Official image on __Docker Hub__:  https://hub.docker.com/_/postgres

# Licence

PostgreSQL License

# Version

13.2

# DEPLOYMENT

Example:

> docker run --name postgres -e POSTGRES_DB=mydatabase -e POSTGRES_USER=admin -e POSTGRES_PASSWORD=p4ss -p 5432:5432 -d postgres:13.2


VOLUMES:

        -v $HOME/postgresdata:/var/lib/postgresql/data

OPTIONS:

        -e POSTGRES_DB=mydatabase
        -e POSTGRES_USER=mydatabaseuser
        -e POSTGRES_PASSWORD=mydatabasepassword


For further configuration options use: 

        -v $HOME/my-postgres.conf:/etc/postgresql/postgresql.conf

or 

        -c shared_buffers=256MB -c max_connections=200

See further details: https://www.postgresql.org/docs/current/static/app-postgres.html

## TEST:


> docker exec -it postgres psql -U admin -d mydatabase -h localhost -p 5432
>>
>> psql (13.2 (Debian 13.2-1.pgdg100+1))
>> 
>> Type "help" for help.
>>
>> mydatabase=#


Some useful commands:
> &gt; \l -- show databases
>
> &gt; \dt -- show tables
>
> &gt; \du -- show users
>
> &gt; CREATE TABLE users ( user_id serial PRIMARY KEY );

# AUTHENTICATION

Use -e POSTGRES_USER, POSTGRES_PASSWORD

__Notes:__ 
- no password required from localhost
- POSTGRES_USER will be superuser (not only in the created database)

## TEST

> docker exec -it postgres psql -U admin -d mydatabase -h host -p 5432
>
>> Password for user admin:


# TLS

__Notes__:
- Requires that OpenSSL is installed on both client and server.
- expected server.crt and server.key, respectively, in the data directory

Create server certificates (ca.crt, server.key, server.crt):

```text
mkdir $HOME/certs
openssl genrsa -out certs/ca.key 4096
openssl req -x509 -new -nodes -sha256 -key certs/ca.key -days 3650 -subj '/O=PostgreSQL Test/CN=Certificate Authority' -out certs/ca.crt
openssl genrsa -out certs/server.key 2048
chmod og-rwx certs/server.key
chown 999:999 certs/server.key
openssl req -new -sha256 -key certs/server.key -subj '/O=PostgreSQL Test/CN=Server' | openssl x509 -req -sha256 -CA certs/ca.crt -CAkey certs/ca.key -CAserial certs/ca.txt -CAcreateserial -days 365 -out certs/server.crt
```

Run with option: ssl=on:
> docker run --name postgres -e POSTGRES_DB=mydatabase -e POSTGRES_USER=admin -e POSTGRES_PASSWORD=p4ss -p 5432:5432 -v $PWD/certs/server.crt:/var/lib/postgresql/server.crt -v $PWD/certs/server.key:/var/lib/postgresql/server.key -d postgres:13.2 -c __ssl=on__ -c ssl_cert_file=/var/lib/postgresql/__server.crt__ -c ssl_key_file=/var/lib/postgresql/__server.key__


## TEST

> psql "host=193.224.59.150 port=5432 user=admin dbname=mydatabase sslmode=verify-full sslcert=client.crt sslkey=/var/lib/postgresql/client.key sslrootcert=/var/lib/postgresql/ca.crt __sslmode=require__"

>> Password for user admin:
>> 
>> psql (13.2 (Debian 13.2-1.pgdg100+1))
>> 
>> SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)

__Notes__:
- __sslmode=prefer__: default value, encryption if the server supports it
- sslmode=require: I trust that the network will make sure I always connect to the server I want.
- sslmode=verify-ca/sslmode=verify-full: I want to be sure that I connect to a server that I trust.
- see: Table 31-3. SSL mode descriptions https://www.postgresql.org/docs/9.0/libpq-ssl.html

With server certificate verification - but without hotname verification:

> docker exec -it postgres psql "host=IP port=5432 user=admin dbname=mydatabase sslmode=__verify-ca__ sslrootcert=/var/lib/postgresql/__ca.crt__"

# TLS mutual authentication

__NOT POSSIBLE VIA DOCKER; init script does not seem to allow hostssl=cert setting__

The manual solution would be to
add line to /var/lib/postgresql/data/pg_hba.conf:

```text
hostssl all myuser 0.0.0.0/0               md5 clientcert=1
# to use only TLS authentication, remove md5
```
__Note__:
- it is not working: -e POSTGRES_INITDB_ARGS="--auth-hostssl=cert"
- it is not working: -e POSTGRES_INITDB_ARGS="--auth-host=cert"
- it is not working: -e POSTGRES_HOST_AUTH_METHOD=cert
- it is not working: -v $PWD/pg_hba.conf:/var/lib/postgresql/data/pg_hba.conf


(initdb location: /usr/lib/postgresql/13/bin/initdb)

See: 
https://www.postgresql.org/docs/9.5/ssl-tcp.html, https://smallstep.com/hello-mtls/doc/server/postgresql.

