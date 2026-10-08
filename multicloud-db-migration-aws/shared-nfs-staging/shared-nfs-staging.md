# Lab 2: Verify Amazon EFS Staging

## Introduction

Validate Amazon EFS storage from the source database EC2 instance and the target Autonomous AI Database. In the source database, inspect the default `DATA_PUMP_DIR` and the Amazon EFS staging directory `EFS_DP_DIR`. In the target database, validate the `ZDM_EFS_DIR` attachment. The EC2 mount and target database attachment must access the same Amazon EFS filesystem over NFSv4.

Estimated Time: 15 minutes

### Objectives

- Run the Amazon EFS validation script.
- Review the EC2 mount, target database attachment, and shared-file results.
- Connect to the target database from EC2 and verify a file write through SQL*Plus.

## Task 1: Run the Amazon EFS Readiness Check

1. Continue in the same Session Manager session from Lab 1. If disconnected, reconnect to your assigned EC2 instance through **Connect → Session Manager → Connect** in **US West (Oregon)** (`us-west-2`).

2. Switch to the `oracle` shell. If you are already in the `oracle` shell, continue to Step 3.

    ```bash
    <copy>
    sudo -iu oracle
    </copy>
    ```

3. Run the script and save its formatted output.

    ```bash
    <copy>
    set -o pipefail
    umask 077
    bash "$HOME/lab2-efs-validation.sh" 2>&1 | tee "$HOME/lab2-efs-validation.log"
    </copy>
    ```

    The script loads your assigned environment variables and checks the existing EC2 mount and target database attachment. It also creates and removes its own test files. It does not require a password prompt.

4. Review the saved output. Use the arrow keys to scroll and press `q` to return to the shell.

    ```bash
    <copy>
    less -S "$HOME/lab2-efs-validation.log"
    </copy>
    ```

    Continue only when the final result is `LAB2_EFS_VALIDATION: PASS`. Resolve any failed or timed-out checks before starting migration.

## Task 2: Understand the Validation Output

The screenshots below show sample output. Compare them with your own assigned values and results.

1. **Assigned Amazon EFS storage.** Confirm your EFS ID, hostname, IP address, mount path, and target database alias. These identify the storage used for the source database export and target database import.

    ![Assigned EFS filesystem, source mount path, and target alias](./images/efs-validation-section-1.png)

2. **Source database EC2 mount.** Confirm the assigned Amazon EFS filesystem is mounted using NFSv4 and that `oracle` can write and read a test file. This verifies the source database host can create Data Pump export files on Amazon EFS.

    ![Assigned NFSv4 mount and successful source write and read checks](./images/efs-validation-section-2.png)

3. **Source database Data Pump directories.** In the script output, locate the `DIRECTORY_NAME` and `DIRECTORY_PATH` columns. Confirm that `DATA_PUMP_DIR` has a local directory path and that `EFS_DP_DIR` points to `/data/oracle/efs/zdm-lab-gold/datapump`, beneath the Amazon EFS mount `/data/oracle/efs`. These are database directory objects: their paths identify where database file operations read and write files.

4. **Target database login and Amazon EFS attachment.** Confirm the current database user is `ADMIN`, and `ZDM_EFS_DIR` references the assigned Amazon EFS filesystem with NFS version `4`. This verifies the target database login and its staging attachment.

    ![Target ADMIN connection and assigned NFSv4 attachment](./images/efs-validation-section-4.png)

5. **Target database write and source database host read.** Confirm the file written by the target database is read successfully on the source database EC2 instance. This verifies that both access the same Amazon EFS path; the script removes its generated test file afterward.

    ![Target-written file successfully read from EC2](./images/efs-validation-section-5.png)

6. **Validation summary.** Require `5` passed checks, `0` failed checks, and `LAB2_EFS_VALIDATION: PASS` before continuing.

    ![All five Amazon EFS validation checks passed](./images/efs-validation-section-6.png)

## Task 3: Validate a Target Database Write from EC2

1. In the `oracle` shell, load the source database and target database connection environment variables.

    ```bash
    <copy>
    source "$HOME/env/source19c.env"
    source /data/oracle/lab/config/lab-env.sh
    source "$HOME/env/adbs.env"
    </copy>
    ```

2. Connect to the target database. At the password prompt, enter its **Target ADMIN Password** from your reservation information.

    ```bash
    <copy>
    sqlplus -L "ADMIN@$TARGET_ALIAS"
    </copy>
    ```

3. Confirm the target database and connected user.

    ```sql
    <copy>
    SET LINESIZE 160
    SET PAGESIZE 100
    COLUMN db_name FORMAT A40
    COLUMN current_user FORMAT A16
    SELECT SYS_CONTEXT('USERENV','DB_NAME') AS db_name,
           SYS_CONTEXT('USERENV','CURRENT_USER') AS current_user
    FROM dual;
    </copy>
    ```

    Confirm that `DB_NAME` matches your assigned target database and `CURRENT_USER` is `ADMIN`.

    ![Target database identity and ADMIN connected user](./images/target-database-identity.png)

4. Check the target database Amazon EFS attachment.

    ```sql
    <copy>
    COLUMN file_system_name FORMAT A18
    COLUMN file_system_location FORMAT A65
    COLUMN directory_name FORMAT A20
    SELECT file_system_name, file_system_location, directory_name, nfs_version
    FROM dba_cloud_file_systems
    WHERE directory_name = 'ZDM_EFS_DIR';
    </copy>
    ```

    Match `FILE_SYSTEM_LOCATION` to the Amazon EFS hostname in Task 2, Step 4. Confirm `DIRECTORY_NAME` is `ZDM_EFS_DIR` and `NFS_VERSION` is `4`.

    ![Target query shows the assigned EFS attachment and NFS version 4](./images/target-efs-attachment.png)

5. Write a validation file from the target database. This replaces only `participant_validation.txt` if that test file already exists.

    ```sql
    <copy>
    DECLARE
      f UTL_FILE.FILE_TYPE;
    BEGIN
      f := UTL_FILE.FOPEN('ZDM_EFS_DIR', 'participant_validation.txt', 'w');
      UTL_FILE.PUT_LINE(f, 'EFS attachment validated from ADB-S');
      UTL_FILE.FCLOSE(f);
    EXCEPTION WHEN OTHERS THEN
      IF UTL_FILE.IS_OPEN(f) THEN UTL_FILE.FCLOSE(f); END IF;
      RAISE;
    END;
    /
    </copy>
    ```

    Require `PL/SQL procedure successfully completed`.

    ![Target SQL write creates participant_validation.txt successfully](./images/efs-target-write-proof.png)

6. Confirm the file is visible from the target database.

    ```sql
    <copy>
    SELECT object_name, bytes
    FROM DBMS_CLOUD.LIST_FILES('ZDM_EFS_DIR')
    WHERE object_name = 'participant_validation.txt';
    </copy>
    ```

    Expect one row for `participant_validation.txt` with a nonzero byte count.

    ![Target file-list query returns participant_validation.txt with 36 bytes](./images/target-validation-file-list.png)

7. Exit SQL*Plus to return to the `oracle` shell prompt.

    ```sql
    <copy>
    EXIT
    </copy>
    ```

8. Read the target database's file from the Amazon EFS mount on the source database EC2 instance.

    ```bash
    <copy>
    cat "$EFS_MOUNT_POINT/participant_validation.txt"
    </copy>
    ```

    Expect `EFS attachment validated from ADB-S`. This confirms a target database write is visible on EC2 through the shared EFS filesystem. Leave this validation file in place. Stop if the write or read fails or hangs.

    ![EC2 reads the validation file written by the target database](./images/efs-source-read-proof.png)

## Acknowledgements

* **Author** - Arnab Saha, Principal Solutions Architect, OCI Multicloud
* **Author** - Vineet Agarwal, Senior Principal Solutions Architect, OCI Multicloud
* **Last Updated By/Date** - Arnab Saha and Vineet Agarwal / October 8, 2026
