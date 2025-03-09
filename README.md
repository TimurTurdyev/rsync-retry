# rsync-retry

Обертка над rsync для переноса больших папок с сервера на сервер, когда связь рвется. Обрыв не страшен: rsync запускается снова и докачивает, пауза растет с каждой неудачей. Можно уйти в фон и закрыть ssh.

Требуется bash 3.2+ и rsync, подходит для Linux и macOS.

## Установка

```bash
curl -fsSL https://raw.githubusercontent.com/TimurTurdyev/rsync-retry/main/rsync-retry -o /usr/local/bin/rsync-retry
chmod +x /usr/local/bin/rsync-retry
```

## Примеры

Перенести картинки магазина на другой сервер, webp-копии пропустить:

```bash
rsync-retry -e '*.webp' /var/www/shop/image/ user@new-server:/var/www/shop/image/
```

То же в фоне, попыток сколько угодно, скорость до 5 МБ/с:

```bash
rsync-retry -b -a 0 -l 5m -n images /var/www/shop/image/ user@new-server:/var/www/shop/image/
rsync-retry status images
rsync-retry stop images
```

Флаги для rsync идут после `--`:

```bash
rsync-retry backup/ user@host:backup/ -- --delete -e 'ssh -p 2222'
```

## Повторы

Повторяются только сбои связи: ssh, таймауты, оборванный поток (коды rsync 5, 10, 12, 30, 35, 255). При ошибке в аргументах, нехватке прав или места скрипт сразу завершается с кодом rsync. Код 24 (часть файлов удалили во время копирования) считается успехом.

Пауза `--delay` удваивается на каждом повторе, но не больше `--max-delay`. Если данные не идут дольше `--timeout` секунд, rsync рвет соединение и попытка повторяется.

## Фон

Pid и лог лежат в `~/.rsync-retry`, другой каталог задается переменной `RSYNC_RETRY_HOME`. Два запуска с одним именем не пойдут параллельно, второй завершится с ошибкой. По умолчанию имя берется по папке источника.

## Уведомления

`--on-success` и `--on-failure` выполняют команду, когда передача закончилась. В окружении есть `RSYNC_RETRY_NAME`, `RSYNC_RETRY_STATUS` (`ok` или `failed`), `RSYNC_RETRY_ATTEMPTS`, `RSYNC_RETRY_CODE` и `RSYNC_RETRY_LOG`. Сообщение в Telegram:

```bash
rsync-retry -b -n images \
  --on-success 'curl -s "https://api.telegram.org/bot$TG_TOKEN/sendMessage" -d chat_id=$TG_CHAT -d text="$RSYNC_RETRY_NAME: готово за $RSYNC_RETRY_ATTEMPTS попыток"' \
  --on-failure 'curl -s "https://api.telegram.org/bot$TG_TOKEN/sendMessage" -d chat_id=$TG_CHAT -d text="$RSYNC_RETRY_NAME: ошибка $RSYNC_RETRY_CODE"' \
  /var/www/shop/image/ user@new-server:/var/www/shop/image/
```
