# Extract and fetch TSLA Sell Orders

## Command

```bash
grep '"symbol": "TSLA".*"side": "sell"' ./transaction-log.txt \
  | grep -o '"order_id": "[^"]*"' \
  | sed 's/"order_id": "//;s/"//' \
  | xargs -I {} curl -s "https://example.com/api/{}" \
  >> ./output.txt
