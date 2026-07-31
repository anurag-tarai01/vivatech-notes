step 1: create two db MFS_Live_CMR, MFS_Report_CMR ..
step 2 : execute the query
step 3 : run the mfs backend & reporting app with ddl auto update on
step 4 : open redis cli, run command FLUSHALL. Close the redis cli
step 5 : run the curl commands

create these folders in 
C:\Users\Vivat\project\mfs-novapay\mfs-backend-new\static\ 
![[Pasted image 20260703110109.png]]

curl localhost:8090/setup/1

curl localhost:8090/setup/local-currency-wallets-so

curl localhost:8090/setup/setup-commision-profile
curl localhost:8090/setup/setup-otp-expiry-config
curl localhost:8090/setup/setup-default-core-wallet-admin-config

curl localhost:8090/setup/switch-commission-wallet
curl localhost:8090/setup/setup-switch-wallet-api-cost-config



Step 6 : Run all the remain app, then try to login
If login didn't work then again open the redis-cli and write command FLUSHALL and then close the CLI
Open command prompt then execute this curl:-
curl localhost:8090/user/update-cache


spring.datasource.url=jdbc:sqlserver://localhost:1433;databaseName=MFS_Live_Nova  
spring.datasource.username=sa  
spring.datasource.password=Admin@123
