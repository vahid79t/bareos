
Follow these steps:

1. **Check PostgreSQL Database and User:**
    ```bash
    sudo -u postgres psql -c "\l"
    sudo -u postgres psql -c "\du"
    ```

2. **Grant Superuser Privileges and Set Password:**
    ```bash
    sudo -u postgres psql
    ALTER USER bareos WITH SUPERUSER;
    ALTER USER bareos WITH PASSWORD 'secret';
    \q
    ```

3. **Update `pg_hba.conf`:**
    ```bash
    nano /var/lib/pgsql/data/pg_hba.conf
    ```
    Add or update the following lines:
    ```
    # TYPE  DATABASE        USER            ADDRESS                 METHOD
    local   all             all                                     md5
    host    all             all             127.0.0.1/32            md5
    host    all             all             ::1/128                 md5
    local   replication     all                                     peer
    host    replication     all             127.0.0.1/32            ident
    host    replication     all             ::1/128                 ident
    ```

4. **Restart PostgreSQL and Bareos Director:**
    ```bash
    systemctl restart postgresql
    systemctl restart bareos-dir
    ```

5. **Test Bareos Connectivity:**
    ```bash
    psql -h 127.0.0.1 -U bareos -d bareos
    sudo -u bareos psql -W -d bareos -U bareos -h localhost
    ```

6. **Update PostgreSQL Configuration:**
    ```bash
    sudo nano /var/lib/pgsql/data/postgresql.conf
    ```
    Adjust `max_connections` as needed:
    ```ini
    max_connections = 100
    ```

7. **Update Bareos Configuration:**
    ```ini
    Catalog {
      Name = MyCatalog
      dbname = "bareos"
      dbuser = "bareos"
      dbpassword = "secret"
      dbaddress = "127.0.0.1"  # Use TCP/IP connection
    }
    ```

8. **Restart PostgreSQL and Bareos Director:**
    ```bash
    sudo systemctl restart postgresql
    systemctl restart bareos-dir
    bareos-dir -t
    ```

## WebUI Installation

1. **Install Bareos WebUI:**
    ```bash
    yum install bareos-webui
    ```

2. **Restart Apache and Bareos Director:**
    ```bash
    systemctl restart httpd
    systemctl restart bareos-dir
    ```

3. **Create Admin User for WebUI:**
    ```bash
    bconsole
    configure add console name=admin password=secret profile=webui-admin
    ```

4. **Verify Console Configuration:**
    ```bash
    cat /etc/bareos/bareos-dir.d/console/admin.conf
    ```
    Ensure it contains:
    ```ini
    Console {
      Name = "admin"
      Password = "secret"
      Profile = webui-admin
      TLS Enable = no
    }
    ```

5. **Access WebUI:**
    Open a browser and navigate to `http://your-server-ip/bareos-webui/`.

## Client Installation

1. **Add Bareos Repositories:**
    ```bash
    wget https://download.bareos.org/current/EL_9/add_bareos_repositories.sh
    chmod +x add_bareos_repositories.sh
    sh ./add_bareos_repositories.sh
    ```

2. **Install Required Packages:**
    ```bash
    yum install epel-release
    yum update
    yum install httpd php php-cli php-common -y
    sudo yum install libstdc++.so.6 libc.so.6
    sudo apt-get install bareos-client
    ```

3. **Start and Enable Bareos File Daemon:**
    ```bash
    sudo systemctl start bareos-fd
    sudo systemctl enable bareos-fd
    ```

## Configuration for Backup

### Server Configuration

1. **Catalog Configuration:**
    ```bash
    cd /etc/bareos/bareos-dir.d/catalog/
    cat MyCatalog.conf
    ```
    Ensure it contains:
    ```ini
    Catalog {
      Name = MyCatalog
      dbname = "bareos"
      dbuser = "bareos"
      dbpassword = "secret"
      dbaddress = "127.0.0.1"  # Use TCP/IP connection
    }
    ```

2. **Client Configuration:**
    ```bash
    cd /etc/bareos/bareos-dir.d/client/
    cat bareos83.conf
    ```
    Ensure it contains:
    ```ini
    Client {
      Name = bareos83
      Address = 91.107.151.83
      FDPort = 9102
      Password = "123456vahid"
      Catalog = MyCatalog
      # TLS Enable = no
      # TLS Require = no
    }
    ```

3. **Console Configuration:**
    ```bash
    cat /etc/bareos/bareos-dir.d/console/admin.conf
    ```
    Ensure it contains:
    ```ini
    Console {
      Name = "admin"
      Password = "secret"
      Profile = webui-admin
      TLS Enable = no
    }
    ```

4. **FileSet Configuration:**
    ```bash
    cd /etc/bareos/bareos-dir.d/fileset/
    cat var-set.conf
    ```
    Ensure it contains:
    ```ini
    FileSet {
      Name = "Var Set"
      Include {
        Options {
          signature = MD5
        }
        File = /var
      }
    }
    ```

5. **Job Configuration:**
    ```bash
    cd /etc/bareos/bareos-dir.d/job/
    cat VarBackupJobbareos83.conf
    ```
    Ensure it contains:
    ```ini
    Job {
      Name = "VarBackupJob"
      Type = Backup
      Client = bareos83
      FileSet = "Var Set"
      Schedule = "WeeklyCycle"
      Storage = File
      Messages = Standard
      Pool = Incremental
      Priority = 10
      Write Bootstrap = "/var/lib/bareos/%c.bsr"
      Full Backup Pool = Full
      Differential Backup Pool = Differential
      Incremental Backup Pool = Incremental
    }
    ```

6. **JobDefs Configuration:**
    ```bash
    cd /etc/bareos/bareos-dir.d/jobdefs/
    cat MyBackupbareos83Job.conf
    ```
    Ensure it contains:
    ```ini
    JobDefs {
      Name = "MyBackupJob"
      Type = Backup
      Level = Incremental
      Client = bareos83                # Replace with the appropriate client name
      FileSet = "Var Set"             # Use the appropriate fileset name for your backups
      Schedule = "WeeklyCycle"        # Use the appropriate schedule name
      Storage = File                  # Replace with the appropriate storage name
      Messages = Standard
      Pool = Incremental              # Replace with the appropriate pool name
      Priority = 10
      Write Bootstrap = "/var/lib/bareos/%c.bsr"
      Full Backup Pool = Full         # Replace with the appropriate pool name for full backups
      Differential Backup Pool = Differential
      Incremental Backup Pool = Incremental
    }
    ```

7. **Schedule Configuration:**
    ```bash
    cd /etc/bareos/bareos-dir.d/schedule/
    cat WeeklyCycle.conf
    ```
    Ensure it contains:
    ```ini
    Schedule {
      Name = "WeeklyCycle"
      Run = Full 1st sat at 21:00
      Run = Differential 2nd-5th sat at 21:00
      Run = Incremental mon-fri at 21:00
    }
    ```

### Storage Configuration

1. **Create and Configure NFS Share:**
    ```bash
    mkdir -p /mnt/nfs_share/storage
    chown bareos:bareos /mnt/nfs_share/storage
    ```

2. **Mount NFS Share and Update `fstab`:**
    ```bash
    nano /etc/fstab
    ```
    Add the following line to ensure NFS share mounts at boot:
    ```ini
    your-nfs-server:/nfs_share /mnt/nfs_share nfs defaults 0 0
    ```

3. **Configure Storage Device:**
    ```bash
    cd /etc/bareos/bareos-sd.d/device/
    cat FileStorage.conf
    ```
    Ensure it contains:
    ```ini
    Device {
      Name = FileStorage
      Media Type = File
      Archive Device = /mnt/nfs_share/storage
      LabelMedia = yes;                   # lets Bareos label unlabeled media
      Random Access = Yes;
      AutomaticMount = yes;               # when device opened, read it
      RemovableMedia = no;
      AlwaysOpen = no;
    }
    ```

## Client Configuration

1. **Configure Client:**
    ```bash
    cd /etc/bareos/bareos-fd.d/client/
    cat myself.conf
    ```
    Ensure it contains:
    ```ini
    Client {
      Name = rocky-2gb-nbg1-1-fd
      Maximum Concurrent Jobs = 20
      FDport = 9102
      WorkingDirectory = /var/lib/bareos
      # remove comment from "Plugin Directory" to load plugins from specified directory.
      # if "Plugin Names" is defined, only the specified plugins will be loaded,
      # otherwise all filedaemon plugins (*-fd.so) from the "Plugin Directory".
      #
      # Plugin Directory = "/usr/lib64/bareos/plugins"
      # Plugin Names = ""
    }
    ```

2. **Configure Director:**
    ```bash
    cd /etc/bareos/bareos-fd.d/director/
    cat bareos83.conf
    ```
    Ensure it contains:
    ```ini
    Director {
      Name = bareos-dir
      Password = "123456vahid"
      Description = "Allow the configured Director to access this file daemon."
      # TLS Enable = no
      # TLS Require = no
    }
    ```

## Final Steps

1. **Restart All Services:**
    ```bash
    systemctl restart postgresql
    systemctl restart bareos-dir
    systemctl restart bareos-sd
    systemctl restart bareos-fd
    systemctl restart httpd
    ```

2. **Test Configuration:**
    ```bash
    bareos-dir -t
    ```

This completes the Bareos installation and configuration. You can now manage backups using the Bareos WebUI and ensure the server and client configurations are correctly set for your backup needs.
