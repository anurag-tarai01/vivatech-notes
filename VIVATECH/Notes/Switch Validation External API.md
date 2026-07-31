## Request API
postman request POST 'https://api.g-payment.net/switch/api/enterprise/oauth/token' \
  --header 'Content-Type: application/json' \
  --body '{
    "client_id": "GPayTest",
    "client_secret": "9Zl31oy<~}1a",
    "grant_type": "partner",
    "scope": "GP"
}'

## Response :
{
    "message": "That went well",
    "status": 0,
    "access_token": "eyJhbGciOiJIUzI1NiJ9.eyJjb21wYW55TmFtZSI6IkctUGF5IFN3aXRjaCIsImVudGl0eVR5cGVJRCI6MTMsImVudGl0eUlEIjoxNSwic3ViIjoiRy1QYXkgU3dpdGNoLjEzLSYtMTUiLCJpYXQiOjE3ODAzNzIxNzIsImV4cCI6MTc4MDM3NTc3Mn0.VYkeMQ1fg2nyiIyFmPhM2yXkkPkr82O9_nd5tAR3XF4",
    "token_type": "Bearer",
    "expires_in": 3600
}

Error:
{
    "message": "Bad credentials",
    "status": 9001
}

---
## Request : Mobile
postman request POST 'https://api.g-payment.net/switch/api/enterprise/transfer/validation?intent=disburse' \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJjb21wYW55TmFtZSI6IkctUGF5IFN3aXRjaCIsImVudGl0eVR5cGVJRCI6MTMsImVudGl0eUlEIjoxNSwic3ViIjoiRy1QYXkgU3dpdGNoLjEzLSYtMTUiLCJpYXQiOjE3ODAyOTMzNTYsImV4cCI6MTc4MDI5Njk1Nn0.bjn6revrRPoNwNyW1evcycds12SbSAMjjTg7HXy-A40' \
  --body '{
  "account" : {
    "countryCode": "CMR",
    "number" : "677061344",
    "type" : "M",
    "amount":1000,
  "currencyCode" : "XAF"
  }
}' \
  --auth-bearer-token 'eyJhbGciOiJIUzI1NiJ9.eyJjb21wYW55TmFtZSI6IkctUGF5IFN3aXRjaCIsImVudGl0eVR5cGVJRCI6MTMsImVudGl0eUlEIjoxNSwic3ViIjoiRy1QYXkgU3dpdGNoLjEzLSYtMTUiLCJpYXQiOjE3ODAyOTMzNTYsImV4cCI6MTc4MDI5Njk1Nn0.bjn6revrRPoNwNyW1evcycds12SbSAMjjTg7HXy-A40'
## Response
{
    "message": "Excellent",
    "status": 0,
    "name": "N A null DUCAS FUEN EPOUSE WANYEH DUCAS FUEN EPOUSE WANYEH",
    "amount": 1000,
    "expectedFee": 0,
    "amountBreakdown": []
}

Error:
{
    "message": "Please get a valid token and try again",
    "status": 9001
}

---
{
   "message": "Query run but with errors",
    "status": 0,
    "name": "Please specify a valid organisation where this account resides",
    "amount": 1000,
    "expectedFee": 0,
    "amountBreakdown": []
}

---
{
    "message": "So sorry, Please report this error to your IT department",
    "status": 9000
}

---
{
    "message": "The country MR is not recognised",
    "status": 9007
}

---
{
    "message": "Sorry, the minimum to send is 10",
    "status": 9012
}

---
{
    "message": "Query run but with errors",
    "status": 0,
    "name": "Sorry, this service is not yet available in your country",
    "amount": 1000,
    "expectedFee": 0,
    "amountBreakdown": []
}

---
## Request : Bank
postman request POST 'https://api.g-payment.net/switch/api/enterprise/transfer/validation?intent=disburse' \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJjb21wYW55TmFtZSI6IkctUGF5IFN3aXRjaCIsImVudGl0eVR5cGVJRCI6MTMsImVudGl0eUlEIjoxNSwic3ViIjoiRy1QYXkgU3dpdGNoLjEzLSYtMTUiLCJpYXQiOjE3ODAyOTMzNTYsImV4cCI6MTc4MDI5Njk1Nn0.bjn6revrRPoNwNyW1evcycds12SbSAMjjTg7HXy-A40' \
  --body '{
  "account" : {
    "orgCode" : "CCA",
    "countryCode": "CMR",
    "number" : "00856171601",
    "type" : "B",
    "amount":1000,
  "currencyCode" : "XAF"
  }
}' \
  --auth-bearer-token 'eyJhbGciOiJIUzI1NiJ9.eyJjb21wYW55TmFtZSI6IkctUGF5IFN3aXRjaCIsImVudGl0eVR5cGVJRCI6MTMsImVudGl0eUlEIjoxNSwic3ViIjoiRy1QYXkgU3dpdGNoLjEzLSYtMTUiLCJpYXQiOjE3ODAyOTMzNTYsImV4cCI6MTc4MDI5Njk1Nn0.bjn6revrRPoNwNyW1evcycds12SbSAMjjTg7HXy-A40'
{
    "message": "WOW",
    "status": 0,
    "name": "TCHEUTCHOUA FRIDE",
    "amount": 1000,
    "expectedFee": 0,
    "amountBreakdown": []
}

error:

{
    "message": "Please get a valid token and try again",
    "status": 9001
}

{

    "message": "Query run but with errors",

    "status": 0,

    "name": "Invalid account length. 11 digits needed - 0856171601 - dstAccounts [{\"iden\":\"0856171601\",\"type\":\"ACCOUNT\"}]",

    "amount": 1000,

    "expectedFee": 0,

    "amountBreakdown": []

}

{

    "message": "Query run but with errors",

    "status": 0,

    "name": "Please specify a valid organisation where this account resides",

    "amount": 1000,

    "expectedFee": 0,

    "amountBreakdown": []

}
{

    "message": "Sorry, the minimum to send is 1",

    "status": 9012

}
{

    "message": "The country CN is not recognised",

    "status": 9007

}