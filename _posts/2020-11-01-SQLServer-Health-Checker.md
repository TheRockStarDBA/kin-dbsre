## SQL Server database health checker

#### The Journey begins 
In a multi-part blog post series, I will take you from designing through implementation details of how I built a SQL Server database health checker using [dbatools](https://dbatools.io/). Also, I will show you how easy is to run dbatools commands as windows service with proper logging.

#### Setup

Setup is pretty simple as shown below 

![SQLServerDBHealthChecker](/images/SQLServerDBHealthChecker.png)


#### Some background
With my current job as SRE (Site Reliability Engineer), I often encounter issues wherein application server/s have issues conneting to SQL Server. The issues could be related to a planned or unplanned failover, SQL Server becoming unresponsive due to high CPU or long blocking chain, etc. We as an SRE team need to know what is going on wrong with the database and based on the symptoms, we can trigger different actions e.g. automatically kickoff failover or page someone to perform appropriate mitigation.

We invest heavily in engineering work / automation thereby [eliminating toil](https://landing.google.com/sre/sre-book/chapters/eliminating-toil/) as much as possible. Also, we try to gather as much information as possible for the on-call engineer to be able to troubleshoot the issue much faster with all **relevant** data. This gives us the ability to reduce our application's MTTD (Mean Time To Detect) and MTTR (Mean Time To Recover) from any outage.
