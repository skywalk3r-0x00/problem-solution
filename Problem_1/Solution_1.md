# Extract and Fetch TSLA Sell Orders

This command filters transaction logs to extract `order_id` values for TSLA sell orders, then queries an API for each order and appends the results to an output file.

## Command

```bash
grep '"symbol": "TSLA".*"side": "sell"' ./transaction-log.txt \
  | grep -o '"order_id": "[^"]*"' \
  | sed 's/"order_id": "//;s/"//' \
  | xargs -I {} curl -s "https://example.com/api/{}" \
  >> ./output.txt
