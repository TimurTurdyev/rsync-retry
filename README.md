# rsync-retry

rsync с повторами при обрыве связи: скрипт перезапускает rsync, пока папка не доедет целиком.

```bash
./rsync-retry /var/www/shop/image/ user@new-server:/var/www/shop/image/
```
