# 112, TIGA 

# Missing EFM Retrieval Guide 

## The Integration Group of Americas (TIGA), A Tetra Tech Company

|   |   |
|---|---|
|**Purpose of this document**|The purpose of this instructional document is to provide clear and comprehensive guidance to users on effectively utilizing products or processes in order to enhance their proficiency, optimize outcomes, and ensure a consistent experience.|
|**Audience**|Technical audiences including new users, experienced practitioners, and anyone seeking detailed instructions and insight into specific products or processes.|


# 1 DOCUMENT INFORMATION

## 1.1 VERSION CONTROL

This document has the following change history:

|                    |           |              |                        |
| ------------------ | --------- | ------------ | ---------------------- |
| **Version Letter** | **Date**  | **Author**   | **Change Description** |
| **A**              | 9.12.2024 | Omar Mokrech | Document Creation      |
|                    |           |              |                        |

# 2 OVERVIEW

Retrieving missing Electronic Flow Measurement (EFM) data through Autosol requires device command triggers. Each device type will require different command triggers to successfully retrieve EFM data. Once device commands are triggered, devices will provide missing data respective to the requested date. Engineers will need to establish publishers and databases within Autosol prior to triggering commands to successfully retrieve missing data. For information on publishers and databases, reference 6 PUBLISHER & DATABASE SETUP.

_Note: As with most devices EFM data can only be collected up to 30 days in the past. Devices typically delete internal data past the 30–35-day point. The earliest data will be deleted and replaced with the newest day data once past the designated data save date._ 

# 3 COMMANDS

Current efforts have determined the required commands for Emmerson ROCs and TotalFlow device missing EFM data retrieval. If Autosol is leveraging a different device type, contact Autosol support to determine the appropriate commands needed.

Emmerson ROCs

|   |   |
|---|---|
|**Command (In order)**|**Command Function**|
|Get Last Hourly Record|Checks the last missing record and calculates how many hourly rows to go back.|
|Move the Hourly Record Pointer|Takes the hourly rows from the previous (above) command and moves the pointer back in the system.|
|Retrieve History|Grabs all historical data from the point to the current time and publishes it to SQL.|

Figure 1 ROCs Flow

Totalflow

|   |   |
|---|---|
|**Command (In order)**|**Command Function**|
|Set Daily Pointer|Sets the daily pointer to the most recent successful data pull prior to missing data.|
|Set Log Pointer|Sets the log pointer to the most recent successfully data pull prior to missing data.|
|Retrieve History|Grabs all historical data from the pointers to the current time and publishes it to SQL.|

_Note: A custom logic is required to identify the log pointer/daily pointer start time. Totalflow devices require a timestamp value to be entered rather than hourly rows for retrieval._


# 4      OPC UA TAG TO COMMAND ASSOCIATION

The above commands exist within the Autosol environment. To trigger commands via Ignition, engineers will need the appropriate OPC UA tag associations to each command. If the commands below do not support current EFM command structures (above), contact Autosol support.  

|   |   |   |   |
|---|---|---|---|
|**Proposed Tag Name**|**UDT**|**OPCitemPath**|**Data**<br><br>**Type**|
|LAST_HOURLY_RECORD|_types_/EMERSONROCL/DEVICE|.LastPollAttemptTimeUTC|DateTime|
|MOVE_HOURLY|_types_/EMERSONROCL/DEVICE|.MoveHourly|Short|
|RETRIEVE_HISTORY|_types_/EMERSONROCL/DEVICE|.RetrieveHistory|Boolean|
|LAST_HOURLY_RECORD|_types_/TOTALFLOWG4/DEVICE|.LastHistoryCollectionUTC|DateTime|
|SET_DAILY_POINTER|_types_/TOTALFLOWG4/DEVICE|.SetDaily/1|DateTime|
|RETRIEVE_HISTORY|_types_/TOTALFLOWG4/DEVICE|.RetrieveHistory|Boolean|
|SET_LOGPERIOD_POINTER|_types_/TOTALFLOWG4/DEVICE|.SetLogPeriod/1|DateTime|

These tags act as communication points between Ignition and Autosol to trigger commands. This overrides the need to manually enter the ACM server to trigger commands. Engineers can have Ignition write to the OPC UA Tag which then writes to Autosol, triggering the command.

Certain commands that cannot be triggered via OPC UA Tags can be leveraged for testing purposes. The Skip Hourly Records command can be used to skip history. Engineers can leverage this for testing purposes to create missing data gaps in order to test EFM retrieval processes. Autosol recommends using this command to simulate missing data gaps that may occur in the field.

_Note: The above OPC UA tags will reoccur anytime Autosol is used in a project. Autosol has **not** exposed all OPC UA tags tied to commands._

# 5      PRUNING PROCESS

The EFM retrieval process will create duplicate records for existing data on devices. A pruning process must be implemented to remove duplicate records from the system. The pruning process should be a named query that runs daily to eliminate duplicate records. Successful pruning will result in one (1) single record for each data device. Pruning is leveraged to mitigate database bloat. Engineers can leverage the following Script and Query for future use cases:

```python
def onScheduledEvent():
	import time
	from java.lang import System
	logger = system.util.getLogger("EFMPruning")
	logger.info("Starting EFM pruning script")
	start = System.nanoTime()
	
	value = system.db.runNamedQuery('EFM/GetMeters')
	meters = str(",".join(item["MeterID"] for item in value))
	logger.info("Meters: %s" %(meters))
	ds = system.db.runNamedQuery('EFM/Pruning/PruneDuplicates', {'meters': meters})
	for i in ds:
	    logger.info("Deleted Records: %d" %(i[0]))
	   
	end = System.nanoTime()
	execTime = str((end - start) / 1e9)
	logger.info("Execution time: %s s" %(execTime))
	logger.info("EFM pruning script completed")
```

Query
```sql
DECLARE @DeletedRecords TABLE (
   MeterID INT,
   RecordTime DATETIME,
   UpdateTime DATETIME
);
WITH CTE AS (
   SELECT 
       MeterID, 
       RecordTime, 
       UpdateTime,
       ROW_NUMBER() OVER (PARTITION BY MeterID, CAST(RecordTime AS DATE), DATEPART(HOUR, RecordTime) ORDER BY UpdateTime DESC) AS rn
   FROM tblAsiEFMHistoryRecords
   WHERE MeterID IN ({meters})
     AND RecordTime >= DATEADD(DAY, -30, GETDATE())
     AND RecordTime <= GETDATE()
)

DELETE t
OUTPUT DELETED.MeterID, DELETED.RecordTime, DELETED.UpdateTime INTO @DeletedRecords
FROM (
   SELECT *
   FROM tblAsiEFMHistoryRecords
   WHERE MeterID IN ({meters})
     AND RecordTime >= DATEADD(DAY, -30, GETDATE())
     AND RecordTime <= GETDATE()
) AS t
WHERE EXISTS (
   SELECT 1
   FROM CTE
   WHERE t.MeterID = CTE.MeterID
     AND t.RecordTime = CTE.RecordTime
     AND t.UpdateTime = CTE.UpdateTime
     AND CTE.rn > 1
);

SELECT COUNT(*) AS DeletedRecords FROM @DeletedRecords;
```

# 6 PUBLISHER & DATABASE SETUP

Autosol Publisher and database connections are required to store all EFM retrieved history. It is recommended to create a new SQL database for EFM tables. Establishing the publisher and database will allow Autosol to create SQL tables for the respective EFM information. Once an EFM database is created, Autosol will start collecting EFM data in the created tables.

_Note: When establishing publishing, engineers should set a specific date for data collection efforts. Without establishing a date, Autosol will default and attempt to collect data from 365 days ago._

Existing EFM data should be collected and stored within a separate database. Missing EFM data means the system lacks EFM information from the generated tables. Engineers can leverage the tables to create supporting metric information for EFM data retrieval screens. For example, Western Midstream development included leverage Alarms and Events to display a total percentage (%) of each retrieved.

Engineers will also need to establish queries that combine EFM tables and send information from the tables to Ignition screens.

# 7 MULTITASKING COMMAND TRIGGERS

Engineers can leverage threading to trigger multiple commands to Autosol at once rather than sequentially. Threading aims to begin all operations concurrently. For each action tasked, the treading query begins a “thread” and then joins all threads together to execute simultaneously, resulting in parallel execution. Engineers can leverage the following script for future use cases:

```python
import time
import threading
from datetime import datetime 

def main(data):
    threads = []
    results = []

    for item in data:
     system.perspective.print(item)
        thread = threading.Thread(target=EFM.process_item(item,results))
        threads.append(thread)
        thread.start()
    system.perspective.print(threads)

    for thread in threads:
        thread.join()  #Wait for all threads to finish
        system.perspective.print('Thread is finished')
return results
  
data = list(self.parent.parent.getChild("HistorySummary").getChild("Table_0").custom.dataRH)
start = time.clock()
results = main(data)
end = time.clock()
etime = end-start
# system.perspective.print(type(data))
# system.perspective.print(data)
system.perspective.print (etime)
# system.perspective.print (type(results))
# system.perspective.print (results)
params = {'arr':results}
self.parent.parent.getChild("FlexContainer_Filters").getChild("missingGaps").props.selected = False
self.view.refreshBinding("custom.historyRecordsPerDay")
self.view.refreshBinding("custom.fhistoryRecordsPerDay")
tbl = self.parent.parent.getChild("HistorySummary").getChild("Table_0")
tbl.refreshBinding("props.data")
system.perspective.openPopup('CS','Views/EFM/Popups/Confirmation_S',params)  #Show the final process results
self.props.value = 0
# self.props.value = 0
```
_Note: Multiprocessing is a possible alternative to threading with Python 3. Ignition currently uses 2.7 resulting in the use of a threading module._

# 8 RECOMMENDATIONS

The following is a list of recommendations when it comes to EFM retrieval. If you have additional recommendations to add, contact Omar Mokrech ([omarmokrech@tetratech.com](mailto:omarmokrech@tetratech.com)) or the current Technical Writer on staff to update this section.

- History retrieval should be limited to 30 days from the retrieval attempt date.
	- Meters usually don’t store history for more than 30 – 35 days. Requests past this may cause errors in the communication manager.
- Systems should limit user choice when selecting retrieval dates in order to mitigate Autosol errors.
- Missing EFM data should be published to a separate SQL server.
