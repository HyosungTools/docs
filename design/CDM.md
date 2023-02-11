# Cash Dispense

What we want to do is, looking at the CDM XFS Spec, cherry pick some of the INFO, EXEC and EVENT from the SP logs, so we can tell the CDM story in an easily understandable form. For this review I am looking at the NHBRM330.pdf but any CDM XFS spec will do, they are essentially all the same. 

Per the spec the CDM interface the WFS_SERVICE_CLASS_CIM is '03'. It supports INFO/EXECUTE commands and there are also EVENTs; together they tell the complete story. All we have to do is find them and present them in an easily readable for. 

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
## CDM INFO Commands
---

### WFS_INF_CDM_STATUS (0301)


## WFS_INF_CDM_CASH_UNIT_INFO ()

---
## CDM EXEC Commands
---

## WFS_CMD_CASH_IN_START ()

## WFS_CDM_CASH_IN ()

## WFS_CDM_CASH_IN_END ()

## WFS_CMD_CASH_IN_ROLLBACK ()

## WFS_CMD_CASH_IN RETRACT ()


)
