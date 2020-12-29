## SQL Server database health checker

#### The Journey begins 

In this blog post, I will take you on a journey from designing through implementation details of how I built **a SQL Server database health checker** using [dbatools](https://dbatools.io/). Also, I will show you how easy is to run dbatools commands as windows service with proper logging.

#### Setup

Setup is pretty simple as shown below 

![SQLServerDBHealthChecker](/images/SQLServerDBHealthChecker_01.png)


#### Some background

With my current job as SRE (Site Reliability Engineer), I often encounter issues wherein application server/s have issues conneting to SQL Server. The issues could be related to a planned or unplanned failover, SQL Server becoming unresponsive due to high CPU or long blocking chain, etc. We as an SRE team need to know what is going on wrong with the database and based on the symptoms, we can trigger different actions e.g. automatically kickoff failover or page someone to perform appropriate mitigation.

We invest heavily in engineering work / automation thereby [eliminating toil](https://landing.google.com/sre/sre-book/chapters/eliminating-toil/) as much as possible. Also, we try to gather as much information as possible for the on-call engineer to be able to troubleshoot the issue much faster with all **relevant** data. This gives us the ability to reduce our application's MTTD (Mean Time To Detect) and MTTR (Mean Time To Recover) from any outage.

#### Adopting SRE mindset 

We need a prober that can probe our Availability group (AG) every X seconds and provide a good enough view about the health of AG. It answers 2 main questions - is the database readable and is the database writable ?

To perform above operation, we will need 2 probes (In simple terms, a probe is an extremely light weight mechanism to check health of a system).

- **IsAlive Probe**: This probe will check if database is readable or accessible. This solves the problem of database being started, but it is still recovering i.e. the db is not ready to process requests. This can happen when a failover happens and the new primary is taking long time to recover due to role reveral.
- **IsReady Probe**: This probe will check if the database is writable or not. This solves the problem of database having issues either due to blocking wherein the becomes slow and based on your application timeout settings, your application throws a timeout error.


#### Building blocks 

Now that we have an idea of what we want to achieve, I will dicusss the building blocks of this automation. We will need 

 - Database Objects :
    - Database table: `dbo.sql_health_check` - This table will be used to record successful writes from the application servers from where the connection was made to the DB.
    - Database Store Procedures 
      - dbo.usp_dbHealthCheckSelect (IsAlive Probe) : This SP will perform a basic `SELECT`. Good enough to tell if DB isAlive. 
      - dbo.usp_dbHealthCheckSelectWrite (IsReady Probe) : This SP will perform a `SELECT` + `INSERT` into `dbo.sql_health_check`
 - 
