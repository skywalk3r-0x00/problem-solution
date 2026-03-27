# Troubleshooting High Disk Usage (NGINX VM)

## 1. Troubleshooting Steps

| Step | Actions |
|------|--------|
| 1 | **Check disk usage & partitions**  <br> ```bash df -h ``` <br> ```bash lsblk ``` |
| 2 | **Check NGINX logs & cache** <br><br> **Log size** <br> ```bash du -sh /var/log/nginx/* ``` <br><br> **Cache / Temp** <br> ```bash du -sh /var/cache/nginx/ 2>/dev/null ``` <br> ```bash du -sh /tmp/ ``` <br><br> **NGINX status** <br> ```bash systemctl status nginx ``` |
| 3 | **Check deleted but still open files** <br> ```bash lsof | grep deleted ``` <br>|

---

## 2. Expected Causes & Fixes

| No | Cause | Impact | Recovery |
|----|------|--------|----------|
| 1 | Log rotation not configured | Disk gets full over time | ```bash logrotate -f /etc/logrotate.d/nginx ``` |
| 2 | Verbose / Debug logging enabled | Log grows rapidly | Set log level to `warn` or `error` <br><br> ```bash # Edit config /etc/nginx/nginx.conf ``` <br> ```bash /var/log/nginx/error.log warn; ``` <br> ```bash systemctl reload nginx ``` |
| 3 | Deleted files still held by process | Disk space not actually freed | Reopen file handles <br><br> ```bash nginx -s reopen ``` <br><br> Verify: <br> ```bash df -h ``` |

---

## 3. Key Observations

- Disk full does **not always mean visible files are large**
- Logs are the most common root cause in NGINX-only systems
- “Phantom usage” often comes from deleted-but-open files
- Always verify after fix using `df -h`

---

## 4. Best Practices (Prevention)

- Enable **logrotate** with retention policy  
- Avoid **debug logging in production**  
- Monitor disk usage via **CloudWatch / Prometheus**  
- Set alert threshold at **70–80% usage**  

---
