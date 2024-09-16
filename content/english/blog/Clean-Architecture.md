---
title: "Vad är Clean Architecture?"
meta_title: ""
description: "Vad är Clean Architecture?"
date: 2024-09-15
image: "/images/clean-architecture-diagram"
categories: ["Architecture"]
author: "Patric Bergkvist"
tags: ["Architecture"]
draft: false
---

Vad händer när ditt system växer snabbt och koden börjar bli oöverskådlig?

Applikationer som utvecklas snabbt kan snabbt bli röriga, svåra att underhålla och fulla av beroenden som är svåra att förstå eller förändra. Det är här Clean Architecture kommer in i bilden. Clean Architecture är mer än bara en uppsättning regler – det är ett sätt att tänka kring systemdesign som hjälper dig att skapa skalbara, testbara och lättunderhållna applikationer. Genom att separera ansvar och hålla din kodbas strukturerad, kan du enklare bygga system som står emot tidens tand.

I det här inlägget kommer vi att utforska principerna bakom Clean Architecture, hur det fungerar i praktiken och varför det kan vara nyckeln till framgång i dina framtida projekt.

### Vad är då det här Clean Architecture?

Clean Architecture introducerades av ingen mindre än Uncle Bob (Robert C. Martin), och det är ett sätt att strukturera arkitekturen för mjukvaran du bygger så den blir mer robust och flexibel. Tanken är att separera ansvar och minska beroenden mellan de olika delarna av systemet som exempelvis databas och affärslogik. Målet är att man skapar en arkitektur där affärslogik och komponenter kan förändras, eller till och med bytas ut utan att det ska behöva påverka hela systemet.

### Principerna bakom Clean Architecture

#### Separation av ansvar
En av de absolut viktigaste elementen i Clean Architecture är separation av ansvar mellan olika lager (mer om lagren längre ner). Varje lager i systemet har ett specifikt syfte och är inte beroende av något annat lager i system för sin funktionalitet. Genom att göra detta så minskar kopplingarna mellan olika delar av systemet, vilket gör det lättare att utveckla, testa och underhålla.

#### Oberoende av teknologier
En annan nyckelprincip är att affärslogiken ska vara oberoende av teknologier som ramverk, databaser och användargränssnitt. Med Clean Architecture kan du byta ut dessa "yttre" teknologier utan att påverka kärnan i applikationen. Du kan alltid använda de senaste teknikerna utan att behöva omstrukturera systemets logik. Ett mer troligt scenario är att om en viss teknologi inte längre passar din applikation, kan du enkelt byta ut den utan att behöva röra kärnlogiken.

#### Testbarhet
Eftersom det är en tydlig ansvartfördelning mellan de olika lagren samt att de är oberoende av varandra, blir det enkelt att enhetstesta varje lager separat. Detta leder till mer robusta tester och högre kodkvalitet.

### Lagren i Clean Architecture
Clean Architecture består av flera lager som jag nämnt tidigare i detta inlägg, och i den vanligaste implementationen i .NET ser vi ofta följande fyra:

- Domain
- Application
- Infrastructure
- Presentation

#### Domain
I detta lager ligger all domänspecifik kod, inklusive affärslogik och entiteter. Det är kärnan (varav många väljer att döpa lagret till **Core**) i applikationen, oberoende av yttre faktorer som databaser och användargränssnitt. Detta lager är applikationens "sanning", där affärsregler definieras.

#### Application
Här finns abstraktioner och interface som definierar de olika användningsfallen (varav många väljer att döpa lagret till **UseCases**). Detta lager hanterar applikationens logik och interagerar med domänlagret, men är fortfarande oberoende av specifika tekniska implementationer, som databaser eller API.

#### Infrastructure

Allting som rör tekniska beroenden placeras i detta lager, såsom databastransaktioner eller externa API-anrop. Om du använder Entity Framework eller någon annan databaslösning, ligger dessa implementationer i Infrastructure-lagret.

#### Presentation

Detta lager hanterar användargränssnittet och API. Ett typiskt exempel är ett web API som exponerar data till en front-end applikation eller en mobil app. Presentation-lagret interagerar med applikationslagret för att få tillgång till affärslogik och data.

![Clean Architecture Diagram](/images/clean-architecture-diagram.png)

### Exempel

Om vi tänker oss att vi bygger en e-bokhandel, så skulle affärslogiken ligga i Domain lagret. En entitet Book:
```csharp
// I Domain-lagret definieras affärslogiken. Här är ett exempel på en bokentitet.
public class Book 
{
    public Guid Id { get; init; }
    public string Title { get; init; }
    public double Price { get; init; }
    public int StockQuantity { get; init; }
}
```
Här har vi själva kärnan i systemet.

Application lagret kommer tillföra funktioner för att lägga en bok i varukorgen samt genomföra en ett köp. 

```csharp
// Application-lagret innehåller tjänster som hanterar affärslogiken och interagerar med domänmodellerna. Här är ett exempel på en tjänst för att hantera bokbeställningar.
public class OrderService : IOrderService
{
    private readonly IBookRepository _bookRepository;

    public OrderService(IBookRepository bookRepository)
    {
        _bookRepository = bookRepository;
    }

    public async Task<Receipt> CompleteOrder(Book order)
    {
        var book = await _bookRepository.GetById(book.Id);

        // resterande logik..
    }
}

public interface IBookRepository
{
    Task<Book> GetById(Guid id);
    // resterande...
}

```

Infrastructure lagret har implementationen för Entity frameork

```csharp
    // Infrastructure-lagret implementerar tekniska detaljer som databasåtkomst. Här är ett exempel på ett repository som använder Entity Framework.
    public class BookRepository : IBookRepository
    {
        private readonly ApplicationContext _context;

        public BookRepository(ApplicationContext context)
        {
            _context = context;
        }

        public async Task<Book> GetById(Guid id)
        {
            return await _context.Books.SingleOrDefault(book => book.Id == id);
        }
    }
```

Skulle vi i detta exempel vilja byta ut entityframework mot något annat ORM så kan vi göra det utan att behöva skriva om koden i Domain eller Application lagret. 
Det enda vi behöver göra är att skriva om koden i Infrastructure lagret. 

### Sammanfattning

Clean Architecture hjälper dig att bygga flexibla och underhållsbara applikationer som enkelt kan anpassas till nya krav. Genom att separera affärslogik från tekniska beroenden får du en struktur som är både testbar och robust mot förändringar i framtiden. Genom att använda de principer och lager som jag nämnt ovan, så kan du skapa skalbara och framtidssäkra system.