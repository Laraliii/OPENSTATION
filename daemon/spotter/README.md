# Штатный Yandex-споттер: «Алиса» и «Яндекс»

Демон автоматически использует штатный `libyandex_spotter` для имени
**«Алиса»**, если в `/data/local/spotter-root` установлен rootfs со споттером.
Поддерживаются две фразы:

- **«Алиса»** — модель `alisa-2023`;
- **«Яндекс»** — модель `yandex-2022`.

Остальные имена ассистента (и случай, когда rootfs не установлен) продолжают
работать через встроенный OpenStation KWS.

## Что должно лежать на устройстве

```text
/data/local/spotter-root/
├── probe                         # ARM64-бинарник, читает PCM из stdin
├── run_spotter.sh                # обёртка «Алиса» (шаблон в этом каталоге)
├── run_spotter_yandex.sh         # обёртка «Яндекс»
├── lib/                          # glibc для chroot (ld-linux, libc, libm, ...)
├── models/alisa-2023/
│   ├── acoustic_model.nnet
│   ├── flags.txt
│   ├── words.txt
│   └── lda.mat
├── models/yandex-2022/
│   ├── acoustic_model.nnet
│   ├── flags.txt
│   ├── words.txt
│   └── lda.mat
└── run/                          # FIFO и pid-файлы создаются обёртками
```

Модели и `libyandex_spotter.so` — собственность Яндекса, в репозиторий они не
входят. Их можно достать из стокового `Quasar-App.apk`:

```bash
unzip Quasar-App.apk 'assets/quasar/spotter_models/activation/alisa/*' \
      'assets/quasar/spotter_models/activation/yandex/*' -d /tmp/quasar
```

Скопируйте каталог `alisa` в `models/alisa-2023`, а `yandex` — в
`models/yandex-2022`. Обе модели должны быть в формате `<Nnet3>`; вариант
`ru-RU-activation-alisa-plus-yandex` из ТВ-приложения использует старый `<Nnet>`
и с библиотекой Station Max не загружается.

## Установка обёрток

```bash
adb push daemon/spotter/run_spotter.sh /data/local/spotter-root/
adb push daemon/spotter/run_spotter_yandex.sh /data/local/spotter-root/
adb shell chmod 755 /data/local/spotter-root/run_spotter*.sh
```

Обёртки сами создают FIFO и pid-файлы, а демон запускает их при старте и
пишет в оба FIFO 16 кГц mono PCM, пока ассистент находится в `Idle`.
Детект каждой фразы уходит в `POST /api/assistant/trigger` с
`{"wake":true,"keyword":"stock"}` или `{"keyword":"yandex"}`.

Проверка статуса:

```bash
curl -s http://<station>:8080/api/status \
  | python3 -c 'import sys,json; print(json.load(sys.stdin)["assistant"]["wakeWord"])'
```

Ожидаемо:

```json
{"engine":"stock","ready":true,"phrases":["алиса","яндекс"],"status":"stock_listening"}
```

Если модели или обёртки «Яндекс» нет, в `phrases` останется только `алиса`,
и демон продолжит работать в обычном режиме.
