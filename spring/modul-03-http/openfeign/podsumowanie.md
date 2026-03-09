OpenFeign to deklaratywny klient REST dla Javy (szczególnie popularny w ekosystemie Spring Boot). Jego działanie polega na tym, że zamiast ręcznie pisać kod do wysyłania zapytań HTTP (jak np. w RestTemplate czy HttpClient), definiujesz jedynie interfejs z adnotacjami, a Spring automatycznie generuje jego implementację (tzw. proxy) w locie.
Aby to zrozumieć, prześledźmy działanie OpenFeign na Twoim przykładzie: pobieraniu informacji o zespole Sabaton z publicznego API iTunes.
Oto krok po kroku, jak OpenFeign realizuje to zadanie pod spodem.
1. Definicja zapytań w Interfejsie (Deklaratywność)
Zamiast pisać logikę nawiązywania połączenia, tworzysz prosty interfejs:
@FeignClient(name = "itunesClient", url = "https://itunes.apple.com")
public interface ITunesClient {

    @GetMapping("/search")
    ItunesResponse searchArtist(@RequestParam("term") String term, 
                                @RequestParam("entity") String entity);
}

Jak to działa pod spodem?
 * Adnotacja @FeignClient mówi Springowi: "Hej, stwórz dla mnie klienta HTTP, który będzie uderzał pod adres https://itunes.apple.com".
 * @GetMapping oraz @RequestParam to standardowe adnotacje Spring MVC. OpenFeign je czyta i wie, że wywołanie metody searchArtist("sabaton", "musicArtist") musi zostać zamienione na fizyczne zapytanie GET: https://itunes.apple.com/search?term=sabaton&entity=musicArtist.
2. Automatyczna Deserializacja (Mapowanie DTO)
API iTunes zwraca odpowiedź w formacie JSON (zawierającą m.in. nazwę, gatunek i identyfikator artysty – warto zaznaczyć, że iTunes API nie zwraca długich tekstowych biografii, ale podstawowe metadane).
Definiujesz obiekty (DTO), do których te dane mają trafić:
public record ItunesResponse(List<Artist> results) {}
public record Artist(String artistName, String primaryGenreName, String artistLinkUrl) {}

Jak to działa pod spodem?
Gdy iTunes zwraca JSON z danymi Sabatonu, OpenFeign przepuszcza tę odpowiedź przez tzw. Dekoder (zazwyczaj oparty na bibliotece Jackson). Dekoder automatycznie parsuje tekstowy JSON i zamienia go na gotowe obiekty Java (ItunesResponse). Nie musisz ręcznie mapować pól.
3. Wstrzykiwanie i Wywołanie (Magia Springa)
W swojej usłudze po prostu wstrzykujesz ten interfejs i używasz go jak zwykłej metody Java:
@Service
public class SabatonService {

    private final ITunesClient iTunesClient;

    public SabatonService(ITunesClient iTunesClient) {
        this.iTunesClient = iTunesClient;
    }

    public void printSabatonInfo() {
        // Wywołujemy metodę z interfejsu - OpenFeign robi resztę!
        ItunesResponse response = iTunesClient.searchArtist("sabaton", "musicArtist");
        
        response.results().forEach(artist -> 
            System.out.println("Zespół: " + artist.artistName() + 
                               " | Gatunek: " + artist.primaryGenreName())
        );
    }
}

Jak to działa pod spodem?
Podczas startu aplikacji Spring (dzięki adnotacji @EnableFeignClients w głównej klasie) skanuje projekt w poszukiwaniu interfejsów Feign. Następnie używa mechanizmu Dynamic Proxy. Tworzy "niewidzialną" klasę, która implementuje Twój interfejs ITunesClient. Kiedy w SabatonService wywołujesz searchArtist, w rzeczywistości odpalasz kod wygenerowany przez Feigna, który otwiera połączenie sieciowe, pobiera dane i je zwraca.
Podsumowanie działania OpenFeign:
 * Zdejmuje z Ciebie ciężar kodu: Piszesz tylko interfejs (CO chcesz pobrać), a OpenFeign zajmuje się implementacją (JAK to pobrać).
 * Tłumaczy Javę na HTTP: Zamienia argumenty metody na parametry URL, nagłówki i body zapytania.
 * Tłumaczy HTTP na Javę: Automatycznie konwertuje odpowiedź (JSON z iTunes) na obiekty Javy.
 * Łatwa obsługa błędów: Za pomocą mechanizmu ErrorDecoder możesz łatwo zdefiniować, co ma się stać, jeśli iTunes API zwróci np. błąd 404 lub 429 (Too Many

 Jasne! Oto kompletny przykład aplikacji Spring Boot, która wykorzystuje OpenFeign do pobrania danych zespołu Sabaton z iTunes API, wraz z przydatną konfiguracją: obsługą błędów (ErrorDecoder) oraz logowaniem zapytań HTTP.
1. Zależności (pom.xml)
Aby używać OpenFeign, musisz dodać odpowiednie zależności w swoim pliku Maven. Upewnij się, że masz zarządzanie zależnościami Spring Cloud (BOM).
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2023.0.0</version> <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

2. Konfiguracja aplikacji (application.yml)
Aby zobaczyć "magię" Feigna w konsoli, włączmy logowanie zapytań na poziomie DEBUG.
logging:
  level:
    # Podaj tu pakiet, w którym znajduje się Twój interfejs klienta Feign
    com.twojprojekt.itunes: DEBUG

spring:
  cloud:
    openfeign:
      client:
        config:
          default:
            loggerLevel: FULL # Loguje pełne nagłówki, body i adresy URL

3. Klasa główna i Modele DTO
Włączamy Feigna za pomocą adnotacji @EnableFeignClients i definiujemy rekordy do odebrania JSON-a.
package com.twojprojekt.itunes;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.openfeign.EnableFeignClients;

import java.util.List;

@SpringBootApplication
@EnableFeignClients
public class SabatonItunesApplication {
    public static void main(String[] args) {
        SpringApplication.run(SabatonItunesApplication.class, args);
    }
}

// Modele DTO reprezentujące odpowiedź z iTunes
record ItunesResponse(Integer resultCount, List<Artist> results) {}
record Artist(String artistName, String primaryGenreName, String artistLinkUrl, Long artistId) {}

4. Obsługa błędów (ErrorDecoder)
Zanim napiszemy samego klienta, stwórzmy klasę, która przechwyci błędy HTTP (np. 400 lub 500) zwrócone przez API iTunes i zamieni je na czytelne wyjątki w naszej aplikacji.
package com.twojprojekt.itunes;

import feign.Response;
import feign.codec.ErrorDecoder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
class FeignConfig {

    @Bean
    public ErrorDecoder errorDecoder() {
        return new CustomErrorDecoder();
    }
}

class CustomErrorDecoder implements ErrorDecoder {
    private final ErrorDecoder defaultErrorDecoder = new Default();

    @Override
    public Exception decode(String methodKey, Response response) {
        if (response.status() >= 400 && response.status() <= 499) {
            return new RuntimeException("Błąd klienta: iTunes API zwróciło status " + response.status());
        }
        if (response.status() >= 500 && response.status() <= 599) {
            return new RuntimeException("Błąd serwera: iTunes API jest niedostępne (status " + response.status() + ")");
        }
        return defaultErrorDecoder.decode(methodKey, response);
    }
}

5. Interfejs Klienta OpenFeign
Teraz definiujemy sam interfejs, wskazując mu naszą konfigurację.
package com.twojprojekt.itunes;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;

// configuration = FeignConfig.class podpina nasz ErrorDecoder
@FeignClient(name = "itunesClient", url = "https://itunes.apple.com", configuration = FeignConfig.class)
public interface ITunesClient {

    @GetMapping("/search")
    ItunesResponse searchArtist(@RequestParam("term") String term, 
                                @RequestParam("entity") String entity);
}

6. Serwis i Uruchomienie (CommandLineRunner)
Abyś od razu po uruchomieniu aplikacji zobaczył efekt w konsoli, stworzymy komponent implementujący CommandLineRunner. Wykona się on automatycznie podczas startu Springa.
package com.twojprojekt.itunes;

import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Service;

@Service
public class SabatonService implements CommandLineRunner {

    private final ITunesClient iTunesClient;

    public SabatonService(ITunesClient iTunesClient) {
        this.iTunesClient = iTunesClient;
    }

    @Override
    public void run(String... args) {
        System.out.println("=== ROZPOCZYNAM POBIERANIE DANYCH Z ITUNES API ===");

        try {
            // Wywołujemy OpenFeign!
            ItunesResponse response = iTunesClient.searchArtist("sabaton", "musicArtist");

            if (response.resultCount() > 0) {
                System.out.println("Znaleziono artystów: " + response.resultCount());
                response.results().forEach(artist -> {
                    System.out.println("-------------------------------------------------");
                    System.out.println("Nazwa zespołu : " + artist.artistName());
                    System.out.println("Główny gatunek: " + artist.primaryGenreName());
                    System.out.println("ID iTunes     : " + artist.artistId());
                    System.out.println("Link do Apple : " + artist.artistLinkUrl());
                });
                System.out.println("-------------------------------------------------");
            } else {
                System.out.println("Nie znaleziono zespołu Sabaton :(");
            }
            
        } catch (Exception e) {
            System.err.println("Wystąpił błąd podczas komunikacji z API: " + e.getMessage());
        }
        
        System.out.println("=== ZAKOŃCZONO POBIERANIE DANYCH ===");
    }
}

Gdy uruchomisz tę aplikację, w konsoli zobaczysz logi wygenerowane przez Feigna (Dzięki loggerLevel: FULL zobaczysz całe zapytanie GET wysłane do Apple), a następnie ładnie sformatowane dane zespołu Sabaton.

