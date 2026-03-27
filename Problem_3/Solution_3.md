# Troubleshooting Guide: Nginx Disk Space Issues

## How to Troubleshoot the Issue

| Steps | Actions |
|-------|---------|
| 1 | **Check disk usage which partition is full**<br>`user@user: df -h`<br>`user@user: lsblk` |
| 2 | **Check nginx log size**<br>`user@user: du -sh /var/log/nginx/*`<br><br>**Check temp/cache**<br>`user@user: du -sh /var/cache/nginx/ 2>/dev/null`<br>`user@user: du -sh /tmp/`<br><br>**Check nginx status**<br>`user@user: systemctl status nginx` |
| 3 | **Check deleted but still open files**<br>`user@user: lsof | grep deleted` |

## Expected Causes/Scenarios and Fixes

| No | Cause | Impact | Recovery |
|----|-------|--------|----------|
| 1 | Log rotation config missing | Disk fills | `user@user: logrotate -f /etc/logrotate.d/nginx` |
| 2 | Verbose or Debug log enabled | Disk fills | Set log to warn or error<br><br>```bash<br># /etc/nginx/nginx.conf<br>error_log /var/log/nginx/error.log warn;<br>systemctl reload nginx<br>``` |
| 3 | Deleted files still consuming | Disk space is not actually freed | Make nginx reopen files:<br>`user@user: nginx -s reopen`<br><br>Then verify:<br>`user@user: df -h` |
