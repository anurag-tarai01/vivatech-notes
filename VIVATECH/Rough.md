if (transfer.getFxSnapshot() != null) {  
    dto.setFxRateSnapshotId(transfer.getFxSnapshot().getId());  
    dto.setTargetCurrency(transfer.getFxSnapshot().getTargetCurrency());  
    dto.setFxTargetAmount(utils.getMoney(transfer.getFxSnapshot().getEffectiveRate(), transfer.getFxSnapshot().getTargetCurrency()));  
}

01M1ATW23RE6H7G9E3MGFE6VWS

100000


Keep bigger precission for rate
when calculate the amount, use 4 decimal
show only 3
[
  {
    "id": "01M1ATW23RE6H7G9E3MGFE6VWS",
    "base_currency": "SOS",
    "created_at": {
      "$binary": {
        "base64": "rO0ABXNyAA1qYXZhLnRpbWUuU2VylV2EuhsiSLIMAAB4cHcNAgAAAABqlOj3N19qQHg="
      }
    },
    "effective_rate": 0.0017,
    "inverse_rate": 574.2,
    "market_rate": 0.00174155,
    "spread_rate": 0,
    "target_currency": "USD",
    "exchange_rate_id": "01M1AHV9JRAS49YEBN5V619DVT"
  }
]