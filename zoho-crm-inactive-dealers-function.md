# Zoho CRM Function: Mark Dealers (Accounts) Inactive After 12 Months

## Goal
Set `Accounts.Status` to `Inactive` when:
- the most recent related `Sales_Orders.Date` is older than 12 months, **or**
- there are no related sales orders and the dealer record is older than 12 months.

## API Names Used (as requested)
- Dealers module: `Accounts`
- Sales Orders module / related list: `Sales_Orders`
- Dealer status field: `Status`
- Sales order date field: `Date`

---

## Deluge Scheduled Function (monthly, `for each` loops only)
```deluge
void automation.markInactiveDealers()
{
    perPage = 200;
    cutoffDate = zoho.currentdate.addMonth(-12);

    // Define page windows to avoid while-loops (extend if your dataset is larger)
    pageNumbers = "1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50".toList(",");

    hasMoreAccounts = true;

    for each accountPageText in pageNumbers
    {
        if(!hasMoreAccounts)
        {
            continue;
        }

        accountPage = accountPageText.toLong();
        accounts = zoho.crm.getRecords("Accounts", accountPage, perPage);

        if(accounts == null || accounts.isEmpty())
        {
            hasMoreAccounts = false;
            continue;
        }

        for each acc in accounts
        {
            accountId = acc.get("id");
            currentStatus = ifnull(acc.get("Status"), "");
            if(currentStatus == "Inactive")
            {
                continue;
            }

            // Get latest Sales_Orders.Date across related Sales Orders
            latestOrderDate = null;
            hasMoreSalesOrders = true;

            for each soPageText in pageNumbers
            {
                if(!hasMoreSalesOrders)
                {
                    continue;
                }

                soPage = soPageText.toLong();
                salesOrders = zoho.crm.getRelatedRecords("Sales_Orders", "Accounts", accountId, soPage, perPage);

                if(salesOrders == null || salesOrders.isEmpty())
                {
                    hasMoreSalesOrders = false;
                    continue;
                }

                for each so in salesOrders
                {
                    soDateVal = so.get("Date");
                    if(soDateVal != null)
                    {
                        soDate = soDateVal.toDate();
                        if(latestOrderDate == null || soDate > latestOrderDate)
                        {
                            latestOrderDate = soDate;
                        }
                    }
                }

                if(salesOrders.size() < perPage)
                {
                    hasMoreSalesOrders = false;
                }
            }

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
                // NOTE: Created_By is creator user info, not a date.
                createdTimeVal = acc.get("Created_Time");
                if(createdTimeVal != null)
                {
                    createdDate = createdTimeVal.toDate();
                    if(createdDate < cutoffDate)
                    {
                        shouldMarkInactive = true;
                    }
                }
            }

            if(shouldMarkInactive)
            {
                updateMap = Map();
                updateMap.put("Status", "Inactive");

                updateResp = zoho.crm.updateRecord("Accounts", accountId.toLong(), updateMap);
                info "Marked Account " + accountId + " as Inactive. Response: " + updateResp;
            }
        }

        if(accounts.size() < perPage)
        {
            hasMoreAccounts = false;
        }
    }
}
```

---

## Schedule Setup
1. Go to **Setup → Developer Space → Functions** and save this function.
2. Go to **Setup → Automation → Schedules**.
3. Create a **monthly** schedule and attach this function.
4. Run once in sandbox first.

## Important Notes
- This version intentionally uses `for each` loops only (no `while`) for Deluge compatibility.
- You asked to compare `Created_By` for no-order dealers. In Zoho CRM, `Created_By` is typically a user lookup (who created the record), not a date. The script uses `Created_Time` to determine whether the dealer record is older than 12 months.
- If your org can exceed 50 pages of Accounts or related Sales Orders, extend `pageNumbers`.


## How to Update This Repo With Your Working Zoho Version
1. In Zoho CRM Function editor, copy your **working** Deluge code.
2. In this repo, open `zoho-crm-inactive-dealers-function.md` and replace the code block under **Deluge Scheduled Function** with your final code.
3. Save the file.
4. Run:
   - `git status --short`
   - `git add zoho-crm-inactive-dealers-function.md`
   - `git commit -m "Update Deluge function to final working Zoho version"`
5. Push your branch:
   - `git push`
6. Open/update the PR with a note like:
   - "Synced final tested function from Zoho CRM Function editor into repository."

### Quick copy/paste tip
Keep only the function code in the fenced `deluge` block so the repo remains the source of truth for the exact version running in Zoho.
