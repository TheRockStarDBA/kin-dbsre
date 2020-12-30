---
title: "SQL Server database health checker"
layout: post
image: images/SQLServerDBHealthChecker_01.png
categories: [sql-server, db-health-checker, powershell, dbatools, automation]
---

#### **The Journey begins** 

 {% twitter https://twitter.com/TheRockstarDBA/status/1313487225234108417 %}

In this blog post, I will take you on a journey from designing through implementation details of how I built **a SQL Server database health checker** using [dbatools](https://dbatools.io/). Also, I will show you how easy is to run dbatools commands as windows service with proper logging.

#### **Setup**

Setup is pretty simple as shown below 

![]({{ site.baseurl }}/images/SQLServerDBHealthChecker_01.png "SQL Server database health checker")

#### **Some background**

With my current job as SRE (Site Reliability Engineer), I often encounter issues wherein application server/s have issues conneting to SQL Server. The issues could be related to a planned or unplanned failover, SQL Server becoming unresponsive due to high CPU or long blocking chain, etc. We as an SRE team need to know what is going on wrong with the database and based on the symptoms, we can trigger different actions e.g. automatically kickoff failover or page someone to perform appropriate mitigation.

We invest heavily in engineering work / automation thereby [eliminating toil](https://landing.google.com/sre/sre-book/chapters/eliminating-toil/) as much as possible. Also, we try to gather as much information as possible for the on-call engineer to be able to troubleshoot the issue much faster with all **relevant** data. This gives us the ability to reduce our application's MTTD (Mean Time To Detect) and MTTR (Mean Time To Recover) from any outage.

#### **Adopting SRE mindset** 

We need a prober that can probe our Availability group (AG) every X seconds and provide a good enough view about the health of AG. It answers 2 main questions - is the database readable and is the database writable ?

To perform above operation, we will need 2 probes (In simple terms, a probe is an extremely light weight mechanism to check health of a system).

- **IsAlive Probe**: This probe will check if database is readable or accessible. This solves the problem of database being started, but it is still recovering i.e. the db is not ready to process requests. This can happen when a failover happens and the new primary is taking long time to recover due to role reveral.
- **IsReady Probe**: This probe will check if the database is writable or not. This solves the problem of database having issues either due to blocking wherein the becomes slow and based on your application timeout settings, your application throws a timeout error.


#### **Building blocks** 

Now that we have an idea of what we want to achieve, I will dicusss the building blocks of this automation. We will need 

 - **Database Objects** :
    - Database table: `dbo.sql_health_check` - This table will be used to record successful writes from the application servers from where the connection was made to the DB.
    - Database Store Procedures 
      - `dbo.usp_dbHealthCheckSelect` (IsAlive Probe) : This SP will perform a basic `SELECT`. Good enough to tell if DB isAlive. 
      - `dbo.usp_dbHealthCheckSelectWrite` (IsReady Probe) : This SP will perform a `SELECT` + `INSERT` into `dbo.sql_health_check`
    - A SQL Agent job that runs periodically to `TRUNCATE` `dbo.sql_health_check` table. This is to make sure that we trim the health check table.
 - **PowerShell code / cmdlets** :
    - dbatools
      - `Connect-DbaInstance` : Connects to a given sql server instance using listener
      - `Invoke-DbaQuery` : Invokes IsAlive and IsReady stored procedures
      
    - PowerShell code functions 
      - main.ps1
        - `Invoke-DbHealthProbeIsAlive` : Performs IsAlive probe by calling `dbo.usp_dbHealthCheckSelect`
        - `Invoke-DbHealthProbeIsReady` : Performs IsReady probe by calling `dbo.usp_dbHealthCheckSelectWrite`
        - `Write-LogToJSON` : Writes to a log file as compressed JSON. JSON was a good choice since our log ingestion tool parses JSON efficiently.
        - `Rotate-LogFile` : Rotates the log file when it reaches 10K lines and keeps 5 files max files around
 - **Config File** :
   - This is where you will make changes and adjust as per your needs. The only thing that will need to be changed is `$global:databaseName`
 - **NSSM - the Non-Sucking Service Manager** :
   - This is the tool that will allow your powershell script to run as windows service. Grab the latest release of the tool from [here](https://nssm.cc/download).
      
    
#### **SQL Server Health Checker** :

Now, we have our building blocks that will help us build our health checker. Lets get to the install process :

- Create database objects :
   - [create table](https://github.com/TheRockStarDBA/SQLServerHealthChecker/blob/main/database_objects/tables/dbo_sql_health_check.sql)
   - [create stored procedures](https://github.com/TheRockStarDBA/SQLServerHealthChecker/tree/main/database_objects/stored_procedures)
   - make sure that the `application_user` is granted proper rights on the database objects. 
- Powershell code
   - Make sure that dbatools is installed on all application servers. See [install guide](https://dbatools.io/download/).
   - [`main.ps1`](https://github.com/TheRockStarDBA/SQLServerHealthChecker/blob/main/powershell/main.ps1) : This is the file that drives the logic of performing probes and writing to log file.
   - [`GlobalGeneric.ps1`](https://github.com/TheRockStarDBA/SQLServerHealthChecker/blob/main/powershell/GlobalGeneric.ps1) : This file just contains functions that are generic and are used in the `main.ps1`. They are pretty generic, so you can turn them into a module or just use them by importing.
   - [`Config.ps1`](https://github.com/TheRockStarDBA/SQLServerHealthChecker/blob/main/powershell/Config.ps1) : This is the file that contains all the variables or config parameters that can be different depending on environments. Most important to change is the database name which should be changed.

#### **Non-Sucking Service Manager (NSSM)**:

Now we have the health checker ready and we just need to run the entire probe as a windows service. A service that should restart when the machine reboots. We will run the health checker as service on all the application servers that connects to the Availability Group database.

Below is how you configure the above SQL Server Health Checker as service using NSSM :

 - Install NSSM. 
 - Follow [these steps to configure](https://github.com/TheRockStarDBA/SQLServerHealthChecker/blob/main/NSSM_install.ps1) `main.ps1` to run as windows service.
 
 Thats it ! Now you have a powershell script that is configured to run as windows service to monitor health of your AlwaysON Availability group database. 

#### **Few important things** :

- This will monitor one database in an AG group, but the script can be enhanced if you prefer to monitor multiple databases or multiple AGs. I dont see a point in monitoring multiple databases which are part of same AG.
- The script will run in an infinite loop with a sleep of 5 secs which is configurable (see [`config.ps1`](https://github.com/TheRockStarDBA/SQLServerHealthChecker/blob/main/powershell/Config.ps1#L14))
- As always, test the code that you find on internet and understand what it does. If you have questions about this process, [file an issue](https://github.com/TheRockStarDBA/SQLServerHealthChecker/issues) and I will be happy to respond.
- If you feel that the process can be improved, feel free to [create a pull request](https://github.com/TheRockStarDBA/SQLServerHealthChecker/pulls)
- I am using TimeSeries database and Grafana to view the metrics that I collect part of the health checker probes.

#### **Advantages of using probing techinique for monitoring health** :

- The probing provides a light weight mechanism of assurance that the health of database is fine i.e. the application is able to read and write to a database successfully.
- The probe also gives me the failover history - what server was primary at a given point in time. If you plot this on Grafana, you can visualize the failover history in a beautiful way.
- I can see READ and WRITE latencies from application side as well as from sql server side. This goes with the [RED method to orcestrate your service](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/).
- This health chekcer provides Error rate when encountered since we are logging all errors and can feed those into log ingestion pipeline or tools for alerting.
- Provides ONCALL engineer high enough confidence about the Availability of critial database along with end-to-end latency metrics.
- There is room for alerting when latency exceeds certain threshold to alert on a potential issue e.g. slow reads or slow writes. This helps early detection and reduce outages to certain extent.

Thanks for reading ! See you in 2021 :smile:

~ **Kin** 



