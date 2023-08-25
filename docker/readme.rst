==========
PostgreSQL
==========

About
=====
**PostgreSQL** [1]_ is an open source object-relational database system.

Version
-------
PostgreSQL version **15.x** deployed based on the official Docker Hub image: [2]_.

License
-------
**PostgreSQL License** [3]_

Pre-requisites
==============
* *docker* installed
* access to DIGITbrain private docker repo (username, password) to pull the image:

  - ``docker login dbs-container-repo.emgora.eu``
  - ``docker pull dbs-container-repo.emgora.eu/postgres:15``

Usage
=====
.. code-block:: bash

  docker run -d --rm \
      --name postgres \
      -e POSTGRES_DB=mydatabase \
      -e POSTGRES_USER=mydatabaseuser \
      -e POSTGRES_PASSWORD=mydatabasepassword \
      -p 5432:5432 \
      postgres:15 \
      -c ssl=on \
      -c ssl_cert_file=/etc/certs/server-cert.pem \
      -c ssl_key_file=/etc/certs/server-key.pem \
      -c ssl_ca_file=/etc/certs/ca.pem

where POSTGRES_DB parameter is the name of an initial database to be created,
POSTGRES_USER and POSTGRES_PASSWORD parameters create a new database user with the given username and password,
standard PostgreSQL port 5432 is opened on the host, and SSL turned on.

.. warning::
  Always update the ``POSTGRES_USER`` and ``POSTGRES_PASSWORD`` parameters with the values of your choice
  prior to running this container.

Security
========
The image uses **SSL/TLS traffic encryption** and **username-password authentication**, by
default using a DIGITbrain server certificate signed by DIGITbrain CA. You can override these certificates with your own,
see *volumes* parameters below.

In clients, choose ``sslmode=require``, ``sslmode=verify-ca``, or preferably ``sslmode=verify-full`` option
(the latter includes hostname verification that will not work with the pre-built certificates) [4]_, for example:

.. code-block:: python

  import psycopg2
  conn = psycopg2.connect(host="xxx.xxx.xxx.xxx", port=5432, database="mydatabase", user="mydatabaseuser", password="mydatabasepassword", \
                          sslmode='verify-ca', sslrootcert='ca.pem')
  cur = conn.cursor()
  cur.execute("""SELECT * FROM mytable""")
  query_results = cur.fetchall()
  cur.close()
  conn.close()

Note that without ``sslmode`` option the connection will succeed without any traffic encryption.
Also note that connection from localhost (PostgreSQL host) is allowed without even authentication.
For forcing SSL connection or setting up TLS client authentication refer to [5]_ (requires altering file: ``/var/lib/postgresql/data/pg_hba.conf``).

Configuration
=============

Environment variables
---------------------
.. list-table::
   :header-rows: 1

   * - Name
     - Example
     - Comment
   * - *database*
     - ``-e POSTGRES_DB=mydatabase``
     - An initial database to create
   * - *username*
     - ``-e POSTGRES_USER=mydatabaseuser``
     - Username for a newly created user
   * - *password*
     - ``-e POSTGRES_PASSWORD=mydatabasepassword``
     - Password for a newly created user

Ports
-----
.. list-table::
  :header-rows: 1

  * - Container port
    - Host port bind example
    - Comment
  * - *5432*
    - ``-p 15432:5432``
    - Default PostgreSQL container port 5432 is opened as port 15432 on the host

Volumes
-------
.. list-table::
  :header-rows: 1

  * - Name
    - Volume mount example
    - Comment
  * - *PostgreSQL data*
    - ``-v $PWD/data:/var/lib/postgresql/data/``
    - PostgreSQL data will be persisted in host directory: ``./data``.
  * - *CA certificate*
    - ``-v $PWD/certificates/ca.pem:/etc/certs/ca.pem``
    - Overrides Certificate Authority (CA) certificate
  * - *Server key*
    - ``-v $PWD/certificates/server-key.pem:/etc/certs/server-key.pem``
    - Overrides server key (path defined by $PGSSLKEY)
  * - *Server certificate*
    - ``-v $PWD/certificates/server-cert.pem:/etc/certs/server-cert.pem``
    - Overrides server certificate (path defined by $PGSSLCERT)

References
==========
.. [1] https://www.postgresql.org/

.. [2] https://hub.docker.com/_/postgres

.. [3] https://www.postgresql.org/about/licence/

.. [4] https://www.postgresql.org/docs/14/libpq-ssl.html#LIBPQ-SSL-PROTECTION

.. [5] https://smallstep.com/hello-mtls/doc/server/postgresql
