
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
