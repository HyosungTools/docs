# Cash Dispense

## <a name='TableofContents'></a>Table of Contents

---

<!-- vscode-markdown-toc -->
* [Table of Contents](#TableofContents)
* [How To Find Lines in the nwLogs](#HowToFindLinesinthenwLogs)
* [Header vs Payload](#HeadervsPayload)
* [CDM INFO Commands](#CDMINFOCommands)
  * [WFS_INF_CDM_STATUS (301)](#WFS_INF_CDM_STATUS301)
  * [WFS_INF_CDM_CASH_UNIT_INFO (303)](#WFS_INF_CDM_CASH_UNIT_INFO303)
* [CDM EXEC Commands](#CDMEXECCommands)
  * [WFS_CMD_CDM_DISPENSE (302)](#WFS_CMD_CDM_DISPENSE302)
  * [WFS_CMD_CDM_PRESENT (303)](#WFS_CMD_CDM_PRESENT303)
  * [WFS_CMD_CDM_REJECT (304)](#WFS_CMD_CDM_REJECT304)
  * [WFS_CMD_CASH_IN RETRACT (305)](#WFS_CMD_CASH_INRETRACT305)
  * [WFS_CMD_CDM_RESET (321)](#WFS_CMD_CDM_RESET321)
* [CDM Events](#CDMEvents)
  * [WFS_SRVE_CDM_CASHUNITINFOCHANGED (304)](#WFS_SRVE_CDM_CASHUNITINFOCHANGED304)
  * [WFS_SRVE_CDM_ITEMSTAKEN (309)](#WFS_SRVE_CDM_ITEMSTAKEN309)
* [TABLES](#TABLES)
  * [CDM Status Table](#CDMStatusTable)
  * [Dispense Transaction Table](#DispenseTransactionTable)
  * [Cash Unit Table](#CashUnitTable)

<!-- vscode-markdown-toc-config
	numbering=false
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->

What we want to do is, looking at the CDM XFS Spec, cherry pick some of the INFO, EXEC and EVENT from the SP logs, so we can tell the CDM story in an easily understandable form. For this review I am looking at the NHBRM330.pdf but any CDM XFS spec will do, they are essentially all the same.

---

## <a name='HowToFindLinesinthenwLogs'></a>How To Find Lines in the nwLogs

---

We can use a regular expression search, where we plug in the command offset, to identify the lines we want. For example:

* Use regex `GETINFO.301.*WFS_GETINFO_COMPLETE` to find all `WFS_INF_CIM_STATUS`
* Use regex `EXECUTE.302.*WFS_EXECUTE_COMPLETE` to find all `WFS_CMD_CDM_DISPENSE`
* Use regex `SERVICE_EVENT.309.*WFS_SERVICE_EVENT` to find all `WFS_SRVE_CDM_ITEMSTAKEN`

The lines we are interested in are listed in the TOC.  

---

## <a name='HeadervsPayload'></a>Header vs Payload

---

In the logs the output of commands have both a header and a payload. The header is standard and looks like this:

```
lpResult =
{
 hWnd = [0x0002019c],
 RequestID = [17631],
 hService = [2],
 tsTimestamp = [2023/01/24 00:59 14.763],
 hResult = [0],
 u.dwCommandCode = [301],
 lpBuffer = [0x24ccae44]
```

They payload follows.

We want to pull:  

* `tsTimestamp`
* `hResult`

from the header. If we are going to report non-zero `hResult` values we should also have a comment column for an English explanation.

---

## <a name='CDMINFOCommands'></a>CDM INFO Commands

---
Looking at the INFO commands, we are only interested in commands pertaining to customer transactions. We want to tell enough of the story to answer most of the questions that come up. For the first cut we won't try to represent the full story because the explanation gets too busy. Its a judgement call.

### <a name='WFS_INF_CDM_STATUS301'></a>WFS_INF_CDM_STATUS (301)

This command is used to obtain the status of the CDM.

Use the regex `GETINFO.301.*WFS_GETINFO_COMPLETE` to find all `WFS_INF_CDM_STATUS` log lines.

The payload is an `lpStatus`.

```
 {
  fwDevice = [0],
  fwSafeDoor = [1],
  fwDispenser = [0],
  fwIntermediateStacker = [0],
```

We should consider reporting on the `lppPosition` data because it gives more insight into cash movement; at least position[0].

### <a name='WFS_INF_CDM_CASH_UNIT_INFO303'></a>WFS_INF_CDM_CASH_UNIT_INFO (303)

This command is used to obtain information regarding the status and contents of the cash units in the CDM.

Use the regex `GETINFO.303.*WFS_GETINFO_COMPLETE` to find all `WFS_INF_CDM_CASH_UNIT_INFO` log lines.

Reviewing the logs, there appears to be 2 formats to the payload. One looks like this:

```
  lppList->
  {
   usNumber  1  2  3  4  5  6
   usType   6  2  12  12  12  12
   lpszCashUnitName [LCU00]  [LCU01]  [LCU02]  [LCU03]  [LCU04]  [LCU05]  

```

the other looks like this

```
  lppList =
  {
   usNumber = [1],
   usType = [2],
   lpszCashUnitName = NULL,
   cUnitID = [LCU00],
```

I think the first format is the complete lists, the second is an update to one of the cash units.

---

## <a name='CDMEXECCommands'></a>CDM EXEC Commands

---

### <a name='WFS_CMD_CDM_DISPENSE302'></a>WFS_CMD_CDM_DISPENSE (302)

This command performs the dispensing of items to the customer.

Use the regex `EXECUTE.302.*WFS_EXECUTE_COMPLETE` to find all `WFS_CMD_CDM_DISPENSE` log lines.

The payload looks like this:

```
 {
  usTellerID = [0],
  usMixNumber = [0],
  fwPosition = [0],
  bPresent = [0],
  lpDenomination =
  {
   cCurrencyID = [USD],
   ulAmount = [0],
   usCount = [6],
   lpulValues = [0, 0, 0, 0, 1, 0],
   ulCashBox = [0]
  }
 }
```

If `bPresent` is true, items are presented to the CUSTOMER as part of the dispense. We may want to comment on that.

The `lpDenomination` is what we want to dispense.

* `cCurrencyID` - ID of the currency in ISO format
* `ulAmout` - amount to dispense
* `usCount` - the size of the lpulValues list
* `lpulValues` - items to be taken from each cash unit.

When we report this we really want to translate that to something like:

| $1 | $2 | $5 | $10 | $20 | $50 | $100 |
|----|----|----|-----|-----|-----|------|
|    |    |    |     | 1   |     |      |

So right away they know we are trying to dispense 1 x \$20.

The other format I see in the logs is just the `lpDenomination`.

```
 {
  cCurrencyID = [USD],
  ulAmount = [20],
  usCount = [6],
  lpulValues = [0, 0, 0, 0, 1, 0],
  ulCashBox = [0]
 }
```

And this time the `ulAmount` is present.

### <a name='WFS_CMD_CDM_PRESENT303'></a>WFS_CMD_CDM_PRESENT (303)

This command will move items to the exit position for removal by the user.

Use the regex `EXECUTE.303.*WFS_EXECUTE_COMPLETE` to find all `WFS_CMD_CDM_PRESENT` log lines.

There is no payload, so look for hResult.

### <a name='WFS_CMD_CDM_REJECT304'></a>WFS_CMD_CDM_REJECT (304)

This command will move items from the intermediate stacker and transport them to a reject cash unit (i.e. a cash unit with usType WFS_CDM_TYPEREJECTCASSETTE).

Use the regex `EXECUTE.304.*WFS_EXECUTE_COMPLETE` to find all `WFS_CMD_CDM_REJECT` log lines.

There is no payload, so look for hResult.

### <a name='WFS_CMD_CDM_RETRACT305'></a>WFS_CMD_CDM_RETRACT (305)

This command will retract items which may have been in customer access from an output position or from internal areas within the CDM.

Use the regex `EXECUTE.305.*WFS_EXECUTE_COMPLETE` to find all `WFS_CMD_CASH_IN_RETRACT` log lines.

In the logs I'm looking at I dont see a payload. Some devices are capable of scanning returned items. For now lets consider this downstream.

### <a name='WFS_CMD_CDM_RESET321'></a>WFS_CMD_CDM_RESET (321)

A command by the application to perform a hardware reset.

Use the regex `EXECUTE.321.*WFS_EXECUTE_COMPLETE` to find all `WFS_CMD_CDM_RESET` log lines.

Looking through the logs, I dont see any payload.

---

## <a name='CDMEvents'></a>CDM Events

---

I searched the logs and the only events I could find were:

* WFS_SRVE_CDM_CASHUNITINFOCHANGED (304)
* WFS_SRVE_CDM_ITEMSTAKEN (309)

So for now let's just look at those two.

---

### <a name='WFS_SRVE_CDM_CASHUNITINFOCHANGED304'></a>WFS_SRVE_CDM_CASHUNITINFOCHANGED (304)

Generated on cash unit change.

Use the regex `SERVICE_EVENT.304.*WFS_SERVICE_EVENT` to find all `WFS_SRVE_CDM_CASHUNITINFOCHANGED` log lines.

The payload is an `lpCashUnit`

```
 {
  usNumber = [5],
  usType = [12],
  lpszCashUnitName = [LCU04],
  cUnitID = [LCU04],
  cCurrencyID = [USD],
  ulValues = [20],
```

### <a name='WFS_SRVE_CDM_ITEMSTAKEN309'></a>WFS_SRVE_CDM_ITEMSTAKEN (309)

This service event is generated when items presented to the user have been taken. This event may be generated at any time.

Use the regex `SERVICE_EVENT.309.*WFS_SERVICE_EVENT` to find all `WFS_SRVE_CDM_CASHUNITINFOCHANGED` log lines.

There is no payload.

---

## <a name='TABLES'></a>TABLES

---
The strategy is to create TABLES and populate them with data from all the INFO, EXEC and EVENTs in chronological order. This builds up a model of how the ATM/CDM changed over time. Then report in spreadsheet form so its understandable. We will create 3 tables:

* CDM Status Table
* Dispense Transaction Table
* Logical Cash Unit

We can create these tables at instantiation time - load empty tables as XML. We dont have to generate them in code. Also when adding a line only report changed values. Its change over time people want to see. If you report every value every time it becomes unreadable; you bury the information.

### <a name='CDMStatusTable'></a>CDM Status Table

---

Status of the CDM device over time. I've mulled the ideal of rolling all devices into 1 table but I'm leaning toward keeping all the interfaces separate for the first pass.

The Worksheet will be called CDM Status.

The CDM Status Table will look like this:

| Column    | Source                | Description                           |
|-----------|-----------------------|---------------------------------------|
| file      | log file              | log file that the entry came from     |
| time      | tsTimestamp           | timestamp of the entry                |
| error     | hResult               | any non-zero HResult                  |
| status    | fwDevice              | in truncated English                  |
| safe      | fwSafeDoor            | in truncated English                  |
| dispenser | fwDispenser           | in truncated English                  |
| intstack  | fwIntermediateStacker | in truncated English                  |
| shutter   | fwShutter             | from position[0] in truncated English |
| posstatus | fwPositionStatus      | from position[0] in truncated English |
| transport | fwTransport           | from position[0] in truncated English |
| transstat | fwTransportStatus     | from position[0] in truncated English |
| position  | wDevicePosition       | in truncated English                  |
| comment   | various               | add context for reader                |

Populated:

* On receipt of [WFS_INF_CDM_STATUS](#WFS_INF_CDM_STATUS1301)

We may have to add documentation to better explain the fields, and what the truncated English means; that can be a doc on the github repo.  

I think there's valuable information in position[0] we should report.

### <a name='DispenseTransactionTable'></a>Dispense Transaction Table

---
The Dispense Transaction Table logs the lifecylce of a Dispense Transaction. This table should answer questions about what $ was dispensed, and where it all went, line by line.

The table is populated by a lot of messages. Not every message field maps to a column. Use the comment field to give the reader additional information.

The Cashin Transaction Table will consist of these columns:

| Column   | Source      | Description                                    |
|----------|-------------|------------------------------------------------|
| file     | log file    | log file that the entry came from              |
| time     | tsTimestamp | timestamp of the entry                         |
| error    | hResult     | any non-zero HResult                           |
| position | various     | OUT, STACKER, REJECT, RETRACT, CUSTOMER, RESET |
| amount   | ulAmount    | WFS_CMD_CDM_DISPENSE                           |
| $1       | lpulValues  | number of $1 bills                             |
| $2       | lpulValues  | number of $2 bills                             |
| $5       | lpulValues  | number of $5 bills                             |
| $10      | lpulValues  | number of $10 bills                            |
| $20      | lpulValues  | number of $20 bills                            |
| $50      | lpulValues  | number of $50 bills                            |
| $100     | lpulValues  | number of $100 bills                           |
| comment  | various     | translation of error code, wStatus             |

It will be populated by these messages:

* On receipt of (EXEC):
  * [WFS_CMD_CDM_DISPENSE](WFS_CMD_CDM_DISPENSE302)
    * set position to OUT or STACKER based on bPresent
    * record bills dispensed
  * [WFS_CMD_CDM_PRESENT](WFS_CMD_CDM_PRESENT303)
    * set position to OUT
    * set comment to notes presented to customer
  * [WFS_CMD_CDM_REJECT](WFS_CMD_CDM_REJECT304)
    * set position to REJECT
    * set comment to all notes retained
  * [WFS_CMD_CDM_RETRACT](WFS_CMD_CDM_RETRACT305)
    * set position to RETRACT
    * set comment to unknown notes retracted
  * [WFS_SRVE_CDM_ITEMSTAKEN](WFS_SRVE_CDM_ITEMSTAKEN309)
    * set position to CUSTOMER
    * set comment to customer has taken the money
  * [WFS_CMD_CDM_RESET](WFS_CMD_CDM_RESET)
    * set position to RESET
    * set comment to application issues hardware reset

### <a name='CashUnitSummary'></a>Cash Unit Summary

---

There are some values that never change, or change rarely,  during a days run of the ATM. It makes sense to peel these off into a separate table.

| Column   | Description                                                       |
|----------|-------------------------------------------------------------------|
| file     | log file that the entry came from                                 |
| time     | timestamp of the entry                                            |
| error    | any non-zero HResult                                              |
| number   | usNumber (index) of the cash unit structure                       |
| type     | usType of cash unit (in English truncated e.g. RECYCL)            |
| name     | lpszCashUnitName the cash unit identifier                         |
| unit     | cUnitId[5] - the cash unit identifier                             |
| currency | cCurrencyID - 3 character currency identifier (possibly 3 spaces) |
| demon    | ulValues - the value of a single item in the cash unit            |
| initial  | ulInitialCount of items that have entered the                     |
| min      | ulMinumum - trigger for WFS_CDM_STATCULOW                         |
| max      | ulMaximum - threshold for status 'high'                           |
| comment  | if we have an error field, we need a comment field                |

### <a name='CashUnitTable'></a>Cash Unit Table

---

There are some values that change rapidly.

| Column    | Description                                                                         |
|-----------|-------------------------------------------------------------------------------------|
| file      | log file that the entry came from                                                   |
| time      | timestamp of the entry                                                              |
| error     | any non-zero HResult                                                                |
| number    | usNumber - index number of the cash unit structure                                  |
| count     | ulCount - the meaning depends on the type of cash unit                              |
| reject    | usNumber - the number of items in the reject bin                                    |
| status    | usStatus of the cash unit (in English truncated e.g. OK, FULL, HIGH)                |
| dispensed | ulDispensedCount - the number of items dispensed from cash unit                     |
| presented | ulPresentedCount - the number of items from this cash unit presented                |
| retracted | ulRetractedCount - the number of items presented then retracted from this cash unit |
| comment   | if we have an error field, we need a comment field                                  |

Populated on receipt of:

* [WFS_INF_CDM_CASH_UNIT_INFO](#WFS_INF_CDM_CASH_UNIT_INFO303)
* [WFS_SRVE_CDM_CASHUNITINFOCHANGED](#WFS_SRVE_CDM_CASHUNITINFOCHANGED304)

We probably want to convert the field `usType` and `usStatus` to something English.
