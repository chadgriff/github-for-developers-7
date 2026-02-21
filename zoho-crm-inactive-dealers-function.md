# Zoho CRM Function: Mark Dealers (Accounts) Inactive After 12 Months

## Goal
Set `Accounts.Status` to `Inactive` when:
- the most recent related `Sales_Orders.Date` is older than 12 months, **or**
- there are no related sales orders and the dealer record is older than 12 months.

## API Names Used (as requested)
- Dealers module: `Accounts`
- Dealers module Sales Orders related list: `SalesOrders`
- Dealer status field: `Dealer_Status`
- Sales order date field: `Date`

---

## Deluge Scheduled Function (monthly, `for each` loops only)
```deluge
void schedule.markInactiveDealers()
{
//----------------------Determine how many records exist in the Dealers module----------------------------------------------------
count = invokeurl
[
	url :"https://www.zohoapis.com/crm/v6/Accounts/actions/count"
	type :GET
	connection:"zoho_crm_modules_read"
];
//info "count: " + count;
//-------------------Grab Dealer records 200 at a time and update the Status as appropriate---------------------------------------
n = count.get("count") / 200;
n_int = n.ceil();
//info n_int;
x = 0;
paramMap = Map();
headersMap = Map();
counter = leftpad("1",n_int).replaceAll(" ","1,").toList();
for each index i in counter
{
	accountQuery = Map();
	accountMap = Map();
	info "x value: " + x;
	//Collect 200 Contact Records at a Time
	if(x == 0)
	{
		selectQuery = "SELECT id, Account_Name, Dealer_Status, Created_Time FROM Accounts WHERE Dealer_Status = Active LIMIT 0, 200";
	}
	else
	{
		selectQuery = "SELECT id, Account_Name, Dealer_Status, Created_Time FROM Accounts WHERE Dealer_Status = Active LIMIT " + x + ", 200";
	}
	//info "selectQuery: " + selectQuery;
	accountQuery.put("select_query",selectQuery);
	response = invokeurl
	[
		url :"https://www.zohoapis.com/crm/v6/coql"
		type :POST
		parameters:accountQuery.toString()
		connection:"zoho_crm_coql"
	];
	//info "response " + response;
	Account_Record = response.get("data");
	cutoffDate = zoho.currentdate.addMonth(-12);
	for each  item in Account_Record
		{

			dealerStatus = item.get("Dealer_Status");
			if (dealerStatus == "Active")
			{
				//----------------------Determine how many Sales Orders exist in the Related List-----------------------------
				soParam = '{"get_related_records_count": [{ "related_list": {"api_name": "SalesOrders"}}]}';
				soCount = invokeurl
				[
					url :"https://www.zohoapis.com/crm/v8/Accounts/" + item.get("id") + "/actions/get_related_records_count"
					type :POST
					parameters: soParam + ""
					connection:"zoho_crm_modules_read"
				];
				//info "soCount: " + soCount;
				soCountValue = soCount.get("get_related_records_count").get(0).get("count");
				if(soCountValue > 0)
				{
					//info "dealerRecord: " + item;
					//info "soCountValue: " + soCountValue;
					//----------------------Collect Sales Orders--------------------------------------------------------------------------
					soN = soCountValue / 200;
					soN_int = soN.ceil();
					//info n_int;
					soX = 0;
					paramMap = Map();
					headersMap = Map();
					soCounter = leftpad("1",soN_int).replaceAll(" ","1,").toList();
					for each index i in soCounter
					{
						soQuery = Map();
						soMap = Map();
						//info "soX value: " + soX;
						//------------Collect 200 Sales Order Records at a Time---------------------------------------------------------------
						if(soX == 0)
						{
							soSelectQuery = "SELECT id, Account_Name.id, Subject, Date FROM Sales_Orders WHERE Account_Name.id = " + item.get("id") + " LIMIT 0, 200";
						}
						else
						{
							soSelectQuery = "SELECT id, Account_Name.id, Subject, Date FROM Sales_Orders WHERE Account_Name.id = " + item.get("id") + " LIMIT " + x + ", 200";
						}
						//info "soSelectQuery: " + soSelectQuery;
						soQuery.put("select_query",soSelectQuery);
						soResponse = invokeurl
						[
							url :"https://www.zohoapis.com/crm/v6/coql"
							type :POST
							parameters:soQuery + ""
							connection:"zoho_crm_coql"
						];
						salesOrders = soResponse.get("data");
						//info "salesOrders: " + salesOrders;
						latestOrderDate = null;
						//-----------Determine Most Recent Sales Order Date----------------------------------------
						for each so in salesOrders
						{
							soDateVal = so.get("Date");
							info "soDateVal: " + soDateVal;
							if(soDateVal != null)
							{
								soDate = soDateVal.toDate();
								if(latestOrderDate == null || soDate > latestOrderDate)
								{
									latestOrderDate = soDate;
								}
							}
						}
						//--------Determine if most recent sales order date is older than 1 year-----------------
						//info "cutoffDate: " + cutoffDate;
						//info "latestOrderDate: " + latestOrderDate;						
						shouldMarkInactive = false;
                        if(latestOrderDate != null)
						{
							if(latestOrderDate < cutoffDate)
							{
								shouldMarkInactive = true;
							}
						}
						else 
						{
							// No sales orders: evaluate account age
							createdTimeVal = item.get("Created_Time");
							if(createdTimeVal != null)
							{
								createdDate = createdTimeVal.toDate();
								if(createdDate < cutoffDate)
								{
									shouldMarkInactive = true;
								}
							}
						}
						//-----------Update Dealer Record as Inactive------------------------------------------------
						//info "shouldMarkInactive: " + shouldMarkInactive;
						if(shouldMarkInactive == true)
						{
							updateMap = Map();
							updateMap.put("Dealer_Status", "Inactive");
							updateResp = zoho.crm.updateRecord("Accounts", item.get("id"), updateMap);
							info "Marked Account " + item.get("id") + " as Inactive. Response: " + updateResp;
						}
					}
					soX = soX + 200;
				}
			}
		}
		x = x + 200;
	}
}
```

---

## Schedule Setup
1. Go to **Setup → Developer Space → Functions** and save this function.
2. Go to **Setup → Automation → Schedules**.
3. Create a **monthly** schedule and attach this function.
4. Run once in sandbox first.
