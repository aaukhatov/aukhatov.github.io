---
title: How to Download a File Partially
categories: java
---
## Как скачивать файл порциями

Иногда требуется скачивать файл порциями. Причины бывают разные, например слишком __“большой”__ объем файла, ширина канала не достаточна или сервер ограничивает объем данных для скачивания.

В этой статье опишу каким образом реализовать скачивание файла небольшими порциями через протокол HTTP. Здесь не рассматриваются вопросы безопасности, это контекст другой статьи.

## Об HTTP

Для таких целей HTTP предоставляет заголовок Range для запроса. В котором указывается диапазон байтов для скачивания. Заголовок Range относится только к телу запроса, заголовки сюда не входят.

Спецификация определяет следующие форматы указания значений заголовка:

`Range: bytes=first-byte-pos "-" [last-byte-pos]`

**first-byte-pos** - начальное смещение байта с которого необходимо начать (продолжить) скачивание, оно должно быть больше либо равно 0, и меньше либо равно **last-byte-pos** минус один;

**last-byte-pos** - конечное смещение байта до которого необходимо скачать файл, оно должно быть больше либо равно **first-byte-pos** и при этом меньше либо равно скачиваемому размеру файла минус один __(потому что это смещение, то есть индекс в массиве байтов)__.

> `bytes=0-255` // включительно
`bytes=256-512` // включительно

`Range: bytes=first-byte-pos "-"`
Скачать начиная с позиции first-byte-pos до конца.

> **bytes=**`512-`

`Range: bytes="-" last-byte-pos`
Скачать  **last-byte-pos** с конца.

> **bytes=**`-32`

На подобный запрос сервер в ответ пришлёт два возможных статуса:

`206 Partial Content` - файл успешно скачан частично;
`416 Range Not Satisfiable` - неудовлетворительный диапазон для скачивания.

Конечно же ответов может быть больше. В контексте статьи они нас не интересуют.

И заголовок Content-Range в котором указан запрошенный диапазон и общий размер.

`Content-Range: bytes 256-512/1024`

Этот заголовок сообщает что пришёл ответ на запрос с `256-512` позиции в массиве байтов из `1024` байтов.

## Реализация на Java

В качестве HTTP клиента возьмем стандартный из JDK, доступный с Java 11 - `java.net.http.HttpClient`.

Для реализации логики выполнения запроса по порциям, напишем класс обёртку - `art.aukhatov.http.WebClient`.

Опишем интерфейс этого класса

`byte[] download(String uri, int chunkSize)` - скачивает файл по указанным порциям байтов
`Response download(String uri, int firstBytePos, int lastBytePos)` - скачивает файл по указанному диапазону
В случае если переданный URI не валидный, то метод бросает исключение `java.net.URISyntaxException`.
Исключение `java.io.IOException` бросается если какая-либо неожиданная ошибка с вводом/выводом.

```java
package art.aukhatov.http;

import java.io.BufferedInputStream;
import java.net.http.HttpClient;
import java.net.http.HttpHeaders;
import java.time.Duration;

public class WebClient {

     private final HttpClient httpClient;

     public WebClient() {
         this.httpClient = HttpClient.newBuilder()
              .connectTimeout(Duration.ofSeconds(10))
              .build();
     }


     public static class Response {
          final BufferedInputStream inputStream;
          final int status;
          final HttpHeaders headers;

          public Response(BufferedInputStream inputStream, int status, HttpHeaders headers) {
               this.inputStream = inputStream;
               this.status = status;
               this.headers = headers;
          }
     }
}
```

В качестве представления ответа опишем `nested class WebClient.Response` с полями BufferedInputStream, HTTP Status, HTTP Header.
Эти данные необходимы для формирования результирующего массива байтов и понимания продолжать скачивать или нет.

```java
import java.io.BufferedInputStream;
import java.io.IOException;
import java.io.InputStream;
import java.net.URI;
import java.net.URISyntaxException;
import java.net.http.HttpClient;
import java.net.http.HttpHeaders;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

private static final String HEADER_RANGE = "Range";
private static final String RANGE_FORMAT = "bytes=%d-%d";

public Response download(final String uri, int firstBytePos, int lastBytePos)
    throws URISyntaxException, IOException, InterruptedException {

    HttpRequest request = HttpRequest
            .newBuilder(new URI(uri))
            .header(HEADER_RANGE, format(RANGE_FORMAT, firstBytePos, lastBytePos))
            .GET()
            .version(HttpClient.Version.HTTP_2)
            .build();

    HttpResponse<InputStream> response = httpClient.send(request, HttpResponse.BodyHandlers.ofInputStream());

    return new Response(new BufferedInputStream(response.body()), response.statusCode(), response.headers());
}
```

Этот метод скачивает указанный диапазон данных. Но прежде чем начать нам нужно знать сколько всего данных нам надо ожидать. Для этого необходимо сделать запрос без получения контента. Воспользуемся методом `HEAD`.

```java
import java.util.OptionalLong;

private static final String HTTP_HEAD = "HEAD";
private static final String HEADER_CONTENT_LENGTH = "content-length";

private long contentLength(final String uri)
			throws URISyntaxException, IOException, InterruptedException {

    HttpRequest headRequest = HttpRequest
            .newBuilder(new URI(uri))
            .method(HTTP_HEAD, HttpRequest.BodyPublishers.noBody())
            .version(HttpClient.Version.HTTP_2)
            .build();

    HttpResponse<String> httpResponse = httpClient.send(headRequest, HttpResponse.BodyHandlers.ofString());

    OptionalLong contentLength = httpResponse
            .headers().firstValueAsLong(HEADER_CONTENT_LENGTH);

    return contentLength.orElse(0L);
}
```

Теперь у нас есть ожидаемая длина файла в байтах.

Можем приступить к написанию метода контролирующего скачивание файла по порциям. Для удобства, договоримся что размер порций будет передаваться вторым аргументом в этот метод `byte[] download(final String uri, int chunkSize)`. Хотя можно было бы придумать умный способ определения размера порций.

#### Определим размер файла
```java
final int expectedLength = (int) contentLength(uri);
```

#### Начальное смещение
```java
int firstBytePos = 0;
```

#### Конечное смещение
```java
int lastBytePos = chunkSize - 1;
```

Скачанные данные за каждую итерацию необходимо накапливать, создадим для этого массив, размер массива нам уже известен.

```java
byte[] downloadedBytes = new byte[expectedLength];
```

Дополнительно к самому массиву необходимо определять суммарно сколько данных скачано. Поэтому эту длину будем считать отдельно.

```java
int downloadedLength = 0;
```

Условие цикла простое: продолжаем скачивать пока не достигнем ожидаемого размера.
После того как успешно скачали очередную порцию данных, необходимо его прочитать и положить в результирующий массив, увеличить счетчики и диапазон.

```java
private static final int HTTP_PARTIAL_CONTENT = 206;

while (downloadedLength < expectedLength) {

        Response response;

        try {
            response = download(uri, firstBytePos, lastBytePos);
        }

        try (response.inputStream) {
            byte[] chunkedBytes = response.inputStream.readAllBytes();

            downloadedLength += chunkedBytes.length;

            if (isPartial(response)) {
                System.arraycopy(chunkedBytes, 0, downloadedBytes, firstBytePos, chunkedBytes.length);
                firstBytePos = lastBytePos + 1;
                lastBytePos = Math.min(lastBytePos + chunkSize, expectedLength - 1);
            }
        }
    }

    return downloadedBytes;
}

private boolean isPartial(Response response) {
    return response.status == HTTP_PARTIAL_CONTENT;
}
```

На вид всё хорошо. **Что не так?**
Когда при скачивании или чтении что-то пойдет не так, броситься I/O исключение и скачивание прекратиться.
Отсутствуют fallback. Давайте напишем простой fallback ввиде количества совершенных попыток.

Определим поле для веб-клиента содержащий максимальное количество допустимых попыток скачивания файла.

```java
private int maxAttempts;

public int maxAttempts() {
    return maxAttempts;
}

public void setMaxAttempts(int maxAttempts) {
    this.maxAttempts = maxAttempts;
}
```

Будем ловить отдельно каждое исключение и инкрементировать локальный счетчик попыток. Цикл скачивания должен остановиться если количество совершенных попыток превышает допустимое. Поэтому дополним условие цикла.

```java
private static final int DEFAULT_MAX_ATTEMPTS = 3;

int attempts = 1;

while (downloadedLength < expectedLength && attempts < maxAttempts) {

    Response response;

    try {
        response = download(uri, firstBytePos, lastBytePos);
    } catch (IOException e) {
        attempts++;
        continue;
    }

    try (response.inputStream) {
        byte[] chunkedBytes = response.inputStream.readAllBytes();

        downloadedLength += chunkedBytes.length;

        if (isPartial(response)) {
            System.arraycopy(chunkedBytes, 0, downloadedBytes, firstBytePos, chunkedBytes.length);
            firstBytePos = lastBytePos + 1;
            lastBytePos = Math.min(lastBytePos + chunkSize, expectedLength - 1);
        }
    } catch (IOException e) {
        attempts++;
        continue;
    }

    attempts = 1;
}
```

Дополним метод еще логами. Окончательный вариант выглядит так:

```java
package art.aukhatov.http;

import java.io.BufferedInputStream;
import java.io.IOException;
import java.io.InputStream;
import java.net.URI;
import java.net.URISyntaxException;
import java.net.http.HttpClient;
import java.net.http.HttpHeaders;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;
import java.util.OptionalLong;

import static java.lang.String.format;
import static java.lang.System.err;
import static java.lang.System.out;

public class WebClient {

	private static final String HEADER_RANGE = "Range";
	private static final String RANGE_FORMAT = "bytes=%d-%d";
	private static final String HEADER_CONTENT_LENGTH = "content-length";
	private static final String HTTP_HEAD = "HEAD";
	private static final int DEFAULT_MAX_ATTEMPTS = 3;
	private static final int HTTP_PARTIAL_CONTENT = 206;

	private final HttpClient httpClient;
	private int maxAttempts;

	public WebClient() {
		this.httpClient = HttpClient.newBuilder()
				.connectTimeout(Duration.ofSeconds(10))
				.build();
		this.maxAttempts = DEFAULT_MAX_ATTEMPTS;
	}

	public WebClient(HttpClient httpClient) {
		this.httpClient = httpClient;
	}

	private long contentLength(final String uri)
			throws URISyntaxException, IOException, InterruptedException {

		HttpRequest headRequest = HttpRequest
				.newBuilder(new URI(uri))
				.method(HTTP_HEAD, HttpRequest.BodyPublishers.noBody())
				.version(HttpClient.Version.HTTP_2)
				.build();

		HttpResponse<String> httpResponse = httpClient.send(headRequest, HttpResponse.BodyHandlers.ofString());

		OptionalLong contentLength = httpResponse
				.headers().firstValueAsLong(HEADER_CONTENT_LENGTH);

		return contentLength.orElse(0L);
	}

	public Response download(final String uri, int firstBytePos, int lastBytePos)
			throws URISyntaxException, IOException, InterruptedException {

		HttpRequest request = HttpRequest
				.newBuilder(new URI(uri))
				.header(HEADER_RANGE, format(RANGE_FORMAT, firstBytePos, lastBytePos))
				.GET()
				.version(HttpClient.Version.HTTP_2)
				.build();

		HttpResponse<InputStream> response = httpClient.send(request, HttpResponse.BodyHandlers.ofInputStream());

		return new Response(new BufferedInputStream(response.body()), response.statusCode(), response.headers());
	}

	public byte[] download(final String uri, int chunkSize)
			throws URISyntaxException, IOException, InterruptedException {

		final int expectedLength = (int) contentLength(uri);
		int firstBytePos = 0;
		int lastBytePos = chunkSize - 1;

		byte[] downloadedBytes = new byte[expectedLength];
		int downloadedLength = 0;

		int attempts = 1;

		while (downloadedLength < expectedLength && attempts < maxAttempts) {

			Response response;

			try {
				response = download(uri, firstBytePos, lastBytePos);
			} catch (IOException e) {
				attempts++;
				err.println(format("I/O error has occurred. %s", e));
				out.println(format("Going to do %d attempt", attempts));
				continue;
			}

			try (response.inputStream) {
				byte[] chunkedBytes = response.inputStream.readAllBytes();

				downloadedLength += chunkedBytes.length;

				if (isPartial(response)) {
					System.arraycopy(chunkedBytes, 0, downloadedBytes, firstBytePos, chunkedBytes.length);
					firstBytePos = lastBytePos + 1;
					lastBytePos = Math.min(lastBytePos + chunkSize, expectedLength - 1);
				}
			} catch (IOException e) {
				attempts++;
				err.println(format("I/O error has occurred. %s", e));
				out.println(format("Going to do %d attempt", attempts));
				continue;
			}

			attempts = 1; // reset attempts counter
		}

		if (attempts >= maxAttempts) {
			err.println("A file could not be downloaded. Number of attempts are exceeded.");
		}

		return downloadedBytes;
	}

	private boolean isPartial(Response response) {
		return response.status == HTTP_PARTIAL_CONTENT;
	}

	public int maxAttempts() {
		return maxAttempts;
	}

	public void setMaxAttempts(int maxAttempts) {
		this.maxAttempts = maxAttempts;
	}

	public static class Response {
		final BufferedInputStream inputStream;
		final int status;
		final HttpHeaders headers;

		public Response(BufferedInputStream inputStream, int status, HttpHeaders headers) {
			this.inputStream = inputStream;
			this.status = status;
			this.headers = headers;
		}
	}
}
```

Теперь можем написать тест на скачивание файла. Для примера возьмем рандомный файл в Интернете из доступных без аутентификации: https://file-examples.com/wp-content/uploads/2017/10/file-example_PDF_1MB.pdf

Сохраним файл во временную директорию. И проверим размер файла.

```java
class WebClientTest {

	@Test
	void downloadByChunk() throws IOException, URISyntaxException, InterruptedException {
		WebClient fd = new WebClient();
		byte[] data = fd.download("https://file-examples.com/wp-content/uploads/2017/10/file-example_PDF_1MB.pdf", 262_144);
		final String downloadedFilePath = System.getProperty("java.io.tmpdir") + "sample.pdf";
		System.out.println("File has downloaded to " + downloadedFilePath);

		Path path = Paths.get(downloadedFilePath);

		try (OutputStream outputStream = Files.newOutputStream(path)) {
			outputStream.write(data);
			outputStream.flush();

			assertEquals(1_042_157, Files.readAllBytes(Paths.get(downloadedFilePath)).length);

			Files.delete(path);
		}
	}
}
```
