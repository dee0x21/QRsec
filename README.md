# QRsec

**Безопасная передача данных через зашифрованный QR-код.**

*Read this in other languages: [Русский](#русский) · [English](#english) · [中文](#中文)*

---

## Русский

Одностраничное веб-приложение без бэкенда: отправитель шифрует данные прямо в браузере, получатель расшифровывает их по коду доступа. Сервер не видит и не хранит ничего — только статические файлы.

### Возможности

* **Сквозное шифрование в браузере** — AES-GCM (256 бит), ключ выводится PBKDF2 (SHA-256, 600 000 итераций) с криптографической солью.
* **Zero-Knowledge** — весь цикл «шифрование → QR → передача → расшифровка» происходит на стороне клиента через Web Crypto API.
* **Передача по URL-фрагменту** — зашифрованный пакет передаётся в хэше ссылки (после `#`). По спецификации HTTP фрагмент не отправляется на сервер и не попадает в access-логи.
* **QR + ссылка** — результат можно отсканировать камерой, скачать PNG или отправить ссылкой (Web Share API / буфер обмена).
* **Обмен файлами P2P** — файлы любого размера передаются напрямую между устройствами по WebRTC DataChannel со спариванием через QR/ссылку (см. ниже).
* **Генератор кодов доступа** — PIN 6/8 цифр или текстовый код `xxxxx-xxxxx` на основе `crypto.getRandomValues`.
* **Автосброс и защита данных**:
  * QR у отправителя сбрасывается через 60 секунд;
  * расшифрованный секрет у получателя показывается только при удержании (Tap & Hold) и уничтожается из памяти через 30 секунд;
  * скопированный в буфер обмена секрет очищается через ~45 секунд.
* **Три языка интерфейса** — RU / EN / ZH, выбор сохраняется в `localStorage`, по умолчанию определяется из `navigator.language`.
* **Без внешних зависимостей** — библиотека QR лежит локально (`qrcode.min.js`), CDN не используются.

### Формат пакета и схема передачи

Пакет имеет формат `nd1.<base64url>`:

| Поле | Размер | Описание |
| --- | --- | --- |
| `nd1.` | 4 байта (текст) | Префикс версии формата |
| version | 1 байт | Текущая версия — `0x01` |
| salt | 16 байт | Соль PBKDF2, генерируется заново при каждом шифровании |
| iv | 12 байт | Nonce для AES-GCM |
| ciphertext | N байт | Шифртекст + тег аутентичности GCM (мин. 16) |

Лимит данных — **1024 байта** (UTF-8). При размере до ~768 байт плотность QR оптимальна.

```
Отправитель                          Получатель
─────────────                        ────────────
1. Ввод данных + кода доступа
2. PBKDF2(passphrase, salt) → ключ
3. AES-GCM encrypt(ключ, данные)
4. Пакет nd1.<base64url>
5. URL: https://host/path#nd1....
        │
        ├──► QR-код (отсканировать камерой)
        └──► ссылка (Web Share / буфер обмена)
                                          6. Браузер открывает URL,
                                             читает хэш #nd1....
                                          7. Ввод кода доступа
                                          8. PBKDF2 → тот же ключ
                                          9. AES-GCM decrypt → данные
                                         10. Tap & Hold для просмотра,
                                             авто-уничтожение через 30 с
```

### Обмен файлами (P2P WebRTC)

Файлы любого размера передаются напрямую между устройствами — без загрузки на сервер и без участия хостинга.

* **Спаривание через QR/ссылку** — отправитель генерирует «QR спаривания» (WebRTC offer в URL-фрагменте `#w1....`); получатель открывает его, создаёт ответный код `w2....` и передаёт отправителю любым каналом (мессенджер, буфер обмена).
* **Двойное шифрование** — транспорт WebRTC защищён DTLS, а поверх каждого 32 КБ-блока файла применяется AES-GCM с ключом из кода доступа (PBKDF2, та же схема, что и для текста).
* **Целостность** — отправитель вычисляет SHA-256 всего файла; получатель сверяет контрольную сумму после приёма.
* **Zero-Knowledge сохраняется** — публичный STUN-сервер (`stun.l.google.com`) видит только служебные метаданные установки соединения, но не данные; сервер хостинга в передаче не участвует.

Ограничения: оба устройства должны быть онлайн одновременно; без TURN-сервера часть пар NAT может не соединиться (в одной сети — всегда работает); на время приёма файл удерживается в памяти браузера (~2× размер).

### Структура проекта

```
QR/
├── index.html      # Разметка, стили и логика (один файл)
├── qrcode.min.js   # Локальная библиотека генерации QR (MIT)
└── README.md
```

### Запуск и использование

Web Crypto API доступен только в **безопасном контексте** (HTTPS или `localhost`). Прямое открытие по `file://` заблокирует работу.

```bash
# Python
python -m http.server 8080
# Node.js
npx serve .
```

* **Отправитель:** Введите текст -> задайте или сгенерируйте код -> нажмите «Сформировать QR» -> передайте QR или ссылку. Пароль передаётся отдельно.
* **Получатель:** Откройте ссылку/сканируйте код -> введите код доступа -> нажмите «Расшифровать» -> удерживайте Tap & Hold.

---

## English

A zero-backend, single-page web app for encrypted data sharing via QR codes and URL fragments. The sender encrypts text directly in the browser, and the recipient decrypts it using a shared passphrase. The hosting server never stores or sees any sensitive data.

### Features

* **In-Browser End-to-End Encryption** — AES-GCM (256-bit), PBKDF2 key derivation (SHA-256, 600,000 iterations) with a unique cryptographic salt.
* **Zero-Knowledge** — The full lifecycle (encrypt → QR → transfer → decrypt) executes purely on the client side via the Web Crypto API.
* **URL Fragment Privacy** — The encrypted payload is attached to the URL hash (after `#`). Following standard HTTP specifications, the hash is never sent to the web server or written to access logs.
* **QR + URL Sharing** — Scan with a mobile camera, download as PNG, or share the link (Web Share API / clipboard fallback).
* **P2P File Transfer** — Files of any size travel directly between devices over WebRTC DataChannel, paired via QR/link (see below).
* **Passcode Generator** — Cryptographically secure 6/8-digit PINs and alphanumeric `xxxxx-xxxxx` codes powered by `crypto.getRandomValues`.
* **Auto-Purge & Privacy Defense**:
  * Sender QR view auto-clears after 60 seconds;
  * Receiver plaintext is visible only while holding the container (Tap & Hold) and is wiped from memory after 30 seconds;
  * Copied clipboard data is automatically purged after ~45 seconds.
* **Tri-lingual UI** — RU / EN / ZH with persistent selection in `localStorage` and fallback to `navigator.language`.
* **Zero Remote Dependencies** — Bundled with local `qrcode.min.js`, operating fully offline once loaded.

### Payload Format & Workflow

Format: `nd1.<base64url>`

| Field | Size | Description |
| --- | --- | --- |
| `nd1.` | 4 bytes (text) | Format version prefix |
| version | 1 byte | Current protocol version — `0x01` |
| salt | 16 bytes | Random PBKDF2 salt per encryption cycle |
| iv | 12 bytes | AES-GCM nonce |
| ciphertext | N bytes | Ciphertext + GCM authentication tag (min. 16) |

Maximum payload size: **1024 bytes** (UTF-8). Payloads under ~768 bytes produce optimal QR scanning density.

```
Sender                               Recipient
──────                               ─────────
1. Input data + passcode
2. PBKDF2(passphrase, salt) → Key
3. AES-GCM encrypt(Key, data)
4. Packet: nd1.<base64url>
5. URL: https://host/path#nd1....
        │
        ├──► QR code (camera scan)
        └──► Link (Web Share / clipboard)
                                     6. Open URL, parse hash #nd1....
                                     7. Enter passcode
                                     8. PBKDF2 → Identical Key
                                     9. AES-GCM decrypt → Plaintext
                                    10. Tap & Hold to reveal,
                                        auto-purge after 30s
```

### P2P File Transfer (WebRTC)

Files of any size travel directly between devices — no upload to a server, the host is not involved.

* **QR/link pairing** — the sender generates a "pairing QR" (a WebRTC offer in the URL fragment `#w1....`); the receiver opens it, creates a response code `w2....`, and passes it back to the sender over any channel (messenger, clipboard).
* **Double encryption** — the WebRTC transport is protected by DTLS, and on top of that every 32 KB block of the file is encrypted with AES-GCM using a key derived from the access code (PBKDF2, same scheme as for text).
* **Integrity** — the sender computes SHA-256 over the whole file; the receiver verifies the checksum after reception.
* **Zero-Knowledge preserved** — the public STUN server (`stun.l.google.com`) only sees connection-setup metadata, never data; the hosting server is not involved in the transfer.

Limitations: both devices must be online at the same time; without a TURN server some NAT pairs may fail to connect (always works on the same network); while receiving, the file is held in browser memory (~2× its size).

### Local Setup & Usage

The Web Crypto API requires a **secure context** (HTTPS or `localhost`). Direct execution via `file://` is restricted.

```bash
# Python
python -m http.server 8080
# Node.js
npx serve .
```

* **Sender:** Enter message -> generate or enter a passcode -> click "Generate QR" -> share QR or link. Transmit the passcode via a separate channel.
* **Recipient:** Open the link or scan the QR -> enter the passcode -> click "Decrypt" -> hold the reveal box to view.

---

## 中文

无需后端的纯前端单页面 Web 应用：通过加密 QR 码与 URL 哈希片段安全传递数据。发送方在浏览器本地对文本加密，接收方凭访问密码完成解密。静态托管服务器完全不参与计算，也无法截获或存储任何机密。

### 核心功能

* **浏览器原生端到端加密** — 基于 AES-GCM (256-bit)，使用 PBKDF2（SHA-256，600,000 次加盐迭代）派生高强度密钥。
* **零知识架构 (Zero-Knowledge)** — “加密 → 生成二维码 → 传输 → 解密”全流程均在客户端通过 Web Crypto API 硬件加速运行。
* **URL 片段隐私隔离** — 密文存储于链接的哈希部分（`#` 之后）。遵循 HTTP 标准规范，哈希片段绝不会发送至服务器端，更不会出现在访问日志中。
* **二维码与链接双模传输** — 支持手机相机扫码、下载 PNG 图片或通过系统 Web Share API / 剪贴板快速分享链接。
* **P2P 文件传输** — 任意大小的文件通过 WebRTC DataChannel 在设备之间直接传输，配对方式为二维码/链接（详见下文）。
* **密码生成工具** — 基于 `crypto.getRandomValues` 安全生成 6 位 / 8 位数字 PIN 或 `xxxxx-xxxxx` 格式的文本密码。
* **防窥屏与数据自毁**：
  * 发送端二维码展示 60 秒后自动超时重置；
  * 接收端明文仅在按住容器（Tap & Hold）时显示，30 秒后自动从浏览器内存中擦除；
  * 复制到剪贴板的明文在约 45 秒后自动尝试清空。
* **三语界面** — 完整支持 RU / EN / ZH，语言设置保存至 `localStorage`，默认跟随浏览器系统语言。
* **无外部依赖** — 离线封装 `qrcode.min.js`，不依赖任何第三方 CDN，页面加载后支持离线使用。

### 数据包结构与流转架构

加密数据包规范：`nd1.<base64url>`

| 字段 | 大小 | 说明 |
| --- | --- | --- |
| `nd1.` | 4 字节 (字符) | 数据格式版本标识前缀 |
| version | 1 字节 | 当前协议版本号 — `0x01` |
| salt | 16 字节 | PBKDF2 随机盐，每次加密重新生成 |
| iv | 12 字节 | AES-GCM 初始化向量 (Nonce) |
| ciphertext | N 字节 | 密文主体 + GCM 认证标签 (不少于 16 字节) |

单次传输最大支持 **1024 字节** (UTF-8)。文本在 768 字节以内时，二维码识别效率最佳。

```
发送方 (Sender)                      接收方 (Recipient)
──────────────                      ─────────────────
1. 输入原始数据 + 访问密码
2. PBKDF2(密码, 盐) → 派生密钥
3. AES-GCM 加密(密钥, 数据)
4. 生成数据包: nd1.<base64url>
5. 拼接 URL: https://host/path#nd1....
        │
        ├──► 二维码 (手机镜头扫描)
        └──► 链接 (Web Share / 剪贴板)
                                     6. 打开链接并解析 URL 哈希 #nd1....
                                     7. 输入解密密码
                                     8. PBKDF2 → 派生相同密钥
                                     9. AES-GCM 解密 → 还原明文
                                    10. 长按 (Tap & Hold) 显示明文，
                                        30 秒后内存自毁
```

### P2P 文件传输（WebRTC）

任意大小的文件在设备之间直接传输——不上传服务器，托管方不参与。

* **二维码/链接配对** — 发送方生成"配对二维码"（URL 片段 `#w1....` 中的 WebRTC offer）；接收方打开后生成响应码 `w2....`，并通过任意渠道（即时通讯、剪贴板）回传给发送方。
* **双重加密** — WebRTC 传输层由 DTLS 保护，文件每 32 KB 数据块再叠加 AES-GCM 加密，密钥由访问密码派生（PBKDF2，与文本模式相同方案）。
* **完整性校验** — 发送方计算整个文件的 SHA-256；接收方在接收完成后核对校验和。
* **Zero-Knowledge 保持不变** — 公共 STUN 服务器（`stun.l.google.com`）仅可见连接建立的元数据，无法获取任何数据；托管服务器不参与传输过程。

限制：两台设备必须同时在线；无 TURN 服务器时部分 NAT 组合可能无法连通（同一网络内始终可用）；接收期间文件驻留于浏览器内存中（约为文件大小的 2 倍）。

### 快速启动与使用

Web Crypto API 必须运行在**安全上下文**环境（HTTPS 或 `localhost`），直接以 `file://` 双击打开将被安全策略拦截。

```bash
# Python
python -m http.server 8080
# Node.js
npx serve .
```

* **发送方：** 输入明文 -> 输入或生成密码 -> 点击“生成安全二维码” -> 将二维码或链接发送给接收人。访问密码请通过独立的安全渠道另行告知。
* **接收方：** 扫码或点击链接进入页面 -> 输入解密密码 -> 点击“解密数据” -> 长按显示框查看解密内容。

---

## License

MIT License.

Dependency: [`qrcode.min.js`](https://github.com/davidshimjs/qrcodejs) (MIT).