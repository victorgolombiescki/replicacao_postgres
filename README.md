# replicacao_postgres
Neste guia, vamos configurar um ambiente de replicação do PostgreSQL utilizando Docker Compose, garantindo que a réplica se mantenha sincronizada com o banco de dados principal.

📌 Tópicos que serão abordados:

1. Introdução à Replicação no PostgreSQL – O que é replicação e seus benefícios.

2. Configuração do Banco de Dados Principal (main) – Docker Compose, ajustes no PostgreSQL e criação do usuário de replicação.

3. Configuração da Réplica (replica_1) – Backup inicial e ativação do modo standby.

4. Testando a Replicação – Verificando sincronização e replicação de dados.

5. Conclusão e Próximos Passos – Recapitulando os passos e próximos desafios.



---

📌 1. Introdução à Replicação no PostgreSQL – O que é replicação e seus benefícios

image.png



A replicação no PostgreSQL é o processo de manter cópias sincronizadas do banco de dados em múltiplos servidores. Isso garante alta disponibilidade, melhor desempenho e recuperação rápida em caso de falhas.

🔹 Benefícios da replicação:

1. Alta disponibilidade (HA) – Se o servidor principal falhar, a réplica assume, garantindo continuidade.
2. Balanceamento de carga – Consultas de leitura podem ser distribuídas para réplicas, aliviando a carga do servidor principal.
3. Recuperação de desastres – Protege contra falhas, evitando perda de dados.
4. Backups eficientes – Permite fazer backups a partir da réplica sem impactar o desempenho do banco principal.
5. Melhoria no desempenho – Reduz a latência ao permitir consultas em servidores próximos aos usuários.

---

📌 2. Configurar e Subir o Banco de Dados Principal (main)

🔹 2.1 Criar o arquivo docker-compose.yml

Crie um diretório para o projeto e dentro dele, um arquivo docker-compose.yml com o seguinte conteúdo:

services:
  main:
    image: postgres:14
    container_name: postgres-main
    ports:
      - "5438:5432"
    environment:
      - POSTGRES_PASSWORD=postgres
      - PGDATA=/var/lib/postgresql/data
    volumes:
      - ./master:/var/lib/postgresql/data
      - type: tmpfs 
        target: /dev/shm

  replica_1:
    image: postgres:14
    container_name: postgres-replica
    ports:
      - "5439:5432"
    environment:
      - POSTGRES_PASSWORD=postgres
      - PGDATA=/tmp
    volumes:
      - ./replication:/tmp
      - type: tmpfs 
        target: /dev/shm
    depends_on:
      - main


💡 Explicação:

* O serviço main será o banco de dados principal.
* O serviço replica_1 será a réplica, apontando inicialmente para /tmp como diretório de dados.
* O depends_on garante que a réplica só suba depois do main.

---

🔹 1.2 Subir o main

Agora, suba apenas o banco principal:

docker-compose up -d main

Aguarde alguns segundos para o PostgreSQL iniciar.

---

📌 3. Configurar postgresql.conf e pg_hba.conf no main

Agora, precisamos ajustar a configuração para permitir conexões da réplica.

🔹 3.1 Ajustar postgresql.conf sudo chmod -R 755 ./master

Acessar pasta master arquivo postgresql.conf  e altere os seguintes parametros:

# WRITE-AHEAD LOG
wal_level = logical
wal_compression = on 

#------------------------------------------------------------------------------
# REPLICATION
#------------------------------------------------------------------------------
max_wal_senders = 10 
max_replication_slots = 10
primary_conninfo = 'host=main port=5432 user=userbackup password=123456'
primary_slot_name = 'slot_replicacao_master'   
hot_standby = on 
hot_standby_feedback = true 

💡 Explicação:

* wal_level = logical → Habilita replicação lógica.
* wal_compression = on → Ativa compressão dos logs de transação.
* max_wal_senders = 10 → Define até 10 conexões de replicação simultâneas.
* max_replication_slots = 10 → Limita a quantidade de slots de replicação.
* primary_conninfo → Define a conexão da réplica com o servidor principal.
* primary_slot_name → Nome do slot de replicação.
* hot_standby = on → Permite consultas de leitura na réplica.
* hot_standby_feedback = true → Evita exclusão de dados antes que a réplica os processe.

---

🔹 3.2 Ajustar pg_hba.conf

Acessar pasta master arquivo pg_hba.conf sudo chmod -R 755 ./master  e adicionar os seguintes parametros:

# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    replication     userbackup      0.0.0.0/0               scram-sha-256

---

🔹 3.3 Criar usuário de replicação

Acessar dbeaver ou qualquer SGBD e criar o usuario:

CREATE ROLE userbackup WITH 
    SUPERUSER
    CREATEDB
    CREATEROLE
    INHERIT
    LOGIN
    REPLICATION
    BYPASSRLS
    CONNECTION LIMIT -1
    PASSWORD '123456';

---

🔹 3.4 Criação do Slot de replicação

SELECT * FROM pg_create_physical_replication_slot('slot_replicacao_master');

🔹 3.5 Reiniciar o main

Saia do contêiner (exit) e reinicie o banco:

docker restart postgres-main


---

📌 4. Subir a Réplica

Agora, subimos a réplica no Compose:

docker-compose up -d replica_1

Verifique se os contêineres estão rodando:

docker ps

sudo chmod -R 755 ./replication

A réplica ainda não está sincronizada, então precisamos fazer o pg_basebackup.

---

📌 5. Criar Backup e Iniciar a Réplica

Agora, entre no contêiner da réplica:

docker exec -it postgres-replica bash


🔹 5.1 Executar pg_basebackup

Dentro do contêiner da réplica, execute:

pg_basebackup -h main -U userbackup -D /var/lib/postgresql/data -v -P -X stream -c fast


💡 Explicação:

* -h main → Conecta ao servidor main via Docker network.
* -U userbackup → Usa o usuário de replicação.
* -D /var/lib/postgresql/data → Baixa os arquivos na pasta de dados.
* -X stream → Copia os logs de transação em tempo real.
* -c fast → Faz o backup com menos impacto no servidor.

⚠️ Se pedir senha, digite 123456.

---

🔹 5.2 Alterar diretório PGDATA replicação 

Esse arquivo indica que a instância é uma réplica:

PGDATA=/var/lib/postgresql/data

---



---

🔹 5.3 Criar o Arquivo standby.signal

Esse arquivo indica que a instância é uma réplica:

touch /var/lib/postgresql/data/standby.signal


---

🔹 5.4 Reiniciar a réplica

Saia do contêiner (exit) e reinicie a réplica:

docker restart postgres-replica

---

📌 6. Verificar se a Replicação Está Funcionando

🔹 6.1 Testar a Conexão na Réplica

Acesse a réplica e veja se está rodando em standby mode:

docker exec -it postgres-replica psql -U postgres -c "SELECT pg_is_in_recovery();"

Se o resultado for t (true), significa que a réplica está sincronizando corretamente!

---

🔹 6.2 Testar a Replicação

Para testar se os dados estão sendo replicados, crie uma tabela no main:

docker exec -it postgres-main psql -U postgres -c "CREATE TABLE teste (id SERIAL PRIMARY KEY, nome TEXT);"


Agora, verifique se a tabela apareceu na réplica:

docker exec -it postgres-replica psql -U postgres -c "\dt"

Se a tabela teste aparecer na réplica, significa que a replicação está funcionando corretamente! 🎉

---

🎯 Resumo

1. Criamos o docker-compose.yml com main e replica_1.
2. Subimos o main e configuramos:
  * postgresql.conf para habilitar replicação.
  * pg_hba.conf para permitir conexões remotas.
  * Criamos o usuário de replicação (userbackup).
3. Subimos a réplica (replica_1).
4. Rodamos pg_basebackup para copiar os dados do main.
5. Criamos standby.signal e configuramos primary_conninfo na réplica.
6. Verificamos se a replicação está funcionando (pg_is_in_recovery() e \dt).

