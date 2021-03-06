<!--
 Licensed to the Apache Software Foundation (ASF) under one
 or more contributor license agreements.  See the NOTICE file
 distributed with this work for additional information
 regarding copyright ownership.  The ASF licenses this file
 to you under the Apache License, Version 2.0 (the
 "License"); you may not use this file except in compliance
 with the License.  You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
-->

# Apache HBase HBCK2 Tool

HBCK2 is the successor to [hbck](https://hbase.apache.org/book.html#hbck.in.depth),
the hbase-1.x fixup tool (A.K.A _hbck1_). Use it in place of _hbck1_ making repairs
against hbase-2.x installs.

_HBCK2_ differs from _hbck1_ philosophically. Each run performs a discrete task rather than
presume the tool can  repair 'all problems'. It is more of the vein of
[`plumbing` than `porecelain`](https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain).

## _hbck1_
The _hbck_ tool that ships with hbase-1.x (A.K.A _hbck1_) should not be run against an
hbase-2.x cluster. It may do damage. While _hbck1_ is still bundled inside hbase-2.x
-- to minimize surprise -- it's write-facility (`-fix`) has been removed. It can report
on the state of an hbase-2.x cluster but its assessments are likely inaccurate since it
does not understand the internal workings of an hbase-2.x.

## Building _HBCK2_

Run:
```
mvn install
```
The built _HBCK2_ jar will be in the `target` sub-directory.

## Running _HBCK2_
The _HBCK2_ jar does not include dependencies; it is not built as a 'fat' jar.
Dependencies must be `provided`. Building, adjusting the target hbase version in the
top-level pom to match your deploy will make for the smoothest operation (See
the parent pom.xml `hbase-operator-tools` for the
[hbase.version to set](https://github.com/apache/hbase-operator-tools/blob/master/pom.xml#L126)).
Where this can get interesting is at runtime when _HBCK2_ is in advance of your hbase
deploy such that your hbase does not support all APIs in current _HBCK2_. Where
_HBCK2_ does not have needed server-side support it should fail gracefully.

The easiest means of 'providing' _HBCK2_ its dependencies is by launching
_HBCK2_ via the `$HBASE_HOME/bin/hbase` script. The `bin/hbase` script natively
makes mention of `hbck` -- there is a `hbck` option listed in the help output.
By default, running `bin/hbase hbck`, the built-in _hbck1_ tooling will be run.
To run _HBCK2_, you need to point at a built _HBCK2_ jar using the `-j` option
as in:
~~~~
 $  ${HBASE_HOME}/bin/hbase --config /etc/hbase-conf hbck -j ~/hbase-operator-tools/hbase-hbck2/target/hbase-hbck2-1.0.0-SNAPSHOT.jar
~~~~
where in the above, `/etc/hbase-conf` is where the deploy's configuration lives.
The _HBCK2_ jar is at
`~/hbase-operator-tools/hbase-hbck2/target/hbase-hbck2-1.0.0-SNAPSHOT.jar`.
The above command with no options or arguments passed will dump out the _HBCK2_ help:
```
usage: HBCK2 [OPTIONS] COMMAND <ARGS>
Options:
 -d,--debug                                       run with debug output
 -h,--help                                        output this help message
 -p,--hbase.zookeeper.property.clientPort <arg>   port of hbase ensemble
 -q,--hbase.zookeeper.quorum <arg>                hbase ensemble
 -s,--skip                                        skip hbase version check
                                                  (PleaseHoldException)
 -v,--version                                     this hbck2 version
 -z,--zookeeper.znode.parent <arg>                parent znode of hbase
                                                  ensemble
Command:
 addFsRegionsMissingInMeta <NAMESPACE|NAMESPACE:TABLENAME>...
   Options:
    -d,--force_disable aborts fix for table if disable fails.
   To be used in scenarios where some regions may be missing in META,
   but there's still a valid 'regioninfo' metadata file on HDFS.
   This is a lighter version of 'OfflineMetaRepair tool commonly used for
   similar issues on 1.x release line.
   This command needs META to be online. For each table name passed as
   parameter, it performs a diff between regions available in META,
   against existing regions dirs on HDFS. Then, for region dirs with
   no matches in META, it reads regioninfo metadata file and
   re-creates given region in META. Regions are re-created in 'CLOSED'
   state at META table only, but not in Masters' cache, and are not
   assigned either. To get these regions online, run HBCK2 'assigns'command
   printed at the end of this command results for convenience.

   NOTE: If using hbase releases older than 2.3.0, a rolling restart of
   HMasters is needed prior to executing the provided 'assigns' command.

   An example adding missing regions for tables 'tbl_1' on default
   namespace, 'tbl_2' on namespace 'n1' and for all tables from
   namespace 'n2':
     $ HBCK2 addFsRegionsMissingInMeta default:tbl_1 n1:tbl_2 n2
   Returns HBCK2 'assigns' command with all re-inserted regions.
   SEE ALSO: reportMissingRegionsInMeta

 assigns [OPTIONS] <ENCODED_REGIONNAME>...
   Options:
    -o,--override  override ownership by another procedure
   A 'raw' assign that can be used even during Master initialization (if
   the -skip flag is specified). Skirts Coprocessors. Pass one or more
   encoded region names. 1588230740 is the hard-coded name for the
   hbase:meta region and de00010733901a05f5a2a3a382e27dd4 is an example of
   what a user-space encoded region name looks like. For example:
     $ HBCK2 assign 1588230740 de00010733901a05f5a2a3a382e27dd4
   Returns the pid(s) of the created AssignProcedure(s) or -1 if none.

 bypass [OPTIONS] <PID>...
   Options:
    -o,--override   override if procedure is running/stuck
    -r,--recursive  bypass parent and its children. SLOW! EXPENSIVE!
    -w,--lockWait   milliseconds to wait before giving up; default=1
   Pass one (or more) procedure 'pid's to skip to procedure finish. Parent
   of bypassed procedure will also be skipped to the finish. Entities will
   be left in an inconsistent state and will require manual fixup. May
   need Master restart to clear locks still held. Bypass fails if
   procedure has children. Add 'recursive' if all you have is a parent pid
   to finish parent and children. This is SLOW, and dangerous so use
   selectively. Does not always work.

 filesystem [OPTIONS] [<TABLENAME...]
   Options:
    -f, --fix    sideline corrupt hfiles, bad links and references.
   Report corrupt hfiles and broken links. Pass '--fix' to sideline
   corrupt files and links. Pass one or more tablenames to narrow the
   checkup. Default checks all tables. Modified regions will need to be
   reopened to pick-up changes.

 reportMissingRegionsInMeta <NAMESPACE|NAMESPACE:TABLENAME>...
   To be used in scenarios where some regions may be missing in META,
   but there's still a valid 'regioninfo metadata file on HDFS.
   This is a checking only method, designed for reporting purposes and
   doesn't perform any fixes, providing a view of which regions (if any)
   would get re-added to meta, grouped by respective table/namespace.
   To effectively re-add regions in meta, addFsRegionsMissingInMeta should be executed.
   This command needs META to be online. For each namespace/table passed
   as parameter, it performs a diff between regions available in META,
   against existing regions dirs on HDFS. Region dirs with no matches
   are printed grouped under its related table name. Tables with no
   missing regions will show a 'no missing regions' message. If no
   namespace or table is specified, it will verify all existing regions.
   It accepts a combination of multiple namespace and tables. Table names
   should include the namespace portion, even for tables in the default
   namespace, otherwise it will assume as a namespace value.
   An example triggering missing regions report for tables 'table_1'
   and 'table_2', under default namespace:
     $ HBCK2 reportMissingRegionsInMeta default:table_1 default:table_2
   An example triggering missing regions report for table 'table_1'
   under default namespace, and for all tables from namespace 'ns1':
     $ HBCK2 reportMissingRegionsInMeta default:table_1 ns1
   Returns list of missing regions for each table passed as parameter, or
   for each table on namespaces specified as parameter.

 setRegionState <ENCODED_REGIONNAME> <STATE>
   Possible region states:
    OFFLINE, OPENING, OPEN, CLOSING, CLOSED, SPLITTING, SPLIT,
    FAILED_OPEN, FAILED_CLOSE, MERGING, MERGED, SPLITTING_NEW,
    MERGING_NEW, ABNORMALLY_CLOSED
   WARNING: This is a very risky option intended for use as last resort.
   Example scenarios include unassigns/assigns that can't move forward
   because region is in an inconsistent state in 'hbase:meta'. For
   example, the 'unassigns' command can only proceed if passed a region
   in one of the following states: SPLITTING|SPLIT|MERGING|OPEN|CLOSING
   Before manually setting a region state with this command, please
   certify that this region is not being handled by a running procedure,
   such as 'assign' or 'split'. You can get a view of running procedures
   in the hbase shell using the 'list_procedures' command. An example
   setting region 'de00010733901a05f5a2a3a382e27dd4' to CLOSING:
     $ HBCK2 setRegionState de00010733901a05f5a2a3a382e27dd4 CLOSING
   Returns "0" if region state changed and "1" otherwise.

 setTableState <TABLENAME> <STATE>
   Possible table states: ENABLED, DISABLED, DISABLING, ENABLING
   To read current table state, in the hbase shell run:
     hbase> get 'hbase:meta', '<TABLENAME>', 'table:state'
   A value of \x08\x00 == ENABLED, \x08\x01 == DISABLED, etc.
   Can also run a 'describe "<TABLENAME>"' at the shell prompt.
   An example making table name 'user' ENABLED:
     $ HBCK2 setTableState users ENABLED
   Returns whatever the previous table state was.

 scheduleRecovery <SERVERNAME>...
   Schedule ServerCrashProcedure(SCP) for list of RegionServers. Format
   server name as '<HOSTNAME>,<PORT>,<STARTCODE>' (See HBase UI/logs).
   Example using RegionServer 'a.example.org,29100,1540348649479':
     $ HBCK2 scheduleRecovery a.example.org,29100,1540348649479
   Returns the pid(s) of the created ServerCrashProcedure(s) or -1 if
   no procedure created (see master logs for why not).
   Command only supported in hbase versions 2.0.3, 2.1.2, 2.2.0 (or newer).

 unassigns <ENCODED_REGIONNAME>...
   Options:
    -o,--override  override ownership by another procedure
   A 'raw' unassign that can be used even during Master initialization
   (if the -skip flag is specified). Skirts Coprocessors. Pass one or
   more encoded region names. 1588230740 is the hard-coded name for the
   hbase:meta region and de00010733901a05f5a2a3a382e27dd4 is an example
   of what a userspace encoded region name looks like. For example:
     $ HBCK2 unassign 1588230740 de00010733901a05f5a2a3a382e27dd4
   Returns the pid(s) of the created UnassignProcedure(s) or -1 if none.

   SEE ALSO, org.apache.hbase.hbck1.OfflineMetaRepair, the offline
   hbase:meta tool. See the HBCK2 README for how to use.
```

Note that when you pass `bin/hbase` the `hbck` argument, it will by
default use the shaded client to get to the targeted hbase cluster.
This is sufficient for most _HBCK2_ usage. If you run into complaints
like the below:
```
bin/hbase --config hbase-conf  hbck
2019-08-30 05:04:54,467 WARN  [main] util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Exception in thread "main" java.io.IOException: No FileSystem for scheme: hdfs
        at org.apache.hadoop.fs.FileSystem.getFileSystemClass(FileSystem.java:2799)
        at org.apache.hadoop.fs.FileSystem.createFileSystem(FileSystem.java:2810)
        at org.apache.hadoop.fs.FileSystem.access$200(FileSystem.java:100)
        at org.apache.hadoop.fs.FileSystem$Cache.getInternal(FileSystem.java:2849)
        at org.apache.hadoop.fs.FileSystem$Cache.get(FileSystem.java:2831)
        at org.apache.hadoop.fs.FileSystem.get(FileSystem.java:389)
        at org.apache.hadoop.fs.Path.getFileSystem(Path.java:356)
        at org.apache.hadoop.hbase.util.CommonFSUtils.getRootDir(CommonFSUtils.java:361)
        at org.apache.hadoop.hbase.util.HBaseFsck.main(HBaseFsck.java:3605)
```
... it is because the HDFS jars are not on the CLASSPATH. The default is NOT
to bundle HDFS jars on the CLASSPATH when running `hbck` via `bin/hbase`. Define
`HADOOP_HOME` in the environment so `bin/hbase` can find your local hadoop
install and load its HDFS jars.

## _HBCK2_ Overview
_HBCK2_ is currently a simple tool that does one thing at a time only.

In hbase-2.x, the Master is the final arbiter of all state, so a general principal for most of
_HBCK2_ commands is that it asks the Master to effect all repair. This means a Master must be
up before you can run (most) _HBCK2_ commands.

_HBCK2_ implementation approach is to make use of an
`HbckService` hosted on the Master. The Service publishes a few methods for the _HBCK2_ tool to
pull on. Therefore, for _HBCK2_ commands relying on Master's `HbckService` facade,
first thing _HBCK2_ does is poke the cluster to ensure the service is available.
This will fail if the remote Server does not publish the Service or if the
`HbckService` is lacking the requested method.

_HBCK2_ versions should be able to work across multiple hbase-2 releases. It will
fail with a complaint if it is unable to run. There is no `HbckService` in versions
of hbase before 2.0.3 and 2.1.1. _HBCK2_ will not work against these versions.

As _HBCK2_ evolves independently from _HBase_ main project, there will be eventually the need to
define new fix methods with client side implementations (at least until a related one can be added
on Master's `HbckService` facade), so that _HBCK2_ can operate on such _HBase_ releases without
requiring a cluster upgrade. One example of such methods is the _setRegionState_.

## Finding Problems

While _hbck1_ performed analysis reporting your cluster GOOD or BAD, _HBCK2_
is less presumptious. In hbase-2.x, the operator figures what needs fixing and
then uses tooling including _HBCK2_ to do fixup.

To figure issues in assignment, make use of the following utilities.

### Diagnosis Tooling

#### Master Logs

The Master runs all assignments, server crash handling, cluster start and
stop, etc. In hbase-2.x, all that the Master does has been cast as
Procedures run on a state machine engine. See
[Procedure Framework](https://hbase.apache.org/book.html#pv2)
and [Assignment Manager](https://hbase.apache.org/book.html#amv2)
for detail on how this new infrastructure works. Each Procedure has a
unique Procedure `id`, its `pid`, that it lists on each logging.
Following the _pid_, you can trace the lifecycle of a Procedure in the
Master logs as Procedures transition from start, through each of the
Procedure's various stages to finish. Some Procedures spawn sub-procedures,
wait on their Children, and then themselves finish. Each child logs
its _pid_ but also its _ppid_; its parent's _pid_.

Generally all runs problem free but if some unforeseen circumstance
arises, the assignment framework may sustain damage requiring
operator intervention.  Below we will discuss some such scenarios
but they can manifest in the Master log as a Region being _STUCK_ or
a Procedure transitioning an entity -- a Region or a Table --
may be blocked because another Procedure holds the exclusive lock
and is not letting go.

_STUCK_ Procedures look like this:

```
2018-09-12 15:29:06,558 WARN org.apache.hadoop.hbase.master.assignment.AssignmentManager: STUCK Region-In-Transition rit=OPENING, location=va1001.example.org,22101,1536173230599, table=IntegrationTestBigLinkedList_20180626110336, region=dbdb56242f17610c46ea044f7a42895b
```


#### /master-status#tables

This section about midway down in Master UI home-page shows a list of tables
with columns for whether the table is _ENABLED_, _ENABLING_, _DISABLING_, or
_DISABLED_ among other attributes. Also listed are columns with counts
of Regions in their various transition states: _OPEN_, _CLOSED_,
etc. A read of this table is good for figuring if the Regions of
this table have a proper disposition. For example if a table is
_ENABLED_ and there are Regions that are not in the _OPEN_ state
and the Master Log is silent about any ongoing assigns, then
something is amiss.

#### Procedures & Locks

This page off the Master UI home page under the
_Procedures & Locks_ menu item in the page heading lists all ongoing
Procedures and Locks as well as the current set of Master Procedure WALs
(named _pv2-0000000000000000###.log_ under the _MasterProcWALs_
directory in your hbase install). On startup, on a large
cluster when furious assigning is afoot, this page is
filled with lists of Procedures and Locks. The count of
MasterProcWALs will bloat too. If after the cluster settles,
there is a stuck Lock or Procedure or the count of WALs
doesn't ever come down but only grows, then operator intervention
is needed to alieve the blockage.

Lists of locks and procedures can also be obtained via the hbase shell:

```
$ echo "list_locks"| hbase shell &> /tmp/locks.txt
$ echo "list_procedures"| hbase shell &> /tmp/procedures.txt
```

#### The 'HBCK Report'
An `HBCK Report` page was added to the Master in versions hbase 2.3.0/2.1.6/2.2.1
at `/hbck.jsp`
which shows output from two inspections run by the master on an interval; one
is output by the CatalogJanitor whenever it runs. If overlaps or holes in
`hbase:meta`, the CatalogJanitor half of the page will list what it has found
(otherwise it is quiet). Another background 'chore' process was added to compare
`hbase:meta` and filesystem content making compare; if anomaly, it will make
note in its `HBCK Report` section.

See the 'HBCK Report' page itself for how to force runs of the inspectors.


#### The [HBase Canary Tool](http://hbase.apache.org/book.html#_canary)

The Canary tool is useful verifying the state of assign.
It can be run with a table focus or against the whole cluster.

For example, to check cluster assigns:

```
$ hbase canary -f false -t 6000000 &>/tmp/canary.log
```

The _-f false_ tells the Canary to keep going across failed Region
fetches and the _-t 6000000_ tells the Canary run for ~two hours
maximum. When done, check out _/tmp/canary.log_. Grep for
_ERROR_ lines to find problematic Region assigns.

You can do a probe like the Canary's in the hbase shell. 
For example, given a Region that has a start row of _d1dddd0c_
belonging to the table _testtable_, do as follows:


```
hbase> scan 'testtable', {STARTROW => 'd1dddd0c', LIMIT => 10}
```

For an overview on parsing a Region name into its constituent parts, see
[RegionInfo API](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/RegionInfo.html).

#### Other Tools

To figure the list of Regions that are not _OPEN_ on an
_ENABLED_ or _ENABLING_ table, read the _hbase:meta_ table _info:state_ column.
For example, to find the state of all Regions in the table
_IntegrationTestBigLinkedList_20180626064758_, do the following:

```
$ echo " scan 'hbase:meta', {ROWPREFIXFILTER => 'IntegrationTestBigLinkedList_20180626064758,', COLUMN => 'info:state'}"| hbase shell > /tmp/t.txt
```

...then grep for _OPENING_ or _CLOSING_ Regions.

To move an _OPENING_ issue to _OPEN_ so it agrees with a table's
_ENABLED_ state, use the `assign` command in the hbase shell to
queue a new Assign Procedure (watch the Master logs to see the
Assign run). If many Regions to assign, use the _HBCK2_ tool. It
can do bulk assigning.

## Fixing

General principals include a Region can not be assigned if
it is in _CLOSING_ state (or the inverse, unassigned if in
_OPENING_ state) without first transitioning via _CLOSED_:
Regions must always move from _CLOSED_, to _OPENING_, to _OPEN_,
and then to _CLOSING_, _CLOSED_.

When making repair, do fixup a table at a time.

Also, if a table is _DISABLED_, you cannot assign a Region.
In the Master logs, you will see that the Master will report
that the assign has been skipped because the table is
_DISABLED_. You may want to assign a Region because it is
currently in the _OPENING_ state and you want it in the
_CLOSED_ state so it agrees with the table's _DISABLED_
state. In this situation, you may have to temporarily set
the table status to _ENABLED_, just so you can do the
assign, and then set it back again after the unassign.
_HBCK2_ has facility to allow you do this. See the
_HBCK2_ usage output.

### Start-over

At an extreme, if the Master is distraught and all attempts at fixup only
turn up undoable locks or Procedures that won't finish, and/or
the set of MasterProcWALs is growing without bound, it is
possible to wipe the Master state clean. Just move aside the
_/hbase/MasterProcWALs/_ directory under your hbase install and
restart the Master process. It will come back as a `tabula rasa` without
memory of the bad times past.

If at the time of the erasure, all Regions were happily
assigned or offlined, then on Master restart, the Master should
pick up and continue as though nothing happened. But if there were Regions-In-Transition
at the time, then the operator may have to intervene to bring outstanding
assigns/unassigns to their terminal point. Read the _hbase:meta_
_info:state_ columns as described above to figure what needs
assigning/unassigning. Having erased all history moving aside
the _MasterProcWALs_, none of the entities should be locked so
you are free to bulk assign/unassign.

### Assigning/Unassigning

Generally, on assign, the Master will persist until successful.
An assign takes an exclusive lock on the Region. This precludes
a concurrent assign or unassign from running. An assign against
a locked Region will wait until the lock is released before
making progress. See the [Procedures & Locks] section above for
current list of outstanding Locks.

### _Master startup cannot progress, in holding-pattern until region onlined_

This should never happen. If it does, here is what it looks like:

```
2018-10-01 22:07:42,792 WARN org.apache.hadoop.hbase.master.HMaster: hbase:meta,,1.1588230740 is NOT online; state={1588230740 state=CLOSING, ts=1538456302300, server=ve1017.example.org,22101,1538449648131}; ServerCrashProcedures=true. Master startup cannot progress, in holding-pattern until region onlined.
```

The Master is unable to continue startup because there is no Procedure to assign
_hbase:meta_ (or _hbase:namespace_). To inject one, use the _HBCK2_ tool:

```
HBASE_CLASSPATH_PREFIX=./hbase-hbck2-1.0.0-SNAPSHOT.jar hbase org.apache.hbase.HBCK2 assigns -skip 1588230740
```

...where 1588230740 is the encoded name of the _hbase:meta_ Region. Pass the '-skip' option to
stop HBCK2 doing a verstion check against the remote master. If the remote master is not up,
the version check will prompt a 'Master is initializing response' or 'PleaseHoldException'
and drop the assign attempt. The '-skip' command punts on version check and will land the
scheduled assign.

The same may happen to the _hbase:namespace_ system table. Look for the
encoded Region name of the _hbase:namespace_ Region and do similar to
what we did for _hbase:meta_. In this latter case, the Master actually
prints out a helpful message that looks like the following:

```2019-07-09 22:08:38,966 WARN  [master/localhost:16000:becomeActiveMaster] master.HMaster: hbase:namespace,,1562733904278.9559cf72b8e81e1291c626a8e781a6ae. is NOT online; state={9559cf72b8e81e1291c626a8e781a6ae state=CLOSED, ts=1562735318897, server=null}; ServerCrashProcedures=true. Master startup cannot progress, in holding-pattern until region onlined.```

To schedule an assign for the hbase:namespace table noted in the above log line, you would do:
```HBASE_CLASSPATH_PREFIX=./hbase-hbck2-1.0.0-SNAPSHOT.jar hbase org.apache.hbase.HBCK2 -skip assigns 9559cf72b8e81e1291c626a8e781a6ae```
... passing the encoded name for the namespace region (the encoded name will differ per deploy).

### Missing Regions in META - hbase:meta region/table restore/rebuild

There's been some extra-ordinary cases where table regions are removed from META table.
Some triage on such cases revealed those were operator-induced, after execution
attempts of the obsolete *hbck1* _OfflineMetaRepair_ tool. _OfflineMetaRepair_ is a well known tool
for fixing META table related issues on HBase 1.x versions. The original version is not compatible
with HBase 2.x or higher versions, and it has undergone some adjustments to be now run within hbck2.

In most of these cases, regions may be missing in meta at random, but hbase may still be
operational. In such situations, problem can be addressed with master online, using _addFsRegionsMissingInMeta_ command.
This command is less disruptive to hbase than the full meta rebuild covered later, and it can be used even for
recovering _namespace_ table region.

#### Online meta rebuild recipe

If meta corruption is not too critical, hbase would still be able to bring it online. Even if namespace region
is among the missing ones in meta, it will still be possible to scan meta in the initialization period,
where master will be waiting for namespace to be assigned. To verify on this, a meta scan command can be executed
as below. If it does not time out or show any errors, _meta_ is online:

```
echo "scan 'hbase:meta', {COLUMN=>'info:regioninfo'}" | hbase shell
```

HBCK2 _addFsRegionsMissingInMeta_ can be used if the above does not show any errors. It reads region
metadata info available on the FS region dirs, in order to re-create regions in META. Since it can
run with hbase partially operational, it attempts to disable online tables that are affected by the
reported problem and is gonna have regions re-added to _meta_.
It can check for specific tables/namespaces, or all tables
from all namespaces. An example adding missing regions for tables 'tbl_1' on default namespace,
'tbl_2' on namespace 'n1' and for all tables from namespace 'n2':

```
$ HBCK2 addFsRegionsMissingInMeta default:tbl_1 n1:tbl_2 n2
```

As it operates independently from Master, once it finishes successfully, additional steps are
required to actually have the re-added regions assigned. These are listed below:

1. _addFsRegionsMissingInMeta_ outputs an _assigns_ command with all regions that got re-added. This
command needs to be executed later, so copy and save it for convenience.

2. For HBase versions prior to 2.3.0, after _addFsRegionsMissingInMeta_ finished successfully and output has been saved,
restart all running HBase Masters.

3. Once Master's are restarted and META is already online (check if Web UI is accessible), run
_assigns_ command from _addFsRegionsMissingInMeta_ output saved per instructions from #1.

NOTE: If _namespace_ region is among the missing ones, you will need to add _--skip_ flag at the
beginning of _assigns_ command returned.


Should a cluster suffer a catastrophic loss of the `hbase:meta` region, a rough rebuild is possible following the below recipe.
In outline: stop the cluster; run the _OfflineMetaRepair_ tool which reads directories and metadata dropped into the filesystem making a best effort at reconstructing a viable _hbase:meta_ table; restart your cluster; inject an assign to bring the system namespace table online; and then finally, re-assign userspace tables you'd like enabled (the rebuilt _hbase:meta_ creates a table with all tables offline and no regions assigned).

#### Detailed rebuild recipe
Stop the cluster.

Run the rebuild _hbase:meta_ command from _HBCK2_. This will move aside the original _hbase:meta_ and put in place a newly rebuilt one. Below is an example of how to run the tool.  It adds the `-details` flag so the tool dumps info on the regions its found in hdfs:
```$ HBASE_CLASSPATH_PREFIX=~/checkouts/hbase-operator-tools/hbase-hbck2/target/hbase-hbck2-1.0.0-SNAPSHOT.jar ./bin/hbase org.apache.hbase.hbck1.OfflineMetaRepair -details```

Start the cluster up. It won’t come up fully. It will be stuck because the _namespace_ table is not online and there is no assign procedure in the procedure store for this contingency. The hbase master log will show this state. Here is an example of what it will log:
```2019-07-10 18:30:51,090 WARN  [master/localhost:16000:becomeActiveMaster] master.HMaster: hbase:namespace,,1562808216225.725a0fe6c2c869d3d0a9ed82bfa80fa3. is NOT online; state={725a0fe6c2c869d3d0a9ed82bfa80fa3 state=CLOSED, ts=1562808619952, server=null}; ServerCrashProcedures=false. Master startup cannot progress, in holding-pattern until region onlined.```

To assign the namespace table region, you cannot use the shell. If you use the shell, it will fail with a `PleaseHoldException` because the master is not yet up (it is waiting for the namepace table to come online before it declares itself ‘up’). You have to use the `HBCK2` _assigns_ command. To assign, you will need the namespace encoded name. It shows in the log quoted above: i.e. _725a0fe6c2c869d3d0a9ed82bfa80fa3_ in this case. You will also have to pass the -skip command to ‘skip’ the master version check (without it, your `HBCK2` invocation will also elicit the above `PleaseHoldException` because the master is not yet up). Here is an example adding an assign of the namespace table:
```$ HBASE_CLASSPATH_PREFIX=~/checkouts/hbase-operator-tools/hbase-hbck2/target/hbase-hbck2-1.0.0-SNAPSHOT.jar ./bin/hbase org.apache.hbase.HBCK2 -skip assigns 725a0fe6c2c869d3d0a9ed82bfa80fa3```

If the invocation comes back with ‘Connection refused’, is the Master up? The Master will shut down after a while if it can’t initialize itself. Just restart the cluster/master and rerun the above assigns command.

When the assigns runs successfully, you’ll see it emit the likes of the following. The ‘48’ on the end is the pid of the assign procedure schedule. If the pid returned is ‘-1’, then the  master startup has not progressed sufficently… retry. Or, the encoded regionname is incorrect. Check.
{{{$  HBASE_CLASSPATH_PREFIX=~/checkouts/hbase-operator-tools/hbase-hbck2/target/hbase-hbck2-1.0.0-SNAPSHOT.jar ./bin/hbase org.apache.hbase.HBCK2 -skip assigns 725a0fe6c2c869d3d0a9ed82bfa80fa3
18:40:43.817 [main] WARN  org.apache.hadoop.util.NativeCodeLoader - Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
18:40:44.315 [main] INFO  org.apache.hbase.HBCK2 - hbck support check skipped
[48]}}}

Check the master logs. The master should have come up. You’ll see successful completion of pid=48. Look for a line like this to verify successful master launch:
```master.HMaster: Master has completed initialization 132.515sec``` It might take a while to appear.

The rebuild of _hbase:meta_ adds the user tables in _DISABLED_ state and the regions in _CLOSED_ mode. Reenable tables via the shell to bring all table regions back online.
Do it one-at-a-time or see the `enable_all ".*"` command to enable all tables in one shot.

The rebuild meta will likely be missing edits and may need subsequent repair and cleaning using facility outlined higher up in this README.

### Dropped reference files, missing hbase.version file, and corrupted hfiles

HBCK2 can check for hanging references and corrupt hfiles. You can ask it to sideline bad files which may be needed to get over humps where regions won't online or reads are failing. See the _filesystem_ command in the HBCK2 listing. Pass one or more tablename (or 'none' to check all tables). It will report bad files. Pass the _--fix_ option to effect repairs.

### Adopting 'Orphan' Data
For how to fix `orphan` regions reported by the 'HBCK Chore',
see the advanced section on the `completebulkload` tool in the refguide,
['Adopting' Stray Data](http://hbase.apache.org/book.html#arch.bulk.load.complete.strays).
