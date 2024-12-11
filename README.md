# Proposal for Table Synchronization Between Two PostgreSQL Databases

## Objective
The goal is to establish a reliable and efficient mechanism to synchronize tables between two PostgreSQL databases for production use. This document outlines the available methods, their feasibility for production environments, and the recommended approach.

---

## Challenges in Production Synchronization
- **Latency:** Synchronization must be timely to avoid data inconsistencies.
- **Reliability:** The solution must handle failures and ensure data integrity.
- **Scalability:** The synchronization mechanism should scale with database size and activity.
- **Maintenance:** The setup must require minimal manual intervention.

---

## Proposed Solutions
### 1. Logical Replication
**Overview:** Built-in PostgreSQL feature that replicates data changes from a source (publisher) to a target (subscriber).
- **Key Features:**
  - Asynchronous replication for low-latency updates.
  - Supports selective table replication.
  - Reliable for production environments.
- **Steps to Implement:**
  1. Enable `wal_level` as `logical` in `postgresql.conf`:
     ```
     wal_level = logical
     max_replication_slots = 4
     max_wal_senders = 4
     ```
     Restart PostgreSQL service.
  2. Create a publication on the source database:
     ```sql
     CREATE PUBLICATION my_publication FOR TABLE my_table;
     ```
  3. Set up the subscription on the target database:
     ```sql
     CREATE SUBSCRIPTION my_subscription CONNECTION 'host=source_host dbname=source_db user=replicator password=your_password' PUBLICATION my_publication;
     ```
- **Pros:** Native feature, robust, suitable for real-time sync.
- **Cons:** Does not replicate DDL changes.

### 2. Foreign Data Wrapper (FDW)
**Overview:** Enables direct access to remote tables, allowing query and updates without duplication.
- **Key Features:**
  - Access remote tables as local.
  - Supports both reading and writing.
- **Steps to Implement:**
  1. Install FDW extension:
     ```sql
     CREATE EXTENSION postgres_fdw;
     ```
  2. Connect to the remote database:
     ```sql
     CREATE SERVER foreign_server FOREIGN DATA WRAPPER postgres_fdw OPTIONS (host 'remote_host', dbname 'remote_db');
     ```
  3. Map users and tables:
     ```sql
     CREATE USER MAPPING FOR local_user SERVER foreign_server OPTIONS (user 'remote_user', password 'remote_password');
     CREATE FOREIGN TABLE my_table (id INT, name TEXT) SERVER foreign_server OPTIONS (schema_name 'public', table_name 'my_table');
     ```
- **Pros:** Reflects changes directly, minimal setup.
- **Cons:** Network-dependent, not ideal for high-frequency updates.

### 3. Triggers + dblink
**Overview:** Customizable solution using database triggers and dblink functions for replicating data.
- **Steps to Implement:**
  1. Install `dblink` extension:
     ```sql
     CREATE EXTENSION dblink;
     ```
  2. Create a trigger function:
     ```sql
     CREATE OR REPLACE FUNCTION sync_data() RETURNS TRIGGER AS $$
     BEGIN
         PERFORM dblink_exec(
             'host=remote_host dbname=remote_db user=remote_user password=remote_password',
             'INSERT INTO my_table (id, name) VALUES (' || NEW.id || ', ' || quote_literal(NEW.name) || ')'
         );
         RETURN NEW;
     END;
     $$ LANGUAGE plpgsql;
     ```
  3. Attach the trigger to the source table:
     ```sql
     CREATE TRIGGER after_insert_trigger AFTER INSERT ON my_table FOR EACH ROW EXECUTE FUNCTION sync_data();
     ```
- **Pros:** Customizable, supports specific synchronization logic.
- **Cons:** Performance overhead for high transaction volumes.

### 4. Third-Party Tools
- **pglogical:** Enhances logical replication with advanced features.
- **Bucardo:** Multi-master replication for complex setups.
- **SymmetricDS:** Supports cross-database synchronization.
- **Slony-I:** Master-slave replication for simple setups.
- **Pros:** Advanced features for complex scenarios.
- **Cons:** Additional installation and potential licensing costs.

---

## AWS DMS for PostgreSQL Synchronization
**Overview:** A managed service for data migration and replication.
- **Key Features:**
  - Continuous Data Replication (CDC).
  - Supports cross-region/account replication.
  - Reduces administrative overhead.
- **Steps to Implement:**
  1. Configure source and target endpoints.
  2. Create a replication task.
  3. Start and monitor the replication.
- **Pros:** Fully managed, supports schema conversion.
- **Cons:** Costs are incurred for instance usage and data transfer.

### Cost Analysis
- **Instance Costs:** `dms.t3.medium` (~$35/month).
- **Storage:** $5/month for 50GB.
- **Data Transfer:** Minimal for intra-region.

---

## Recommendation
1. **Primary Recommendation:** Logical Replication for its native integration and reliability.
2. **Alternate Recommendations:**
   - Use FDW for direct access.
   - Consider AWS DMS for managed services or complex environments.

---

## Implementation Plan
1. Stakeholder Approval: Review and approve the proposed solution.
2. Proof of Concept (POC): Test Logical Replication in a non-production environment.
3. Deployment: Configure and deploy the chosen solution in production.
4. Monitoring: Set up monitoring tools to track replication delays and failures.

---

