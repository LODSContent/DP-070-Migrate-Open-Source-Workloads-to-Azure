# Lab: Migrate an on-premises MySQL database to Azure

## Overview

In this lab, you'll use the information learned in this module to migrate a MySQL database to Azure. To give complete coverage, students will perform two migrations. The first is an offline migration, transferring an on-premises MySQL database to a virtual machine running on Azure. The second is an online migration of the database running on the virtual machine to Azure Database for MySQL.

You'll also reconfigure and run a sample application that uses the database, to verify that the database operates correctly after each migration.

## Objectives

After completing this lab, you will be able to:

1. Perform an offline migration of an on-premises MySQL database to an Azure virtual machine.
1. Perform an online migration of a MySQL database running on a virtual machine to Azure Database for MySQL.

## Scenario

You work as a database developer for the AdventureWorks organization. AdventureWorks has been selling bicycles and bicycle parts directly to end-consumer and distributors for over a decade. Their systems store information in a database that currently runs using MySQL, located in their on-premises datacenter. As part of a hardware rationalization exercise, AdventureWorks want to move the database to Azure. You have been asked to perform this migration.

Initially, you decide to relocate quickly the data to a MySQL database running on an Azure virtual machine. This is considered to be a low-risk approach as it requires few if any changes to the database. However, this approach does require that you continue to perform most of the day-to-day monitoring and administrative tasks associated with the database. You also need to consider how the customer base of AdventureWorks has changed. Initially AdventureWorks targeted customers in their local region, but now they expanded to be a world-wide operation. Customers can be located anywhere, and ensuring that customers querying the database are subject to minimal latency is a primary concern. You can implement MySQL replication to virtual machines located in other regions, but again this is an administrative overhead.

Instead, once you have got the system running on a virtual machine, you will then consider a more long-term solution; you will migrate the database to Azure Database for MySQL. This PaaS strategy removes much of the work associated with maintaining the system. You can easily scale the system, and add read-only replicas to support customers anywhere in the world. Additionally Microsoft provide a guaranteed SLA for availability.

## Setup

You have an on-premises environment with an existing MySQL database containing the data that you wish to migrate to Azure. Before you start the lab, you need to create an Azure virtual machine running MySQL that will act as the target for the initial migration. We have provided a script that creates this virtual machine and configures it. Use the following steps to download and run this script:

1. Sign in to the **LON-DEV-01** virtual machine running in the classroom environment. The username is **azureuser**, and the password is **Pa55w.rd**.

    This virtual machine simulates your on-premises environment. It is running a MySQL server that is hosting the AdventureWorks database that you need to migrate.

1. Using a browser, sign in to the Azure portal.
1. Open an Azure Cloud Shell window. Make sure that you are running the **Bash** shell.
1. Clone the repository holding the scripts and sample databases if you haven't done this previously.

    ```bash
    git clone https://github.com/MicrosoftLearning/DP-070-Migrate-Open-Source-Workloads-to-Azure workshop 
    ```

1. Move to the *migration_samples/setup* folder.

    ```bash
    cd ~/workshop/migration_samples/setup
    ```

1. Run the *create_mysql_vm.sh* script as follows. Specify the name of a resource group and a location for holding the virtual machine as parameters. The resource group will be created if it doesn't already exist. Specify a location near to you, such as *eastus* or *uksouth*:

    ```bash
    bash create_mysql_vm.sh [resource group name] [location]
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

1. On the **LON-DEV-01** virtual machine running in the classroom environment, in the **Favorites** bar on the left-hand side of the screen, select **MySQLWorkbench**.
1. In the **MySQLWorkbench** window, select **LON-DEV-01**, and select **OK**.
1. Expand **adventureworks**, and then expand **Tables**.
1. Right-click the **contact** table, select **Select Rows - Limit 1000**, and then, in the **contact** query window, select **Limit to 1000 rows** and select **Don't Limit**.
1. Select **Execute** to run the query. It should return 19972 rows.
1. In the list of tables, right-click the **Employee** table, and select **Select rows**.
1. Select **Execute** to run the query. It should return 290 rows.
1. Spend a couple of minutes browsing the data for the other tables in the various tables in the database.

### Task 2: Run a sample application that queries the database

1. On the **LON-DEV-01** virtual machine, on the favorites bar, select **Show Applications** and then type **term**.
1. Select **Terminal** to open a terminal window.
1. In the terminal window, download the sample code for the lab. If prompted, enter **Pa55w.rd** for the password:

   ```bash
   sudo rm -rf ~/workshop
   git clone  https://github.com/MicrosoftLearning/DP-070-Migrate-Open-Source-Workloads-to-Azure ~/workshop
   ```

1. Move to the *~/workshop/migration_samples/code/mysql/AdventureWorksQueries* folder:

   ```bash
   cd ~/workshop/migration_samples/code/mysql/AdventureWorksQueries
   ```

    This folder contains a sample app that runs queries to count the number of rows in several tables in the *adventureworks* database.

1. Run the app:

    ```bash
    dotnet run
    ```

    The app should generate the following output:

    ```bash
    Querying AdventureWorks database
    SELECT COUNT(*) FROM product
    504

    SELECT COUNT(*) FROM vendor
    104

    SELECT COUNT(*) FROM specialoffer
    16

    SELECT COUNT(*) FROM salesorderheader
    31465

    SELECT COUNT(*) FROM salesorderdetail
    121317

    SELECT COUNT(*) FROM customer
    19185
    ```

### Task 3: Perform an offline migration of the database to the Azure virtual machine

Now that you have an idea of the data in the adventureworks database, you can migrate it to the MySQL server running on the virtual machine in Azure. You'll perform this operation as an offline task, using backup and restore commands.

> [!NOTE]
> If you wanted to migrate the data online, you could configure replication from the on-premises database to the database running on the Azure virtual machine.

1. From the terminal window, run the following command to take a backup of the *adventureworks* database. Note that the MySQL server on the LON-DEV-01 virtual machine is listening using port 3306:

    ```bash
    mysqldump -u azureuser -pPa55w.rd adventureworks > aw_mysql_backup.sql
    ```

1. Using the Azure Cloud shell, connect to the virtual machine containing the MySQL server and database. Replace \<*nn.nn.nn.nn*\> with the IP address of the virtual machine. If you are asked if you want to continue, type **yes** and press Enter.

    ```bash
    ssh azureuser@nn.nn.nn.nn
    ```

1. Type **Pa55w.rdDemo** and press Enter.
1. Connect to the MySQL server:

    ```bash
    mysql -u azureuser -pPa55w.rd
    ```

1. Create the target database on the Azure virtual machine:

    ```sql
    create database adventureworks;
    ```

1. Quit MySQL:

    ```bash
    quit
    ```

1. Exit the SSH session:

    ```bash
    exit
    ```

1. Restore the backup into the new database by running this mysql command in the LON-DEV-01 terminal:

    ```bash
    mysql -h [nn.nn.nn.nn] -u azureuser -pPa55w.rd adventureworks < aw_mysql_backup.sql
    ```

    This command will take a few minutes to run.

### Task 4: Verify the database on the Azure virtual machine

1. Run the following command to connect to the database on the Azure virtual machine. The password for the *azureuser* user in the MySQL server running on the virtual machine is **Pa55w.rd**:

    ```bash
    mysql -h [nn.nn.nn.nn] -u azureuser -pPa55w.rd adventureworks
    ```

1. Run the following query:

    ```SQL
    SELECT COUNT(*) FROM specialoffer;
    ```

    Verify that this query returns 16 rows. This is the same number of rows that is in the on-premises database.

1. Query the number of rows in the *vendor* table.

    ```SQL
    SELECT COUNT(*) FROM vendor;
    ```

    This table should contain 104 rows.

1. Close the *mysql* utility with the **quit** command.
1. Switch to the **MySQL Workbench** tool.
1. On the **Database** menu, select **Manage Connections**, and select **New**.
1. Select the **Connection** tab.
1. In **Connection name** type **MySQL on Azure VM**
1. Enter the following details:

    | Property  | Value  |
    |---|---|
    | Hostname | *[nn.nn.nn.nn]* |
    | Port | 3306 |
    | Username | azureuser |
    | Default Schema | adventureworks |

1. Select **Test connection**.
1. At the password prompt, type **Pa55w.rd** and select **OK**.
1. Select **OK** and select **Close**.
1. On the **Database** menu, select **Connect to Database**, select **MySQL on Azure VM**, and then select **OK**
1. In **adventureworks** browse the tables in the database. The tables should be the same as those in the on-premises database.

### Task 5: Reconfigure and test the sample application against the database on the Azure virtual machine

1. Return to the **terminal** window.
1. Open the App.config file for the test application using the *nano* editor:

    ```bash
    nano App.config
    ```

1. Change the value of the **ConnectionString** setting and replace **127.0.0.1** with the IP address of the Azure virtual machine. The file should look like this:

    ```XML
    <?xml version="1.0" encoding="utf-8" ?>
    <configuration>
    <appSettings>
        <add key="ConnectionString" value="Server=nn.nn.nn.nn;Database=adventureworks;Uid=azureuser;Pwd=Pa55w.rd;" />
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

## Exercise 2: Perform an online migration to Azure Database for MySQL

In this exercise, you'll perform the following tasks:

1. Configure the MySQL server running on the Azure virtual machine and export the schema.
1. Create the Azure Database for MySQL server and database.
1. Import the schema into the target database.
1. Perform an online migration using the Database Migration Service.
1. Modify data, and cutover to the new database.
1. Verify the database in Azure Database for MySQL.
1. Reconfigure and test the sample application against the database in the Azure Database for MySQL.

### Task 1: Configure the MySQL server running on the Azure virtual machine and export the schema

1. Using a web browser, return to the Azure portal.
1. Open an Azure Cloud Shell window. Make sure that you are running the **Bash** shell.
1. Connect to the Azure virtual machine running the MySQL server. In the following command, replace *nn.nn.nn.nn* with the IP address of the virtual machine. Enter the password **Pa55w.rdDemo** when prompted:

    ```bash
    ssh azureuser@nn.nn.nn.nn
    ```

1. Verify that MySQL has started correctly:

    ```bash
    service mysql status
    ```

    If the service is running, you should see messages similar to the following:

    ```bash
         mysql.service - MySQL Community Server
           Loaded: loaded (/lib/systemd/system/mysql.service; enabled; vendor preset: enabled)
           Active: active (running) since Mon 2019-09-02 14:45:42 UTC; 21h ago
          Process: 15329 ExecStart=/usr/sbin/mysqld --daemonize --pid-file=/run/mysqld/mysqld.pid (code=exited, status=0/SUCCESS)
          Process: 15306 ExecStartPre=/usr/share/mysql/mysql-systemd-start pre (code=exited, status=0/SUCCESS)
         Main PID: 15331 (mysqld)
            Tasks: 30 (limit: 4070)
           CGroup: /system.slice/mysql.service
                   └─15331 /usr/sbin/mysqld --daemonize --pid-file=/run/mysqld/mysqld.pid
        
            Sep 02 14:45:41 mysqlvm systemd[1]: Starting MySQL Community Server...
            Sep 02 14:45:42 mysqlvm systemd[1]: Started MySQL Community Server.
    ```

1. Export the schema for the source database using the mysqldump utility:

    ```bash
    mysqldump -u azureuser -pPa55w.rd adventureworks --no-data > adventureworks_mysql_schema.sql
    ```

1. At the bash prompt, run the following command to export the **adventureworks** database to a file named **adventureworks_mysql.sql**

    ```bash
    mysqldump -u azureuser -pPa55w.rd adventureworks > adventureworks_mysql.sql
    ```

1. Disconnect from the virtual machine and return to the Cloud Shell prompt:

    ```bash
    exit
    ```

1. In the Cloud Shell, copy the schema file from the virtual machine. Replace *nn.nn.nn.nn* with the IP address of the virtual machine. Enter the password **Pa55w.rdDemo** when prompted::

    ```bash
    scp azureuser@nn.nn.nn.nn:~/adventureworks_mysql_schema.sql adventureworks_mysql_schema.sql
    ```

### Task 2. Create the Azure Database for MySQL server and database

1. Switch to the Azure portal.
1. Select **+ Create a resource**.
1. In the **Search the Marketplace** box, type **Azure Database for MySQL**, and press enter.
1. On the **Azure Database for MySQL** page, select **Create**.
1. On the **Select Azure Database for MySQL deployment option** page, under **Single server** select **Create**.
1. On the **Create MySQL server** page, enter the following details:

    | Property  | Value  |
    |---|---|
    | Subscription | Your subscription |
    | Resource group | Use the same resource group that you specified when you created the Azure virtual machine earlier in the *Setup* task for this lab |
    | Server name | **adventureworks*nnn***, where *nnn* is a suffix of your choice to make the server name unique |
    | Data source | None |
    | Location | Select your nearest location |
    | Version | 5.7 |
    | Compute + storage | Select **Configure server**, select the **Basic** pricing tier, and then select **OK** |
    | Admin username | awadmin |
    | Password | Pa55w.rdDemo |
    | Confirm password | Pa55w.rdDemo |

1. Select **Review + create**.
1. On the **Review + create** page, select **Create**. Wait for the service to be created before continuing.
1. When the service has been created, select **Go to resource**, and then under **Settings**, and select **Connection security**.
1. On the **Connection security page**, set **Allow access to Azure services** to **Yes**.
1. In the list of firewall rules, add a rule named **VM**, and set the **START IP ADDRESS** and **END IP ADDRESS** to the IP address of the virtual machine running the MySQL server.
1. Select **Add current client IP address**, to enable the **LON-DEV-01** virtual machine acting as the on-premises server to connect to Azure Database for MySQL. You will need this access later, when running the reconfigured client application.
1. Under **SSL settings**, next to **Enfore SSL connection**, select **DISABLED**
1. **Save**, and wait for the firewall rules to be updated.
1. At the Cloud Shell prompt, run the following command to create a new database in your Azure Database for MySQL service. Replace *[nnn]* with the suffix you used when you created the Azure Database for MySQL service. Replace *[resource group]* with the name of the resource group you specified for the service:

    ```bash
    az MySQL db create \
    --name azureadventureworks \
    --server-name adventureworks[nnn] \
    --resource-group [resource group]
    ```

    If the database is created successfully, you should see a message similar to the following:

    ```json
    {
          "charset": "latin1",
          "collation": "latin1_swedish_ci",
          "id": "/subscriptions/nnnnnnnnnnnnnnnnnnnnnnnnnnnnn/resourceGroups/nnnnnn/providers/Microsoft.DBforMySQL/servers/adventureworksnnnn/databases/azureadventureworks",
          "name": "azureadventureworks",
          "resourceGroup": "nnnnn",
          "type": "Microsoft.DBforMySQL/servers/databases"
    }
    ```

### Task 3: Import the schema into the target database

1. In the Cloud Shell, run the following command to connect to the azureadventureworks[nnn] server. Replace the two instances of *[nnn]* with the suffix for your service. Note that the username has the *@adventureworks[nnn]* suffix:

    ```bash
    mysql -h adventureworks[nnn].MySQL.database.azure.com -u awadmin@adventureworks[nnn] -p
    ```

1. At the **Enter password** prompt, enter **Pa55w.rdDemo**.
1. Run the following commands to create a user named *azureuser* and set the password for this user to *Pa55w.rd*. The second statement gives the *azureuser* user the necessary privileges to create objects in the *azureadventureworks* database.

    ```SQL
    GRANT SELECT ON *.* TO 'azureuser'@'localhost' IDENTIFIED BY 'Pa55w.rd';
    GRANT CREATE ON *.* TO 'azureuser'@'localhost';
    ```

1. Run the following commands to create an *adventureworks* database.

    ```SQL
    CREATE DATABASE adventureworks;
    ```

1. Close the *mysql* utility with the **quit** command.
1. Import the **adventureworks** schema to your Azure Database for MySQL service. You are performing the import as *azureuser*, so enter the password **Pa55w.rd** when prompted.

    ```bash
    mysql -h adventureworks[nnnn].MySQL.database.azure.com -u awadmin@adventureworks[nnn] -pPa55w.rdDemo adventureworks < adventureworks_mysql_schema.sql
    ```

### Task 4.  Perform an online migration using the Database Migration Service

1. Switch back to the Azure portal.
1. In the menu on the left, select **Subscriptions**, and then select your subscription.
1. On your subscription page, under **Settings**, select **Resource providers**.
1. In the **Filter by name** box, type **DataMigration**, and then select **Microsoft.DataMigration**.
1. If the **Microsoft.DataMigration** isn't registered, select **Register**, and wait for the **Status** to change to **Registered**. It might be necessary to select **Refresh** to see the status change.
1. Select **Create a resource**, in the **Search the Marketplace** box type **Azure Database Migration Service**, and then press Enter.
1. On the **Azure Database Migration Service** page, select **Create**.
1. On the **Create Migration Service** page, enter the following details:

    | Property  | Value  |
    |---|---|
    | Subscription | Select your own subscription |
    | Select a resource group | Specify the same resource group that you used for the Azure Database for MySQL service and the Azure virtual machine |
    | Migration service name | adventureworks_migration_service |
    | Location | Select your nearest location |
    | Service mode | Azure |
    | Pricing tier | Premium, with 4 vCores |

1. Select **Next: Networking \>\>**.
1. On the **Networking** page, select the **MySQLvnet/mysqlvmSubnet** virtual network. This network was created as part of the setup.
1. Select **Review + create** and then select **Create**. Wait while the Database Migration Service is created. This will take a few minutes.
1. In the Azure portal, go to the page for your Database Migration Service.
1. Select **New Migration Project**.
1. On the **New migration project** page, enter the following details:

    | Property  | Value  |
    |---|---|
    | Project name | adventureworks_migration_project |
    | Source server type | MySQL |
    | Target Database for MySQL | Azure Database for MySQL |
    | Choose type of activity | Online data migration |

1. Select **Create and run activity**.
1. When the **Migration Wizard** starts, on the **Select source** page, enter the following details:

    | Property  | Value  |
    |---|---|
    | Source server name | nn.nn.nn.nn *(The IP address of the Azure virtual machine running MySQL)* |
    | Server port | 3306 |
    | User Name | azureuser |
    | Password | Pa55w.rd |

1. Select **Next: Select target\>\>**.
1. On the **Select target** page, enter the following details:

    | Property  | Value  |
    |---|---|
    | Target server name | adventureworks[nnn].MySQL.database.azure.com |
    | User Name | awadmin@adventureworks[nnn] |
    | Password | Pa55w.rdDemo |

1. Select **Next: Select databases\>\>**.
1. On the **Select databases** page, ensure that both the **Source Database** and the **Target Database** are set to **adventureworks** and then select **Next: Configure migration settings**.
1. On the **Configure migration settings** page, select **Next: Summary\>\>**.
1. On the **Migration summary** page, in the **Activity name** box type **AdventureWorks_Migration_Activity**, and then select **Start migration**.
1. On the **AdventureWorks_Migration_Activity** page, select **Refresh** at 15 second intervals. You will see the status of the migration operation as it progresses. Wait until the **MIGRATION DETAILS** column changes to **Ready to cutover**.

### Task 5. Modify data, and cutover to the new database

1. Return to the **AdventureWorks_Migration_Activity** page in the Azure portal.
1. Select the **adventureworks** database.
1. On the **adventureworks** page, verify that the status for all tables is marked as **COMPLETED**.
1. Select **Incremental data sync**. Verify that the status for every table is marked as **Syncing**.
1. Switch back to the Cloud Shell.
1. Run the following command to connect to the **adventureworks** database running using MySQL on the virtual machine:

    ```bash
    mysql -h nn.nn.nn.nn -u azureuser -pPa55w.rd adventureworks
    ```

1. Execute the following SQL statements to display, and then remove orders 43659, 43660, and 43661 from the database. Note that the database implements a cascading delete on the *salesorderheader* table, which automatically deletes the corresponding rows from the *salesorderdetail* table.

    ```SQL
    SELECT * FROM salesorderheader WHERE salesorderid IN (43659, 43660, 43661);
    SELECT * FROM salesorderdetail WHERE salesorderid IN (43659, 43660, 43661);
    DELETE FROM salesorderheader WHERE salesorderid IN (43659, 43660, 43661);
    ```

1. Close the *mysql* utility with the **quit** command.
1. Return to the **adventureworks** page in the Azure portal, and then select **Refresh**. Scroll to the page for the *salesorderheader* and *salesorderdetail* tables. Verify that the *salesorderheader* table indicates that 3 rows have been deleted, and 29 rows have been removed from the **sales.salesorderdetail** table. If there are no updates applied, check that there are **Pending changes** for the database.
1. Select **Start cutover**.
1. On the **Complete cutover** page, select **Confirm**, and then select **Apply**. Wait until the status changes to **Completed**.
1. Return to the Cloud Shell.
1. Run the following command to connect to the **azureadventureworks** database running using your Azure Database for MySQL service:

    ```bash
    mysql -h adventureworks[nnn].MySQL.database.azure.com -u awadmin@adventureworks[nnn] -pPa55w.rdDemo adventureworks
    ```

1. Run the following SQL statements to display the orders and details for orders 43659, 43660, and 43661. The purpose of these queries is to show that the data has been transferred:

    ```SQL
    SELECT * FROM salesorderheader WHERE salesorderid IN (43659, 43660, 43661);
    SELECT * FROM salesorderdetail WHERE salesorderid IN (43659, 43660, 43661);
    ```

    The first query should return 3 rows. The second query should return 29 rows.

1. Close the *mysql* utility with the **quit** command.

### Task 6: Verify the database in Azure Database for MySQL

1. Return to the virtual machine acting as your on-premises computer
1. Switch to the **MySQL Workbench** tool.
1. On the **Database** menu, select **Connect to database**
1. Enter the following details:

    | Property  | Value  |
    |---|---|
    | Hostname | adventureworks*[nnn]*.MySQL.database.azure.com |
    | Port | 3306 |
    | Username | awadmin@adventureworks*[nnn]* |
    | Password | Pa55w.rdDemo |

1. Select **OK**.
1. Expand **Databases**, expand **adventureworks**, and then browse the tables in the database. The tables should be the same as those in the on-premises database.

### Task 7: Reconfigure and test the sample application against the database in the Azure Database for MySQL

1. Return to the **terminal** window on the **LON-DEV-01** virtual machine.
1. Move to the *workshop/migration_samples/code/mysql/AdventureWorksQueries* folder:

   ```bash
   cd ~/workshop/migration_samples/code/mysql/AdventureWorksQueries
   ```

1. Open the App.config file using the nano editor:

    ```bash
    nano App.config
    ```

1. Change the value of the **ConnectionString** setting and replace the IP address of the Azure virtual machine to **adventureworks[nnn].MySQL.database.azure.com**. Change the **User Id** to **awadmin@adventureworks[nnn]**. Change the **Password** to **Pa55w.rdDemo**. The file should look like this:

    ```XML
    <?xml version="1.0" encoding="utf-8" ?>
        <configuration>
          <appSettings>
            <add key="ConnectionString" value="Server=adventureworks[nnn].MySQL.database.azure.com;database=adventureworks;port=3306;uid=awadmin@adventureworks[nnn];password=Pa55w.rdDemo" />
          </appSettings>
        </configuration>
    ```

    The application should now connect to the database running on Azure Database for MySQL.

1. Save the file and close the editor.
1. Build and run the application:

    ```bash
    dotnet run
    ```

    The app should display the same results as before, except that it is now retrieving the data from the database running in Azure.

   You have now migrated your database to Azure Database for MySQL, and reconfigured your application to use the new database.