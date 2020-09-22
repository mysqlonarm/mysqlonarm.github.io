---
layout: post
title: Using PMM to track MySQL on ARM statistics
img: ./images/blog12/pmm.png
author: Krunal Bauskar
update: 22nd Sep 2020
---

Percona Monitoring and Management (PMM) is an effective tool in tracking stats of the running MySQL servers. Especially, the timelines capability helps users to get the picture of how the given stats changes over tenure of the workload. PMM official packages are not yet available on ARM but part of the PMM (importantly the stats collector aka exporter) could be compiled on ARM that would facilitate reporting stats of the MySQL instance running on ARM to PMM-Server there-by allowing it to track MySQL on ARM.

## <span style="color:#4885ed">Background</span>

Few days back Agustin from Percona tried compiling PMM Client on ARM. You can read more about it [here](https://www.percona.com/blog/2020/08/04/compiling-a-percona-monitoring-and-management-v2-client-in-arm-architecture/). I just extended the process to also compile mysqld_exporter that is needed to connect and collect MySQL stats.

### <span style="color:#1aa260">Compiling mysqld-exporter</span>

Steps assume you have followed the blog above and you have needed structure in place that compiles and enables node_exporter.

Next is to checkout and compile mysqld_exporter

```

cd ~/go/src/github.com/percona/
git clone https://github.com/percona/mysqld_exporter.git
cd mysqld_exporter
make build

<failure>
cmd/go: unsupported GOOS/GOARCH pair linux/aarch64
Makefile:65: recipe for target 'promu' failed
</failure>

<temp-fix>
diff --git a/Makefile b/Makefile
index 08304cb..d98a49a 100644
--- a/Makefile
+++ b/Makefile
@@ -63,7 +63,7 @@ docker:
 
 promu:
 	@GOOS=$(shell uname -s | tr A-Z a-z) \
-		GOARCH=$(subst x86_64,amd64,$(patsubst i%86,386,$(shell uname -m))) \
+		GOARCH=$(subst aarch64,arm64,$(shell uname -m)) \
 		$(GO) get -u github.com/prometheus/promu
 
</temp-fix>

make build
sudo cp -a mysqld_exporter /usr/local/percona/pmm2/exporters/
```

### <span style="color:#1aa260">Starting mysqld-exporter</span>

If you have used PMM then no changes here. Start mysqld-exporter using pmm-admin
(assuming pmm-agent is running)

```
For example:
pmm-admin add mysql mysql-on-arm --socket=/tmp/n1.sock


Service type  Service name         Address and port  Service ID
MySQL         mysql-on-arm         /tmp/n1.sock      /service_id/c3c0c58d-97cd-468b-b6d5-1dfed40dc5df

Agent type                  Status     Agent ID                                        Service ID
pmm_agent                   Connected  /agent_id/7973fe6c-eda1-48d3-87ab-ec8f1ffddd02  
node_exporter               Running    /agent_id/4cae9b52-26b2-460f-8f96-f8dda89dee1a  
mysqld_exporter             Running    /agent_id/42a50302-cba4-40df-9fd9-cf4522852269  /service_id/c3c0c58d-97cd-468b-b6d5-1dfed40dc5df
mysql_slowlog_agent         Waiting    /agent_id/7417f44b-3dd4-4b25-9f0f-9479f86449aa  /service_id/c3c0c58d-97cd-468b-b6d5-1dfed40dc5df
```
We have not enabled slow-log (on MySQL Server) so the slow agent is in a waiting state.

### <span style="color:#1aa260">Seeing things in action</span>

Node-name auto-detected is **```krunal-bauskar-arm-2-new```** and mysql-instance name is **```mysql-on-arm```**. Graph below plots InnoDB Buffer Pool stats for mysql-instance running on ARM with sysbench workload active.

<img src="/images/blog12/img1.png" width="120%" class="centerimg"/>
<br>

## <span style="color:#4885ed">Conclusion</span>

Start monitoring your MySQL on ARM instances with PMM.

<br>
<em>If you have more questions/queries do let me know. Will try to answer them.</em>
