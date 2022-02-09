# Lab: Migrate an on-premises MySQL database to Azure

## Overview

In this lab, you'll use the information learned in this module to migrate a MySQL database to Azure. To give complete coverage, students will perform an offline migration, transferring an on-premises MySQL database to a virtual machine running on Azure. 

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
    
    > **Tip**: If an error occurs, try updating to .NET 3.1 by using the following commands:
    > ```
    > sudo apt update
    > sudo apt install -y dotnet-sdk-3.1
    > ```


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
