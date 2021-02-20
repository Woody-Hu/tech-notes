# Best Practice For Comsmos Client V3
1. now Cosmos client support httpclient factory https://devblogs.microsoft.com/cosmosdb/httpclientfactory-cosmos-db-net-sdk/

2. https://docs.microsoft.com/en-us/azure/cosmos-db/performance-tips-dotnet-sdk-v3-sql
    
    2.1 we should use singleton client
    
    2.2 diect mode will provide better performance https://docs.microsoft.com/en-us/azure/cosmos-db/sql-sdk-connection-modes

    2.3 client will auto try when get 429 
    >You can change the default retry count by setting the RetryOptions on the CosmosClientOptions instance. By default, the CosmosException with status code 429 is returned after a cumulative wait time of 30 seconds if the request continues to operate above the request rate. This error is returned even when the current retry count is less than the maximum retry count, whether the current value is the default of 9 or a user-defined value.

3. Now cosmos client support transaction which will create a single playload to achive ACID which needs same local partition https://devblogs.microsoft.com/cosmosdb/introducing-transactionalbatch-in-the-net-sdk/

4. for write cases we can use bulk https://devblogs.microsoft.com/cosmosdb/introducing-bulk-support-in-the-net-sdk/

5. we can close the response body to decrease bandwidth https://devblogs.microsoft.com/cosmosdb/enable-content-response-on-write/

6. now cosmos support like https://devblogs.microsoft.com/cosmosdb/like-keyword-cosmosdb/

7. query will use higher ru https://devblogs.microsoft.com/cosmosdb/point-reads-versus-queries/



