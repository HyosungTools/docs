# Cash Accept

What we want to do is, looking at the CIM XFS Spec, cherry pick some of the CIM INFO, EXEC and EVENT from the SP logs, so we can tell the CIM story in an easily understandable form. For this review I am looking at the NHCashCheckAcceptor330.pdf but any CIM XFS spec will do, they are essentially all the same. 

Per the spec the CIM interface the WFS_SERVICE_CLASS_CIM is '13'. It supports INFO/EXECUTE commands and there are also EVENTs; together they tell the complete story. All we have to do is find them and present them in an easily readable for. 

## Header vs Payload

In the logs the output of commands have both a header and a payload. The header looks like this: 

```
lpResult =
{
	hWnd = [0x0001005c],
	RequestID = [16473],
	hService = [17],
	tsTimestamp = [2023/01/16 15:53 31.126],
	hResult = [0],
	u.dwCommandCode = [1303],
	lpBuffer = [0x08f5d52c]
```
They payload follows. The header is pretty standard across all commands; you can cherry pick the same fields (tsTimestamp, hResult, etc). The payload differs from command to command. 

We want to pull the `tsTimestamp` and the `hResult` from the header. If we are going to report non-zero `hResult` values we should also have a comment column for an English explanation. 

---
## CIM INFO Commands
---

Looking at the INFO commands, we are only interested in commands pertaining to customer transactions. We want to tell enough of the story to answer most of the questions that come up. For the first cut we won't try to represent the full story because the explanation gets too busy. Its a judgement call. 

### WFS_INF_CIM_STATUS (1301)

This command is used to obtain the status of the CIM. 

In the nwlog files, a WFS_INF_CIM_STATUS line is one containing `CATEGORY[1301]`, `WFS_GETINFO_COMPLETE` and `fwDevice`.

The payload is an `lpStatus`. 

```
	{
		fwDevice = [0],
		fwSafeDoor = [1],
        . . .
```

The sort of information we can harvest: 

`fwDevice`      - the state of the device (online, offline, hwerror etc)
`fwSafeDoor`    - open, closed unknown

For the Hyosung SPs we can get `lpszExtra` items such as:

`ErrorCode`     - hardware error code
`Description`   - description of the error code
`EP_Version`    - version of firmware in use
`SP_Version`    - version of SP executable file
`CCIMType`      - the type of CCIM device

There is a list of positions too. In the logs I see 2 positions listed (In/OUT). Not sure we want to capture that information right now. 

Only generic error messages are raised by this command. 

### WFS_INF_CIM_CASH_UNIT_INFO (1303)

This command is use to obtain information about the status and contents of the cash units and recycler units in the CIM. This information changes over time. 

In the nwlog files, a line that contains `CATEGORY[1303]`, `WFS_GETINFO_COMPLETE` and then `lppCashIn` is a line that contains CIM status info.

Reviewing the logs, there appears to be 2 formats to the payload. One looks like this: 

```
	{
		usCount = [7],
		lppCashIn =
		{
			usNumber = [1],
			fwType = [4],
			fwItemType = [0x0001],
			cUnitID = [LCU00],
			cCurrencyID = [   ],
			ulValues = [0],
```

and the other looks like this: 

```
	{
		usCount=2
		lppCashIn->
		{
			usNumber		1		2
			fwType			4		2
			fwItemType		0x0001		0x0001
			cUnitID			LCU00		LCU01
			cCurrencyID		USD		USD
			ulValues		0		0

```
I think the first stype is for the CCIM and the second style is for the BRM. In both cases a lot of the same inforamtion is present, but the layout is different so they have to be handled differently. 

### WFS_INF_CIM_BANKNOTE_TYPES (1306)

Bank note types that can be detected by the banknote reader. The idea here is that over time the treasury updates what it defines as a given note (e.g. 5 USD) because security features are added and notes evolve over time. For example: usNoteID 3, 8, 13 all refer to $ 5 USD. Some commands refer to bank note types. We want to capture this set so we can translate. 

For the first cut I dont think we will do anything with this command. Instead we will pre-configure the mapping into code. Its the quickest solution. 

### WFS_INF_CIM_CASH_IN_STATUS (1307)

Information about the status of the currently active cash-in transaction or in the case where no cash in transaction is active, the status of the most recently ended cash in transaction. 

In the nwLog files, a line that contains `CATEGORY[1307]` and `WFS_GETINFO_COMPLETE` and then `wStatus` is a line that contains CashIn Status:

```
{
    wStatus = [2],
    usNumOfRefused = [0],
    lpNoteNumberList = 
    {
        usNumOfNoteNumbers = [0],
        lppNoteNumber = NULL
    }
    lpszExtra = NULL
```

Good for indicating current state of the CashIn transaction or how it ended. 

---
## CIM EXECUTE Commands
---

Looking at the EXECUTE commands, we are only interested in commands pertaining to customer transactions. We want to tell enough of the story to answer most of the questions that come up. We won't try to represent the full story because the explanation gets too busy. Its a judgement call. 

### WFS_CMD_CIM_CASH_IN_START (1301)

The start of a Cashin transaction. If were want to track CashIn transactions, this is where is starts. During the CashIn any number of WFS_CMD_CIM_CASH_IN commands can be issued; the transation ends when either a WFS_CMD_CIM_CASH_IN_ROLLBACK, WFS_CMD_CIM_CASH_IN_END, WFS_CMD_CIM_RETRACT or WFS_CMD_CIM_RESET is called. 

In the nwlogs, the line is identified as containing `dwCommand = [1301]`, `usTellerID` and `bUseRecycleUnits`. 

The payload is a `lpCashInStart` or pointer to a `WFSCIMCASHINSTART`:

```
{
    usTellerID = [0],
    bUseRecycleUnits = [0],
    fwOutputPosition = [512],
    fwInputPosition = [4]
}
```

I dont think we want to pull any information from the payload, just know that a CASH_IN has started. 

### WFS_CMD_CIM_CASH_IN (1302)

This command moves items into the CIM from an input position. 

In the nwlogs, the line is identified as containing `COMMAND[1302]`, `WFS_EXECUTE_COMPLETE` and `usNumOfNoteNumbers`. 

The payload is a `lpNoteNumberList` or pointer to a `WFSCIMNOTENUMBERLIST`:

```
usNumOfNoteNumbers = [2],
lppNoteNumber = 
{
    usNoteID = [15],
    ulCount = [20]
}
{
    usNoteID = [17],
    ulCount = [1]
} 
```

We already have some media movements. The above tells us we moved 20 x $20 and 1 x $100 to the input position. 

### WFS_CMD_CIM_CASH_IN_END (1303)

This command ends a CASH IN Transaction. 

In the nwlogs, the line is identified as containing `COMMAND[1303]`, `WFS_EXECUTE_COMPLETE` and `lppCashIn`. 

The payload is a `lpCashInfo` or pointer to a `WFSCIMCASHINFO`:

```
{
    usCount=1
    lppCashIn->
    {
        usNumber		5
        fwType			1
        fwItemType		0x0004
        cUnitID			LCU04
        cCurrencyID		USD
        ulValues		20
        ulCashInCount		2
        ulCount			2
        ulMaximum		0
        usStatus		0
        bAppLock		0
```


### WFS_CMD_CIM_CASH_IN_ROLLBACK (1304)

This command is used to roll back a cash-in transaction. It causes all the cash items cashed since the last CASH_IN_START to be returned to the customer. 

In the nwlogs, the line is identified as containing `COMMAND[1304]`, `WFS_EXECUTE_COMPLETE` and `lpBuffer`. 

I dont see a payload, just a [header](#Header_vs_Payload). 

### WFS_CMD_CIM_RETRACT (1305)

Retract items from the Output position. 

In the nwlogs, the line is identified as containing `COMMAND[1305]`, `WFS_EXECUTE_COMPLETE` and `lpBuffer`. 

The payload looks like this:

```
{
    usCount = [1],
    lppCashIn =
    {
```

The hResult of the header matters. The first one I looked at had `hResult = [-1316]`. Looking at the spec, -1316 means `WFS_ERR_CIM_NOITEMS`. That's something we want to report on. 

### WFS_CMD_CIM_RESET (1313)

This command is used by the application to perform a hardware reset which will attempt to return the CIM device to a known good state.

In the nwlogs, the line is identified as containing `COMMAND[1313]`, `WFS_EXECUTE_COMPLETE` and `lpBuffer`. 

I dont see a payload, just a [header](#Header_vs_Payload). 

---
## CIM Events
---

### WFS_USRE_CIM_CASHUNITTHRESHOLD (1303)

Generated when a threshold condition has occurred in one of the cash units.

This event can be triggered either by hardware sensors in the device or by the logical ulCount reaching the ulMaximum value as specified in the WFSCIMCASHIN structure.

In the nwlogs, the line is identified as containing `USER_EVENT[1303]`, `WFS_USER_EVENT` and `usNumber`.

The payload is `lpCashUnit` or a pointer to a `WFSCIMCASHIN`. 

```
	{
		usNumber = [3],
		fwType = [1],
		fwItemType = [0x0004],
		cUnitID = [LCU02],
		cCurrencyID = [USD],
		ulValues = [1],
		ulCashInCount = [0],
		ulCount = [2000],
```

### WFS_SRVE_CIM_CASHUNITINFOCHANGED (1304)

Generated when:

- the status of usStatus and/or usPStatus changes
- for every cash unit changed in any way (e.g. ulCount, ulRejectCount, ulInitialCount, ulDispensedCount and ulPresentedCount)

In the nwlogs, the line is identified as containing `SERVICE_EVENT[1304]`, `WFS_SERVICE_EVENT` and `usNumber`.

The payload is `lpCashUnit` or a pointer to a `WFSCIMCASHIN`.

### WFS_SRVE_CIM_ITEMSTAKEN (1307)

This service event specifies that items presented to the user have been taken.

In the nwlogs, the line is identified as containing `SERVICE_EVENT[1307]` and `WFS_SERVICE_EVENT`.

There is no payload.

### WFS_EXEE_CIM_INPUTREFUSE (1309)

This execute event specifies that the device has refused either a portion or the entire amount of the cash-in order.

In the nwlogs, the line is identified as containing `SERVICE_EVENT[1309]`, `WFS_SERVICE_EVENT` and `usReason`.

The payload is a reason for refusal: 

```
	{
		usReason = [8]
	}
```
We will want to report the reason in plain English. 

### WFS_SRVE_CIM_ITEMSPRESENTED (1310)

This service event specifies that items have been presented to the output position, and the shutter has been opened to allow the user to take the items.

In the nwlogs, the line is identified as containing `SERVICE_EVENT[1310]`, `WFS_SERVICE_EVENT` and `wPosition`.

The payload is a `lpPositionInfo`. For example:

	{
		wPosition = [4]
		wAdditionalBunches = [1]
		usBunchesRemaining = [0]
	}


### WFS_SRVE_CIM_ITEMSINSERTED (1311)

This service event specifies that items have been inserted into the cash-in position by the user.

In the nwlogs, the line is identified as containing `SERVICE_EVENT[1311]` and `WFS_SERVICE_EVENT`.

The payload is supposed to be a `lpPositionInfo`, but I'm not seeing it in our logs. 

### WFS_EXEE_CIM_NOTEERROR (1312)

This execute event specifies the reason for an item detection error during an operation which involves moving items. 

No examples in our logs. 

### WFS_SRVE_CIM_MEDIADETECTED (1314)

This service event is generated if media is detected during a reset (WFS_CMD_CIM_RESET command).

No examples in our logs. 

---

## TABLES

---

The strategy is to create an TABLES and populate it with data from all the INFO, EXEC and EVENTs happening, in chronological order. This builds up a model of how the ATM/CIM changed over time. Then report in spreadsheet form so its understandable. 

Note we can create these tables up front - as XML - and read them in. We dont have to generate them in code. 

Also - line by line - only report changed values. Its change people want to see. If you report every value every time it becomes unreadable; you bury the information. 

---
### CIM Status Table
---

Status of the CIM device over time. I've mulled the ideal of rolling all devices into 1 table but I'm leaning toward keeping all the interfaces separate for the first pass. 

| Column | Description |
| ----------- | ----------- |
| log file | log file that the entry came from |
| timestamp | timestamp of the entry |
| error | any non-zero HResult | 
| CIM | numeric device status |
| CIM/SAFE | numeric safe status | 
| comment | English interpretation of error, CIM and CIM/SAFE status |

Populated: 
- On receipt of [WFS_INF_CIM_STATUS](#WFS_INF_CIM_STATUS-(1301))

---
### CashIn Transaction Table
---
The Cashin Transaction Table logs the lifecylce of a CashIn Transaction. This table should answer questions about what $ was deposited, and where it all went, line by line. 

The table is populated by a lot of messages. Not every message field maps to a column. Use the comment field to give the reader additional information (e.g. the reason for an INPUT REFUSE). 

The Cashin Transaction Table will consist of these columns:

| Column | Description |
| ----------- | ----------- |
| log file | log file that the entry came from |
| timestamp | timestamp of the entry |
| error | any non-zero HResult | 
| wStatus | START, END, ROLLBACK, ACTIVE, RETRACT, UNKN, RESET, REFUSE  | 
| usNumOfRefused | number of items refused | 
| position | INPUT, OUTPUT, CUSTOMER |
| $1 | number of $1 bills | 
| $2 | number of $2 bills |
| $5 | number of $5 bills |
| $10 | number of $10 bills |
| $20 | number of $20 bills |
| $50 | number of $50 bills |
| $100 |number of $100 bills |
| comment | translation of error code, wStatus | 

It will be populated by these messages: 

- On receipt of (INFO): 
    - [WFS_INF_CIM_CASH_IN_STATUS](#WFS_INF_CIM_CASH_UNIT_INFO-(1307))
        - set wStatus to one of END, ROLLBACK, ACTIVE, RETRACT, UNKN, RESET
        - set usNumofRefused
        - record bills received by denomination
        - set comment to (based on wStatus): 
            - the cash-in transaction ended OK
            - the cash-in transaction ended with a ROLLBACK
            - the cash-in transaction is active
            - the cash-in transaction ended with a RETRACT
            - state unknown
            - the cash-in transaction ended with a RESET
- On receipt of (EXECUTE): 
    - [WFS_CMD_CIM_CASH_IN_START](#WFS_CMD_CIM_CASH_IN_START-(1301))
        - set wStatus to START
        - set position to CUSTOMER
        - set comment to "starting cash in transaction"
    - [WFS_CMD_CIM_CASH_IN](#WFS_CMD_CIM_CASH_IN-(1302))
        - set position to INPUT
        - record bills recieved by denomination
    - [WFS_CMD_CIM_CASH_IN_END](#WFS_CMD_CIM_CASH_IN_END-(1303))
        - set comment to "ending cash-in transaction"
- On receipt of [WFS_CMD_CIM_CASH_IN_ROLLBACK](#WFS_CMD_CIM_CASH_IN_ROLLBACK-(1304))
    - set comment to "returning all items to the customer"
- On receipt of [WFS_CMD_CIM_RETRACT](#WFS_CMD_CIM_RETRACT-(1305))
    - set comment to "retracting items from the OUTPUT position"
- On receipt of [WFS_CMD_CIM_RESET](#WFS_CMD_CIM_RESET-(1313))
    - set comment to "performing a harware reset"
- on receipt of (EVENT):
    - [WFS_SRVE_CIM_ITEMSTAKEN](#WFS_SRVE_CIM_ITEMSTAKEN-(1307))
        - set position to CUSTOMER
    - [WFS_EXEE_CIM_INPUTREFUSE](#WFS_EXEE_CIM_INPUTREFUSE-(1309))
        - set wStatus to REFUSE
        - set comment to English equivalent of uReason
    - [WFS_SRVE_CIM_ITEMSPRESENTED](#WFS_SRVE_CIM_ITEMSPRESENTED-(1310))
        - set position to OUTPUT
        - set comment to English equivalent of wAdditionalBunches

---
### Logical Cash Unit Table
---

We can have 1 table for all logical cash units. We can put all entries in the same table because we can select by usNumber. It means some field values (`timestamp`, `error`, `comment`) will be doubled. 

 In reporting we will have 1 worksheet per logical cash unit. The timstamp allows us to produce a time-series - how the cash unit changed over time. 

| Column | Description |
| ----------- | ----------- |
| log file | log file that the entry came from |
| timestamp | timestamp of the entry |
| error | any non-zero HResult | 
| usNumber | index number of the cash unit structure |
| fwType | type of cash unit (in English truncated e.g. RECYCL) | 
| cUnitId | the cash unit identifier | 
| cCurrencyID | 3 character currency identifier (possibly 3 spaces) |
| ulValues | the value of a single item in the cash unit |  
| ulCashInCount | count of items that have entered the  | 
| ulCount | the meaning depends on the type of cash unit | 
| ulMaximum | threshold for status 'high' |
| usStatus | status of the cash unit (in English truncated e.g. OK, FULL, HIGH) | 
| $1 | number of $1 bills | 
| $2 | number of $2 bills |
| $5 | number of $5 bills |
| $10 | number of $10 bills |
| $20 | number of $20 bills |
| $50 | number of $50 bills |
| $100 |number of $100 bills |
| comment | if we have an error field, we need a comment field | 

Populated on: 
- On receipt of [WFS_INF_CIM_CASH_UNIT_INFO](#WFS_INF_CIM_CASH_UNIT_INFO-(1303))

We probably want to convert the field `fwType` and `usStatus` to something English, possibly on-store so the on-write is a lot easier. 

There is a `lpNoteNumberList` that contains note by type. Currently there are 18 entries but they still fold back into the following denominations: 1, 2, 5, 10, 20, 50 and 100. I think we should dom some in line process to get a sum of notes for the row we want to add. 

For Output we probably want to have 1 worksheet per logical cash unit. 



---
## CLASSES
---

Support classes. 

### Discriminates

--- 

Discriminates are configured classes where if you throw a log line at it, it returns with a tuple - found and line-type. There is one discriminate for each line INFO, EXEC and EVENT line we want to identify. 

### BankNoteType

---

We need a class that can convert a note list to a demonination list. The note type list comes from [BANKNOTE TYPEs](#WFS_INF_CIM_BANKNOTE_TYPES-(1306)) but for expedience we should load it as configuration. 

This is the note / currency / value setting. 

| usNoteID | cCurrencyID | ulValues | usRelease | bConfigured | 
| ----------- | ----------- | ----------- | ----------- | ----------- |
| 1 | USD | 1 | 0 | 1 |
| 2 | USD | 2 | 0 | 1 |
| 3 | USD | 5 | 0 | 1 |
| 4 | USD | 10 | 0 | 1 |
| 5 | USD | 20 | 0 | 1 |
| 6 | USD | 50 | 0 | 1 |
| 7 | USD | 100 | 0 | 1 |
| 8 | USD | 5 | 0 | 1 |
| 9 | USD | 10 | 0 | 1 |
| 10 | USD | 20 | 0 | 1 |
| 11 | USD | 50 | 0 | 1 |
| 12 | USD | 100 | 0 | 1 |
| 13 | USD | 5 | 0 | 1 |
| 14 | USD | 10 | 0 | 1 |
| 15 | USD | 20 | 0 | 1 |
| 16 | USD | 50 | 0 | 1 |
| 17 | USD | 100 | 0 | 1 |
| 18 | USD |  | 0 | 1 |

We get something like this : 

```
	usNumOfNoteNumbers = [2],
	lppNoteNumber = 
	{
		usNoteID = [15],
		ulCount = [20]
	}
	{
		usNoteID = [17],
		ulCount = [1]
	} 
```

and we need to convert it to this: 

| $1 | $2 | $5 | $10 | $20 | $50 | $100 | 
| -- | -- | -- | --- | --- | --- | ---  |
|    |    |    |     |  20 |     |   1  |



Something meaninful to the end user. 
