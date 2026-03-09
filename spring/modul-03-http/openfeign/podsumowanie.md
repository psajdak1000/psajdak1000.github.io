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
 * Łatwa obsługa błędów: Za pomocą mechanizmu ErrorDecoder możesz łatwo zdefiniować, co ma się stać, jeśli iTunes API zwróci np. błąd 404 lub 429 (Too Many Requests).
Czy chciałbyś, abym przygotował dla Ciebie pełny, gotowy do uruchomienia kod takiej aplikacji w Spring Boot, uwzględniający np. obsługę potencjalnych błędów z API iTunes?
