```bash
curl -X POST http://localhost:8090/geography/create-domain \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Cameroon Region",
    "shortName": "CMR",
    "description": "Domain for Cameroon Operations",
    "localCurrency": "XAF",
    "foreignCurrency": "USD",
    "createdBy": 1,
    "isAsync": false
  }'
```
[
  {
    "id": "01M2SKCR8KAM82TCQ52B7KM83D",
    "created_at": "2026-09-18T06:31:20.596Z",
    "created_by": 1,
    "description": "Republic of Congo Region",
    "foreign_currency": "USD",
    "local_currency": "CDF",
    "name": "Republic of Congo",
    "reason": null,
    "short_name": "CG",
    "status": "ACTIVE",
    "updated_at": "2026-09-18T06:31:27.170Z",
    "updated_by": 0,
    "updated_data": null
  }
]

[
  {
    "id": "01M2SXCTWVHDTY5STQFXNNN6A2",
    "created_at": "2026-09-18T09:26:09.052Z",
    "created_by": 1,
    "description": "Domain for Cameroon Operations",
    "foreign_currency": "USD",
    "local_currency": "XAF",
    "name": "Cameroon Region",
    "reason": null,
    "short_name": "CMR",
    "status": "PENDING",
    "updated_at": null,
    "updated_by": 0,
    "updated_data": null
  }
]

{  
  "name": "India",  
  "shortName": "IN",  
  "description": "Indian Region",  
  "localCurrency": "INR",  
  "foreignCurrency": "USD"  
}

{  
  "name": "India",  
  "shortName": "IN",  
  "description": "Indian Region",  
  "localCurrency": "INR",  
  "foreignCurrency": "USD"  
}