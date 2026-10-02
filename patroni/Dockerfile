FROM postgres:16-bookworm

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
       python3 \
       python3-pip \
       python3-venv \
       curl \
       pgbackrest \
    && python3 -m venv /opt/patroni \
    && /opt/patroni/bin/pip install --no-cache-dir patroni[etcd3] psycopg[binary] \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

ENV PATH="/opt/patroni/bin:$PATH"

RUN mkdir -p /etc/patroni /var/lib/pgbackrest \
    && chown -R postgres:postgres /etc/patroni /var/lib/pgbackrest

USER postgres

CMD ["patroni", "/etc/patroni/patroni.yml"]
