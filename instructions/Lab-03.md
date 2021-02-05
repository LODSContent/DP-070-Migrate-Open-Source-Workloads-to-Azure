# Lab: Migrate an on-premises PostgreSQL database to Azure

## Overview

In this lab, you'll use the information learned in this module to migrate a PostgreSQL database to Azure. To give complete coverage, students will perform two migrations. The first is an offline migration, transferring an on-premises PostgreSQL database to a virtual machine running on Azure. The second is an online migration of the database running on the virtual machine to Azure Database for PostgreSQL.

You'll also reconfigure and run a sample application that uses the database, to verify that the database operates correctly after each migration.

## Objectives

After completing this lab, you will be able to:

1. Perform an offline migration of an on-premises PostgreSQL database to an Azure virtual machine.
1. Perform an online migration of a PostgreSQL database running on a virtual machine to Azure Database for PostgreSQL.

## Scenario

You work as a database developer for the AdventureWorks organization. AdventureWorks has been selling bicycles and bicycle parts directly to end-consumer and distributors for over a decade. Their systems store information in a database that currently runs using PostgreSQL, located in their on-premises datacenter. As part of a hardware rationalization exercise, AdventureWorks want to move the database to Azure. You have been asked to perform this migration.

Initially, you decide to relocate the data quickly to a PostgreSQL database running on an Azure virtual machine. This is considered to be a low-risk approach as it requires few if any changes to the database. However, this approach does require that you continue to perform most of the day-to-day monitoring and administrative tasks associated with the database. You also need to consider how the customer base of AdventureWorks has changed. Initially AdventureWorks targeted customers in their local region, but now they expanded to be a world-wide operation. Customers can be located anywhere, and ensuring that customers querying the database are subject to minimal latency is a primary concern. You can implement PostgreSQL replication to virtual machines located in other regions, but again this is an administrative overhead.

Instead, once you have got the system running on a virtual machine, you will then consider a more long-term solution; you will migrate the database to Azure Database for PostgreSQL. This PaaS strategy removes much of the work associated with maintaining the system. You can easily scale the system, and add read-only replicas to support customers anywhere in the world. Additionally Microsoft provide a guaranteed SLA for availability.

## Setup

You have an on-premises environment with an existing PostgreSQL database containing the data that you wish to migrate to Azure. Before you start the lab, you need to create an Azure virtual machine running PostgreSQL that will act as the target for the initial migration. We have provided a script that creates this virtual machine and configures it. Use the following steps to download and run this script:

1. Sign in to the **LON-DEV-01** virtual machine running in the classroom environment. The username is **azureuser**, and the password is **Pa55w.rd**.
1. Using a browser, sign in to the Azure portal.
1. Open an Azure Cloud Shell window. Make sure that you are running the **Bash** shell.
1. Clone the repository holding the scripts and sample databases if you haven't done this previously.

    ```bash
    git clone https://github.com/MicrosoftLearning/DP-070-Migrate-Open-Source-Workloads-to-Azure ~/workshop
    ```

1. Move to the *workshop/migration_samples/setup* folder.

    ```bash
    cd ~/workshop/migration_samples/setup
    ```

1. Run the *create_postgresql_vm.sh* script as follows. Specify the name of a resource group and a location for holding the virtual machine as parameters. The resource group will be created if it doesn't already exist. Specify a location near to you, such as *eastus* or *uksouth*:

    ```bash
    bash create_postgresql_vm.sh [resource group name] [location]
    ```

    The script will take approximately 10 minutes to run. It will generate plenty of output as it runs, finishing with the IP address of the new virtual machine, and the message **Setup Complete**.

1. Make a note of the IP address.

> [!NOTE]
> You will need this IP address for the exercise.

## Exercise 1: Migrate the on-premises database to an Azure virtual machine

In this exercise, you'll perform the following tasks:

1. Review the on-premises database.
1. Run a sample application that queries the database.
1. Perform an offline migration of the database to the Azure virtual machine.
1. Verify the database on the Azure virtual machine.
1. Reconfigure and test the sample application against the database in the Azure virtual machine.

### Task 1: Review the on-premises database

1. Sign in to the **LON-DEV-01** virtual machine running in the classroom environment. The username is **azureuser**, and the password is **Pa55w.rd**.

    This virtual machine simulates your on-premises environment. It is running a PostgreSQL server that is hosting the AdventureWorks database that you need to migrate.

1. On the **LON-DEV-01** virtual machine running in the classroom environment, in the **Favorites** bar on the left-hand side of the screen, select the **pgAdmin4** utility.
1. In the **Unlock Saved Password** dialog box, enter the password **Pa55w.rd**, and then select **OK**.
1. In the **pgAdmin4** window, expand **Servers**, expand **LON-DEV-01**, expand **Databases**, expand **adventureworks**, and then expand **Schemas**.
1. In the **sales** schema, expand **Tables**.
1. Right-click the **salesorderheader** table, select **Scripts**, and then select **SELECT Script**.
1. Press **F5** to run the query. It should return 31465 rows.
1. In the list of tables, right-click the **salesorderdetail** table, select **Scripts**, and then select **Select Script**.
1. Press **F5** to run the query. It should return 121317 rows.
1. Spend a couple of minutes browsing the data for the other tables in the various schemas in the database.

### Task 2: Run a sample application that queries the database

1. On the **LON-DEV-01** virtual machine, on the favorites bar, select **Show Applications** and then type **term**.
1. Select **Terminal** to open a terminal window.
1. In the terminal window, download the sample code for the demonstration. If prompted, enter **Pa55w.rd** for the password:

   ```bash
   sudo rm -rf ~/workshop
   git clone https://github.com/MicrosoftLearning/DP-070-Migrate-Open-Source-Workloads-to-Azure ~/workshop
   ```

1. Move to the *~/workshop/migration_samples/code/postgresql/AdventureWorksQueries* folder:

   ```bash
   cd ~/workshop/migration_samples/code/postgresql/AdventureWorksQueries
   ```

    This folder contains a sample app that runs queries to count the number of rows in several tables in the *adventureworks* database.

1. Run the app:

    ```bash
    dotnet run
    ```

    The app should generate the following output:

    ```text
    Querying AdventureWorks database
    SELECT COUNT(*) FROM production.vproductanddescription
    1764

    SELECT COUNT(*) FROM purchasing.vendor
    104

    SELECT COUNT(*) FROM sales.specialoffer
    16

    SELECT COUNT(*) FROM sales.salesorderheader
    31465

    SELECT COUNT(*) FROM sales.salesorderdetail
    121317

    SELECT COUNT(*) FROM person.person
    19972
    ```

### Task 3: Perform an offline migration of the database to the Azure virtual machine

Now that you have an idea of the data in the adventureworks database, you can migrate it to the PostgreSQL server running on the virtual machine in Azure. You'll perform this operation as an offline task, using backup and restore commands.

> [!NOTE]
> If you wanted to migrate the data online, you could configure replication from the on-premises database to the database running on the Azure virtual machine.

1. From the terminal window, run the following command to take a backup of the *adventureworks* database:

    ```bash
    pg_dump adventureworks -U azureuser -Fc > adventureworks_backup.bak
    ```

1. Create the target database on the Azure virtual machine. Replace [nn.nn.nn.nn] with the IP address of the virtual machine that was created during the setup stage of this lab. Enter the password **Pa55w.rd** when prompted for the password for **azureuser**:

    ```bash
    createdb -h [nn.nn.nn.nn] -U azureuser --password adventureworks
    ```

1. Restore the backup into the new database with the pg_restore command:

    ```bash
    pg_restore -d adventureworks -h [nn.nn.nn.nn] -Fc -U azureuser --password adventureworks_backup.bak
    ```

    This command will take a couple of minutes to run.

### Task 4: Verify the database on the Azure virtual machine

1. Run the following command to connect to the database on the Azure virtual machine. The password for the *azureuser* user in the PostgreSQL server running on the virtual machine is **Pa55w.rd**:

    ```bash
    psql -h [nn.nn.nn.nn] -U azureuser adventureworks
    ```

1. Run the following query:

    ```SQL
    SELECT COUNT(*) FROM sales.salesorderheader;
    ```

    Verify that this query returns 31465 rows. This is the same number of rows that is in the on-premises database.

1. Query the number of rows in the *sales.salesorderdetail* table.

    ```SQL
    SELECT COUNT(*) FROM sales.salesorderdetail;
    ```

    This table should contain 121317 rows.

1. Close the *psql* utility with the **\q** command.
1. Switch to the **pgAdmin4** tool.
1. In the left-pane, right-click **Servers**, select **Create**, and then select **Server**.
1. In the **Create - Server** dialog box, on the **General** tab, in the **Name** box, enter **Virtual Machine**, and then select the **Connection** tab.
1. Enter the following details:

    | Property  | Value  |
    | --- | --- |
    | Host name/address | *[nn.nn.nn.nn]* |
    | Port | 5432 |
    | Maintenance database | postgres |
    | Username | azureuser |
    | Password | Pa55w.rd |
    | Save password | Selected |
    | Role | *leave blank* |
    | Service | *Leave blank* |

1. Select **Save**.
1. In the left pane of the **pgAdmin4** window, under **Servers**, expand **Virtual Machine**.
1. Expand **Databases**, expand **adventureworks**, and then browse the schemas and tables in the database. The tables should be the same as those in the on-premises database.

### Task 5: Reconfigure and test the sample application against the database on the Azure virtual machine

1. Return to the **terminal** window.
1. Open the App.config file for the test application using the *nano* editor.

    ```bash
    nano App.config
    ```

1. Change the value of the **ConnectionString** setting and replace **localhost** with the IP address of the Azure virtual machine. The file should look like this:

    ```XML
    <?xml version="1.0" encoding="utf-8" ?>
    <configuration>
      <appSettings>
        <add key="ConnectionString" value="Server=nn.nn.nn.nn;Database=adventureworks;Port=5432;User Id=azureuser;Password=Pa55w.rd;" />
      </appSettings>
    </configuration>
    ```

    The application should now connect to the database running on the Azure virtual machine.

1. To save the file and close the editor, press ESC, then press CTRL X. Save your changes when you are prompted, by pressing Y and then Enter.
1. Build and run the application:

    ```bash
    dotnet run
    ```

    Verify that the application runs successfully, and returns the same number of rows for each table as before.

    You have now migrated your on-premises database to an Azure virtual machine, and reconfigured your application to use the new database.

## Exercise 2: Perform an online migration to Azure Database for PostgreSQL

In this exercise, you'll perform the following tasks:

1. Configure the PostgreSQL server running on the Azure virtual machine and export the schema.
1. Create the Azure Database for PostgreSQL server and database.
1. Import the schema into the target database.
1. Perform an online migration using the Database Migration Service
1. Modify data, and cut over to the new database
1. Verify the database in Azure Database for PostgreSQL
1. Reconfigure and test the sample application against the database in the Azure Database for PostgreSQL.

### Task 1: Configure the PostgreSQL server running on the Azure virtual machine and export the schema

1. Using a web browser, return to the Azure portal.
1. Open an Azure Cloud Shell window. Make sure that you are running the **Bash** shell.
1. Clone the repository holding the scripts and sample databases if you haven't done this previously.

    ```bash
    git clone https://github.com/MicrosoftLearning/DP-070-Migrate-Open-Source-Workloads-to-Azure ~/workshop
    ```

1. Move to the *~/workshop/migration_samples/setup/postgresql/adventureworks* folder.

    ```bash
    cd ~/workshop/migration_samples/setup/postgresql/adventureworks

1. Connect to the Azure virtual machine running the PostgreSQL server. In the following command, replace *nn.nn.nn.nn* with the IP address of the virtual machine. Enter the password **Pa55w.rdDemo** when prompted:

    ```bash
    ssh azureuser@nn.nn.nn.nn
    ```

1. On the virtual machine, switch to the *root* account. Enter the password for the *azureuser* user if prompted (**Pa55w.rdDemo**).

    ```bash
    sudo bash
    ```

1. Move to the directory */etc/postgresql/10/main*:

    ```bash
    cd /etc/postgresql/10/main
    ```

1. Using the *nano* editor, open the *postgresql.conf* file.

    ```bash
    nano postgresql.conf
    ```

1. Scroll to the bottom of the file, and verify that the following parameters have been configured:

    ```text
    listen_addresses = '*'
    wal_level = logical
    max_replication_slots = 5
    max_wal_senders = 10
    ```

1. To save the file and close the editor, press ESC, then press CTRL X. Save your changes, if you are prompted to, by pressing Enter.
1. Restart the PostgreSQL service:

    ```bash
    service postgresql restart
    ```

1. Verify that PostgreSQL has started correctly:

    ```bash
    service postgresql status
    ```

    If the service is running, you should see messages similar to the following:

    ```text
     postgresql.service - PostgreSQL RDBMS
      Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
       Active: active (exited) since Fri 2019-08-23 12:47:02 UTC; 2min 3s ago
       Process: 115562 ExecStart=/bin/true (code=exited, status=0/SUCCESS)
      Main PID: 115562 (code=exited, status=0/SUCCESS)

    Aug 23 12:47:02 postgreSQLVM systemd[1]: Starting PostgreSQL RDBMS...
    Aug 23 12:47:02 postgreSQLVM systemd[1]: Started PostgreSQL RDBMS.
    ```

1. Leave the *root* account and return to the *azureuser* account:

    ```bash
    exit
    ```

1. Run the following command to connect to the database on the Azure virtual machine. The password for the *azureuser* user in the PostgreSQL server running on the virtual machine is **Pa55w.rd**:

    ```bash
    psql -h [nn.nn.nn.nn] -U azureuser adventureworks
    ```

1. Grant replication permission to azureuser:

    ```SQL
    ALTER ROLE azureuser REPLICATION;
    ```

1. At the bash prompt, run the following command to export the schema for the **adventureworks** database to a file named **adventureworks_schema.sql**

    ```bash
    pg_dump -o  -d adventureworks -s > adventureworks_schema.sql
    ```

1. Disconnect from the virtual machine and return to the Cloud Shell prompt:

    ```bash
    exit
    ```

1. In the Cloud Shell, copy the schema file from the virtual machine. Replace *nn.nn.nn.nn* with the IP address of the virtual machine. Enter the password **Pa55w.rdDemo** when prompted::

    ```bash
    scp azureuser@nn.nn.nn.nn:~/adventureworks_schema.sql adventureworks_schema.sql
    ```

## Task 2. Create the Azure Database for PostgreSQL server and database

1. Switch to the Azure portal.
1. Select **+ Create a resource**.
1. In the **Search the Marketplace** box, type **Azure Database for PostgreSQL**, and press enter.
1. On the **Azure Database for PostgreSQL** page, select **Create**.
1. On the **Select Azure Database for PostgreSQL deployment option** page, in the **Single server** box, select **Create**.
1. On the **Single server** page, enter the following details:

    | Property  | Value  |
    |---|---|
    | Subscription | Select your subscription |
    | Resource group | Use the same resource group that you specified when you created the Azure virtual machine earlier in the *Setup* task for this lab |
    | Server name | **adventureworks*nnn***, where *nnn* is a suffix of your choice to make the server name unique |
    | Data source | None |
    | Location | Select your nearest location |
    | Version | 10 |
    | Compute + storage | Select **Configure server**, select the **Basic** pricing tier, and then select **OK** |
    | Admin username | awadmin |
    | Password | Pa55w.rdDemo |
    | Confirm password | Pa55w.rdDemo |

1. Select **Review + create**.
1. On the **Review + create** page, select **Create**. Wait for the service to be created before continuing.
1. When the service has been created, go to the page for the service in the portal, and select **Connection security**.
1. On the **Connection security page**, set **Allow access to Azure services** to **Yes**.
1. In the list of firewall rules, add a rule named **VM**, and set the **START IP ADDRESS** and **END IP ADDRESS** to the IP address of the virtual machine running the PostgreSQL server you created earlier.
1. Select **Add current client IP address**, to enable the **LON-DEV-01** virtual machine acting as the on-premises server to connect to Azure Database for PostgreSQL. You will need this access later, when running the reconfigured client application.
1. **Save**, and wait for the firewall rules to be updated.
1. At the Cloud Shell prompt, run the following command to create a new database in your Azure Database for PostgreSQL service. Replace *[nnn]* with the suffix you used when you created the Azure Database for PostgreSQL service. Replace *[resource group]* with the name of the resource group you specified for the service:

    ```bash
    az postgres db create \
      --name azureadventureworks \
      --server-name adventureworks[nnn] \
      --resource-group [resource group]
    ```

    If the database is created successfully, you should see a message similar to the following:

    ```json
    {
      "charset": "UTF8",
      "collation": "English_United States.1252",
      "id": "/subscriptions/nnnnnnnnnnnnnnnnnnnnnn/resourceGroups/nnnnnnnn/providers/Microsoft.DBforPostgreSQL/servers/adventureworksnnn/databases/azureadventureworks",
      "name": "azureadventureworks",
      "resourceGroup": "nnnnnnnn",
      "type": "Microsoft.DBforPostgreSQL/servers/databases"
    }
    ```

### Task 3: Import the schema into the target database

1. In the Cloud Shell, run the following command to connect to the azureadventureworks[nnn] server. Replace the two instances of *[nnn]* with the suffix for your service. Note that the username has the *@adventureworks[nnn]* suffix. At the password prompt, enter **Pa55w.rdDemo**.

    ```bash
    psql -h adventureworks[nnn].postgres.database.azure.com -U awadmin@adventureworks[nnn] -d postgres
    ```

1. Run the following commands to create a user named *azureuser* and set the password for this user to *Pa55w.rd*. The third statement gives the *azureuser* user the necessary privileges to create and manage objects in the *azureadventureworks* database. The *azure_pg_admin* role enables the *azureuser* user to install and use extensions in the database.

    ```SQL
    CREATE ROLE azureuser WITH LOGIN;
    ALTER ROLE azureuser PASSWORD 'Pa55w.rd';
    GRANT ALL PRIVILEGES ON DATABASE azureadventureworks TO azureuser;
    GRANT azure_pg_admin TO azureuser;
    ```

1. Close the *psql* utility with the **\q** command.
1. Import the schema for the **adventureworks** database to the **azureadventureworks** database running on your Azure Database for PostgreSQL service. You are performing the import as *azureuser*, so enter the password **Pa55w.rd** when prompted.

    ```bash
    psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks -f adventureworks_schema.sql
    ```

    You will see a series of messages as each item is created. The script should complete without any errors.

1. Run the following command. The *findkeys.sql* script generates another SQL script named *dropkeys.sql* that will remove all the foreign keys from the tables in the **azureadventureworks** database. You will run the *dropkeys.sql* script shortly:

    ```bash
    psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks -f findkeys.sql -o dropkeys.sql -t
    ```

    You can examine the *dropkeys.sql* script using a text editor if you have time.

1. Run the following command. The *createkeys.sql* script generates another SQL script named *addkeys.sql* that will recreate all the foreign keys. You will run the *addkeys.sql* script after you have migrated the database:

    ```bash
    psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks -f createkeys.sql -o addkeys.sql -t
    ```

1. Run the *dropkeys.sql* script:

    ```bash
    psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks -f dropkeys.sql
    ```

    You will see a series **ALTER TABLE** messages displayed, as the foreign keys are dropped.

1. Stat the psql utility again and connect to the *azureadventureworks* database.

    ```bash
    psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks
    ```

1. Run the following query to find the details of any remaining foreign keys:

    ```SQL
    SELECT constraint_type, table_schema, table_name, constraint_name
    FROM information_schema.table_constraints
    WHERE constraint_type = 'FOREIGN KEY';
    ```

    This query should return an empty result set. However, if any foreign keys still exist, for each foreign key, run the following command:

    ```SQL
    ALTER TABLE [table_schema].[table_name] DROP CONSTRAINT [constraint_name];
    ```

1. After you have removed any remaining foreign keys, execute the following SQL statement to display the triggers in the database:

    ```SQL
    SELECT trigger_name
    FROM information_schema.triggers;
    ```

    This query should also return an empty result set, indicating that the database contains no triggers. If the database did contain triggers, you would have to disable them before migrating the data, and re-enable them afterwards.

1. Close the *psql* utility with the **\q** command.

## Task 4.  Perform an online migration using the Database Migration Service

1. Switch back to the Azure portal.
1. Select **All services**, select **Subscriptions**, and then select your subscription.
1. On your subscription page, under **Settings**, select **Resource providers**.
1. In the **Filter by name** box, type **DataMigration**, and then select **Microsoft.DataMigration**.
1. If the **Microsoft.DataMigration** isn't registered, select **Register**, and wait for the **Status** to change to **Registered**. It might be necessary to select **Refresh** to see the status change.
1. Select **Create a resource**, in the **Search the Marketplace** box type **Azure Database Migration Service**, and then press Enter.
1. On the **Azure Database Migration Service** page, select **Create**.
1. On the **Create Migration Service** page, enter the following details:

    | Property  | Value  |
    |---|---|
    | Subscription | Select your own subscription |
    | Select a resource group | Specify the same resource group that you used for the Azure Database for PostgreSQL service and the Azure virtual machine |
    | Service name | adventureworks_migration_service |
    | Location | Select your nearest location |
    | Service mode | Azure |
    | Pricing tier | Premium, with 4 vCores |

1. Select **Next: Networking\>\>**.
1. On the **Networking** page, select the **postgresqlvnet/posgresqlvmSubnet** virtual network. This network was created as part of the setup.
1. Select **Review + create** and then select **Create**. Wait while the Database Migration Service is created. This will take a few minutes.
1. In the Azure portal, go to the page for your Database Migration Service.
1. Select **New Migration Project**.
1. On the **New migration project** page, enter the following details:

    | Property  | Value  |
    |---|---|
    | Project name | adventureworks_migration_project |
    | Source server type | PostgreSQL |
    | Target Database for PostgreSQL | Azure Database for PostgreSQL |
    | Choose type of activity | Online data migration |

1. Select **Create and run activity**.
1. When the **Migration Wizard** starts, on the **Select source** page, enter the following details:

    | Property  | Value  |
    |---|---|
    | Source server name | nn.nn.nn.nn *(The IP address of the Azure virtual machine running PostgreSQL)* |
    | Server port | 5432 |
    | Database | adventureworks |
    | User Name | azureuser |
    | Password | Pa55w.rd |
    | Trust server certificate | Selected |
    | Encrypt connection | Selected |

1. Select **Next: Select target\>\>**.
1. On the **Select target** page, enter the following details:

    | Property  | Value  |
    |---|---|
    | Subscription | Select your subscription |
    | Azure PostgreSQL | adventureworks[nnn] |
    | Database | azureadventureworks |
    | User Name | azureuser@adventureworks[nnn] |
    | Password | Pa55w.rd |

1. Select **Next: Select databases\>\>**.
1. On the **Select databases** page, select the **adventureworks** database and map it to **azureadventureworks**. Deselect the **postgres** database. Select **Next: Select tables\>\>**.
1. On the **Select tables** page, select **Next: Configure migration settings\>\>**.
1. On the **Configure migration settings** page, expand the **adventureworks** dropdown, expand the **Advanced online migration settings dropdown**, verify that **Maximum number of instances to load in parallel** is set to 5, and then select **Next: Summary\>\>**.
1. On the **Summary** page, in the **Activity name** box type **AdventureWorks_Migration_Activity**, and then select **Start migration**.
1. On the **AdventureWorks_Migration_Activity** page, select **Refresh** at 15 second intervals. You will see the status of the migration operation as it progresses. Wait until the **MIGRATION DETAILS** column changes to **Ready to cutover**.
1. Switch back to the Cloud Shell.
1. Run the following command to recreate the foreign keys in the **azureadventureworks** database. You generated the **addkeys.sql** script earlier:

    ```bash
    psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks -f addkeys.sql
    ```

    You will see a series of **ALTER TABLE** statements as the foreign keys are added. You may see an error concerning the *SpecialOfferProduct* table, which you can ignore for now. This is due to a UNIQUE constraint that doesn't get transferred correctly. In the real world, you should retrieve the details of this constraint from the source database using the following query:

    ```SQL
    SELECT constraint_type, table_schema, table_name, constraint_name
    FROM information_schema.table_constraints
    WHERE constraint_type = 'UNIQUE';
    ```

    You could then manually reinstate this constraint in the target database in Azure Database for PostgreSQL.

    There should be no other errors.

## Task 5. Modify data, and cut over to the new database

1. Return to the **AdventureWorks_Migration_Activity** page in the Azure portal.
1. Select the **adventureworks** database.
1. On the **adventureworks** page, verify that the **Full load completed** value is **66** and that all other values are **0**.
1. Switch back to the Cloud Shell.
1. Run the following command to connect to the **adventureworks** database running using PostgreSQL on the virtual machine:

    ```bash
    psql -h nn.nn.nn.nn -U azureuser -d adventureworks
    ```

1. Execute the following SQL statements to display, and then remove orders 43659, 43660, and 43661 from the database. Note that the database implements a cascading delete on the *salesorderheader* table, which automatically deletes the corresponding rows from the *salesorderdetail* table.

    ```SQL
    SELECT * FROM sales.salesorderheader WHERE salesorderid IN (43659, 43660, 43661);
    SELECT * FROM sales.salesorderdetail WHERE salesorderid IN (43659, 43660, 43661);
    DELETE FROM sales.salesorderheader WHERE salesorderid IN (43659, 43660, 43661);
    ```

1. Close the *psql* utility with the **\q** command.
1. Return to the **adventureworks** page in the Azure portal, and select **Refresh**. Verify that 32 changes have been applied.
1. Select **Start Cutover**.
1. On the **Complete cutover** page, select **Confirm**, and then select **Apply**. Wait until the status changes to **Completed**.
1. Return to the Cloud Shell.
1. Run the following command to connect to the **azureadventureworks** database running using your Azure Database for PostgreSQL service:

    ```bash
    psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks
    ```

1. Execute the following SQL statements to display the orders and order details in the database. Quit after the first page of each table. The purpose of these queries is to show that the data has been transferred:

    ```SQL
    SELECT * FROM sales.salesorderheader;
    SELECT * FROM sales.salesorderdetail;
    ```

1. Run the following SQL statements to display the orders and details for orders 43659, 43660, and 43661.

    ```SQL
    SELECT * FROM sales.salesorderheader WHERE salesorderid IN (43659, 43660, 43661);
    SELECT * FROM sales.salesorderdetail WHERE salesorderid IN (43659, 43660, 43661);
    ```

    Both queries should return 0 rows.

1. Close the *psql* utility with the **\q** command.

### Task 6: Verify the database in Azure Database for PostgreSQL

1. Return to the virtual machine acting as your on-premises computer
1. Switch to the **pgAdmin4** tool.
1. In the left-pane, right-click **Servers**, select **Create**, and then select **Server**.
1. In the **Create - Server** dialog box, on the **General** tab, in the **Name** box, enter **Azure Database for PostgreSQL**, and then select the **Connection** tab.
1. Enter the following details:

    | Property  | Value  |
    |---|---|
    | Host name/address | adventureworks*[nnn]*.postgres.database.azure.com |
    | Port | 5432 |
    | Maintenance database | postgres |
    | Username | azureuser@adventureworks*[nnn]* |
    | Password | Pa55w.rd |
    | Save password | Selected |
    | Role | *leave blank* |
    | Service | *Leave blank* |

1. Select the **SSL** tab.
1. On the **SSL** tab, set **SSL mode** to **Require**, and then select **Save**.
1. In the left pane of the **pgAdmin4** window, under **Servers**, expand **Azure Database for PostgreSQL**.
1. Expand **Databases**, expand **azureadventureworks**, and then browse the schemas and tables in the database. The tables should be the same as those in the on-premises database.

### Task 7: Reconfigure and test the sample application against the database in the Azure Database for PostgreSQL

1. Return to the **terminal** window.
1. Open the App.config file for the test application using the *nano* editor.

    ```bash
    nano App.config
    ```

1. Change the value of the **ConnectionString** setting and replace the IP address of the Azure virtual machine to **adventureworks[nnn].postgres.database.azure.com**.  Change the **Database** to **azureadventureworks**. Change the **User Id** to **azureuser@adventureworks[nnn]**. Append the text **Ssl Mode=Require;** to the end of the connection string. The file should look like this:

    ```XML
    <?xml version="1.0" encoding="utf-8" ?>
    <configuration>
      <appSettings>
        <add key="ConnectionString" value="Server=adventureworks[nnn].postgres.database.azure.com;Database=azureadventureworks;Port=5432;User Id=azureuser@adventureworks[nnn];Password=Pa55w.rd;Ssl Mode=Require;" />
      </appSettings>
    </configuration>
    ```

    The application should now connect to the database running on the Azure virtual machine.

1. To save the file and close the editor, press ESC, then press CTRL X. Save your changes when you are prompted, by pressing Y and then Enter.
1. Build and run the application:

    ```bash
    dotnet run
    ```

    The application will connect to the database, but will fail with the message **materialized view "vproductanddescription" has not been populated**. You need to refresh the materialized views in the database.

1. In the terminal window, use the *psql* utility to connect to the azureadventureworks database:

    ```bash
    psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks
    ```

1. Run the following query to list all the materialized views in the database:

    ```SQL
    SELECT schemaname, matviewname, matviewowner, ispopulated
    FROM pg_matviews;
    ```

    This query should return two rows, for *person.vstateprovincecountryregion*, and *production.vproductanddescription*. The *ispopulated* column for both views is *f*, indicating that they haven't yet been populated.

1. Run the following statements to populate both views:

   ```SQL
   REFRESH MATERIALIZED VIEW person.vstateprovincecountryregion;
   REFRESH MATERIALIZED VIEW production.vproductanddescription;
   ```

1. Close the *psql* utility with the **\q** command.
1. Run the application again:

    ```bash
    dotnet run
    ```

    Verify that the application now runs successfully.

    You have now migrated your database to Azure Database for PostgreSQL, and reconfigured your application to use the new database.
