
---
title: OpenFeign — podstawy
permalink: /spring/modul-03-http/openfeign/
---

# OpenFeign (Spring Cloud) – podstawy

**OpenFeign** to biblioteka, która pozwala wywoływać inne API HTTP w Springu w prosty sposób – **tworzysz interfejs w Javie**, dodajesz adnotacje i Spring generuje klienta HTTP automatycznie.

---

## Po co?
- zamiast pisać ręcznie `RestTemplate` / `WebClient`
- czytelny kod: metoda = endpoint
- mniej “boilerplate” i szybsza integracja z zewnętrznymi usługami
- łatwiej testować (można mockować interfejs)

---

## Jak działa (w skrócie)
1. Dodajesz zależność `spring-cloud-starter-openfeign`
2. Włączasz Feign w aplikacji: `@EnableFeignClients`
3. Tworzysz interfejs z `@FeignClient`
4. Definiujesz metody z `@GetMapping`, `@PostMapping`, `@RequestParam`, `@PathVariable` itd.

---

## Konfiguracja projektu (Maven)

### 1) Zależność OpenFeign
W `pom.xml` dodaj:

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

Jeśli projekt nie startuje lub masz konflikty wersji, zwykle brakuje Spring Cloud BOM (dopasowuje wersje bibliotek Cloud do Spring Boot).

### 2) (Opcjonalnie) Spring Cloud BOM

Dodaj w pom.xml:
```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version><!-- dopasuj do Spring Boot --></version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

## Włączenie Feign w aplikacji

W klasie głównej dodaj @EnableFeignClients:
```xml
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.openfeign.EnableFeignClients;

@EnableFeignClients
@SpringBootApplication
public class AppApplication {
  public static void main(String[] args) {
    SpringApplication.run(AppApplication.class, args);
  }
}
```

## Minimalny klient (GET + parametry)

Przykład: iTunes Search API.

```xml
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;

@FeignClient(name = "itunes", url = "https://itunes.apple.com")
public interface ItunesClient {

  // GET https://itunes.apple.com/search?term=shawn+mendes&limit=1
  @GetMapping("/search")
  String search(@RequestParam String term, @RequestParam int limit);
}
```

## Użycie w serwisie

```xml
import org.springframework.stereotype.Service;

@Service
public class MusicService {

  private final ItunesClient itunesClient;

  public MusicService(ItunesClient itunesClient) {
    this.itunesClient = itunesClient;
  }

  public String findOneArtist(String name) {
    return itunesClient.search(name, 1);
  }
}
}
```

## Użycie w kontrolerze (demo)

```
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class DemoController {

  private final MusicService musicService;

  public DemoController(MusicService musicService) {
    this.musicService = musicService;
  }

  @GetMapping("/demo/itunes")
  public String itunes(@RequestParam String term) {
    return musicService.findOneArtist(term);
  }
}
```

## Zwrot JSON jako obiekt (Record / DTO)

Zamiast String, lepiej mapować na typy (czytelniej i bez ręcznego parsowania).

1) Proste rekordy (minimum pól)


```
import java.util.List;

public record ItunesSearchResponse(int resultCount, List<ItunesTrack> results) {}
public record ItunesTrack(String trackName, String artistName, String collectionName) {}
```

2) Feign zwraca rekord

```
@FeignClient(name = "itunes", url = "https://itunes.apple.com")
public interface ItunesClient {
  @GetMapping("/search")
  ItunesSearchResponse search(@RequestParam String term, @RequestParam int limit);
}
```

Uwaga: mapowanie działa, jeśli masz JSON mappera (zwykle jest w Spring Boot Web: Jackson).

## POST (Body) + nagłówki
Przykład POST z body

```
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;

@FeignClient(name = "example-api", url = "${example.api.url}")
public interface ExampleClient {

  @PostMapping("/items")
  ItemResponse createItem(@RequestBody CreateItemRequest request);
}

public record CreateItemRequest(String name, int quantity) {}
public record ItemResponse(String id, String name, int quantity) {}
```

## Przykład nagłówków

```
import org.springframework.web.bind.annotation.RequestHeader;

@FeignClient(name = "secure-api", url = "${secure.api.url}")
public interface SecureClient {

  @GetMapping("/profile")
  String profile(@RequestHeader("Authorization") String bearerToken);
}
```

## Konfiguracja przez application.properties

Warto trzymać URL w configu:

example.api.url=https://api.example.com
secure.api.url=https://secure.example.com

Wtedy w kliencie:

```
@FeignClient(name = "example-api", url = "${example.api.url}")
public interface ExampleClient { ... }

```

## Timeouty (minimum ustawień)

Jeśli API “wisi” lub działa wolno, ustaw timeouty (nazwa właściwości zależy od wersji Spring Cloud).
Często spotykane podejście:

# idea: ograniczamy czas na połączenie i odczyt
properties
spring.cloud.openfeign.client.config.default.connectTimeout=3000
spring.cloud.openfeign.client.config.default.readTimeout=5000



Logowanie requestów (debug)
1) Poziom logów w application.properties
logging.level.com.example=INFO
logging.level.org.springframework.cloud.openfeign=DEBUG
logging.level.feign=DEBUG
2) Feign Logger Level (Bean)
import feign.Logger;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
```
@Configuration
public class FeignLoggingConfig {

  @Bean
  Logger.Level feignLoggerLevel() {
    return Logger.Level.FULL; // BASIC / HEADERS / FULL
  }
}
```
I przypinasz config do klienta:
```
@FeignClient(
  name = "itunes",
  url = "https://itunes.apple.com",
  configuration = FeignLoggingConfig.class
)
public interface ItunesClient { ... }
Obsługa błędów (minimum)
```
Feign przy 4xx/5xx rzuca wyjątki. Minimum: przechwycić i zamienić na sensowny komunikat.
```
import feign.FeignException;
import org.springframework.stereotype.Service;

@Service
public class SafeMusicService {

  private final ItunesClient itunesClient;

  public SafeMusicService(ItunesClient itunesClient) {
    this.itunesClient = itunesClient;
  }

  public String safeSearch(String term) {
    try {
      return itunesClient.search(term, 1).toString();
    } catch (FeignException.NotFound e) {
      return "Nie znaleziono wyników (404).";
    } catch (FeignException e) {
      return "Błąd wywołania zewnętrznego API: " + e.status();
    }
  }
}
```
Testowanie (minimum podejście)
1) Najprościej: mock interfejs Feign w teście serwisu

testujesz logikę serwisu, a nie HTTP.
```
import org.junit.jupiter.api.Test;
import static org.mockito.Mockito.*;
import static org.assertj.core.api.Assertions.*;

class MusicServiceTest {

  @Test
  void shouldCallFeignClient() {
    ItunesClient client = mock(ItunesClient.class);
    when(client.search("shawn mendes", 1)).thenReturn("OK");

    MusicService service = new MusicService(client);

    String result = service.findOneArtist("shawn mendes");
    assertThat(result).isEqualTo("OK");
    verify(client).search("shawn mendes", 1);
  }
}
```
Jeśli używasz rekordów zamiast String, to w thenReturn(...) zwracasz ItunesSearchResponse(...).


## Przykład zapytania reddit 
https://www.reddit.com/r/java/top.json?limit=1&t=year

robi dokładnie to:

/r/java/ → bierze posty z subreddita r/java

top.json → zwraca listę postów posortowaną według TOP (najwięcej upvote’ów) w formacie JSON

limit=1 → zwraca tylko 1 post (najlepszy)

t=year → zakres czasu: ostatni rok (może być też np. day, week, month, all)

Co dostajesz w odpowiedzi (najważniejsze pola)

W JSON-ie masz m.in.:

data.children[0].data.title → tytuł posta

data.children[0].data.author → autor

data.children[0].data.ups / score → punkty

data.children[0].data.permalink → link względny do posta

data.children[0].data.url → link do treści (np. obrazek/link)

data.after → token do paginacji (do następnej strony wyników)

Do czego to się przydaje (praktyczne zastosowania)

Top post dnia/tygodnia/roku → na stronę “Java news”

Agregator treści: zbierasz TOP N i wyświetlasz w swojej aplikacji

Alerty: “jeśli post ma > X upvote’ów, wyślij powiadomienie”

Analiza trendów: porównujesz TOP z różnych zakresów czasu (day vs year)

Demo do OpenFeign: świetny przykład API GET + parametry + JSON

```
package com.psajdak.modul3springboothttp;

import org.springframework.boot.context.event.ApplicationStartedEvent;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

@Component
public class RedditRequestExample {

    private final RedditClient redditClient;

    public RedditRequestExample(RedditClient redditClient) {
        this.redditClient = redditClient;
    }

    @EventListener(ApplicationStartedEvent.class)
    public void makeRequestToReddit() {
        String json = redditClient.topPosts("java", 1, "year");
        System.out.println(json);
    }
}
```



### Najczęstsze problemy (szybkie fixy)

## Brak @EnableFeignClients → klient nie wstrzyknie się (NoSuchBeanDefinition).

## Zły URL (brak https:// lub zła ścieżka) → 404/SSL error.

## Złe mapowanie JSON → sprawdź nazwy pól i typy (czasem potrzebne @JsonProperty).

## Brak timeoutów → aplikacja może “wisieć” na zewnętrznym API.

## Brak logów → dodaj Logger.Level.FULL i logging.level.feign=DEBUG.





