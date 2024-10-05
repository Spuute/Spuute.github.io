---
title: "Skapa en generisk Enum select i Blazor"
meta_title: ""
description: "Generisk Enum select i Blazor"
date: 2024-10-05
image: "/images/inputform"
categories: ["Blazor"]
author: "Patric Bergkvist"
tags: ["Blazor"]
draft: false
---

Har du ett formulär som växer och innehåller många fält baserade på enums? Då kan det vara dags att refaktorera och optimera din kod genom att skapa en generisk dropdown-komponent för enums. Detta kan förenkla din kod avsevärt och eliminera duplicering.

Jag har skapat ett exempel för att visa hur du kan förbättra koden. Föreställ dig att vi bygger ett internt system för ett konsultbolag. När företaget anställer en ny konsult ska denne registreras i systemet. Förutom grundläggande information som förnamn och efternamn vill företaget även samla in detaljer som roll, avdelning, konsultstatus, senioritet och tillgänglighet.

För enkelhetens skull har jag skalat bort funktionalitet som att spara data till en databas, då detta ligger utanför scopet för inlägget. Vi använder MudBlazor, ett populärt komponentbibliotek för Blazor, för att skapa formulärets användargränssnitt.

### Definiera Employee-klassen
Vi börjar med att definiera informationen vi behöver för vår Employee-klass. Den innehåller ett unikt ID (en Guid), förnamn och efternamn som strängar. Roll, avdelning, status, senioritet och tillgänglighet är fördefinierade värden inom företaget, och dessa definierar vi som enums.

```csharp
public enum ProjectRole
{
    [Display(Name = "Ogiltligt state")]
    InvalidState = 0,
    [Display(Name = "Utvecklare")]
    Developer = 1,
    [Display(Name = "UI/UX Designer")]
    Designer = 2,
    [Display(Name = "Testare")]
    Tester = 3,
    [Display(Name = "Projektledare")]
    ProjectManager = 4
}

public enum Department
{
    [Display(Name = "Ogiltligt state")]
    InvalidState = 0,
    [Display(Name = "IT")]
    IT = 1,
    [Display(Name = "HR")]
    HR = 2,
    [Display(Name = "Finans")]
    Finance = 3,
    [Display(Name = "Marknadsföring")]
    Marketing = 4
}

public enum EmployeeStatus
{
    [Display(Name = "Ogiltligt tillstånd")]
    InvalidState = 0,
    [Display(Name = "Tillgänglig")]
    Available = 1,
    [Display(Name = "På ledighet")]
    OnLeave = 2,
    [Display(Name = "Inte tillgänglig")]
    NotAvailable = 3
}

public enum SeniorityLevel
{
    [Display(Name = "Ogiltligt state")]
    InvalidState = 0,
    [Display(Name = "Junior")]
    Junior = 1,
    [Display(Name = "Mid")]
    Mid = 2,
    [Display(Name = "Senior")]
    Senior = 3,
    [Display(Name = "Lead")]
    Lead = 4
}

public enum Availability
{
    [Display(Name = "Ogiltligt tillstånd")]
    InvalidState = 0,
    [Display(Name = "25%")]
    TwentyFivePercent = 1,
    [Display(Name = "50%")]
    FiftyPercent = 2,
    [Display(Name = "75%")]
    SeventyFivePercent = 3,
    [Display(Name = "100%")]
    HundredPercent = 4
}
```
### Hantering av ogiltiga enum-värden
När en enum initialiseras sätts automatiskt det första värdet som default (index 0). Av den anledningen föredrar jag att alltid inkludera ett värde som InvalidState i början. Detta säkerställer att inga ogiltiga värden oavsiktligt lagras i systemet som giltiga alternativ.

Att använda ett "ogiltigt" enum-värde som standard ger oss också möjligheten att lägga till extra valideringslogik i systemet. Om ett felaktigt värde som InvalidState skickas, kan vi blockera åtgärden, visa ett felmeddelande, eller logga problemet för att undvika att felaktig data når databasen eller externa tjänster.

Detta tillvägagångssätt gör systemet mer robust eftersom vi tidigt kan upptäcka felaktigheter i datainmatningen och stoppa dessa innan de leder till problem längre fram. Det gör också koden mer självdokumenterande och enkel att underhålla.

### Exempel på valideringslogik
Här är ett exempel på hur du kan kontrollera att en giltig roll har valts innan data skickas vidare:
```csharp
if (employee.Role == ProjectRole.InvalidState)
{
  throw new InvalidOperationException("Ogiltlig roll: En konsult måste ha en giltlig roll innan den kan sparas.");
}
```

### Varför använda [Display]-attributet?
[Display]-attributet är särskilt användbart när du vill visa användarvänliga namn för dina enum-värden i användargränssnittet. I stället för att visa interna kodnamn som **Developer** eller TwentyFivePercent, kan du ange tydligare och mer beskrivande strängar som exempelvis "Utvecklare" eller "25%". Det gör formulär mer intuitiva och lätta att förstå för användarna.

När du använder MudBlazor eller andra komponentbibliotek som binder en enum till en UI-komponent, som en dropdown, kan [Display]-attributet användas för att visa det beskrivande namnet. Detta ger bättre användarupplevelse eftersom vi kan visa anpassade eller lokaliserade värden till användarna utan att kompromissa med strukturen i koden.

Exempelvis:
```html
<MudSelect T="ProjectRole" @bind-Value="Employee.Role" Label="Ange roll" ToStringFunc="GetAllWithoutDefaultValue">
  @foreach (var role in Enum.GetValues(typeof(ProjectRole)).Cast<ProjectRole>().Where(e => !e.Equals(default(ProjectRole))))
  {
    <MudSelectItem Value="@role">@GetDisplayName(role)</MudSelectItem>
  }
</MudSelect>
```

och för att få namnet från [Display]-attributet kan du skapa en hjälpmetod:
```csharp
private static string GetDisplayName(ProjectRole enumValue)
{
    var displayAttribute = enumValue.GetType()
        .GetMember(enumValue.ToString())
        .FirstOrDefault()?
        .GetCustomAttributes(false)
        .OfType<DisplayAttribute>()
        .FirstOrDefault();

    return displayAttribute?.Name ?? enumValue.ToString();
}
```

Att använda [Display]-attributet säkerställer också att om företaget vill ändra hur något visas för användarna – exempelvis ändra från "Junior" till "Nyexaminerad" – kan detta göras centralt i enum-definitionen. Det eliminerar behovet av att ändra texten på flera ställen i systemet, vilket är särskilt användbart om du har en metod för att översätta enums som används på olika platser i systemet, såsom en enum-översättare:

```csharp
public static string TranslateEnum(ProjectRole enumValue)
{
    return enumValue switch
    {
        ProjectRole.Developer => "Utvecklare",
        ProjectRole.Tester => "Testare",
        // och så vidare...
    };
}
```

### Skapa formulär för att lägga till ny anställd
Nu är det dags att skapa en ny sida för detta formulär. Vi börjar med att skapa en ny ***page***-komponent i Pages-mappen och namnge den AddEmployee.razor. Denna komponent kommer att innehålla vårt formulär för att registrera nya anställda. Formuläret kommer att se ut enligt följande:

![Inputform](/images/inputform.png)

Och koden för den har vi här:

```csharp
@page "/add-resource"

<MudContainer>
    <MudPaper Elevation="4" Class="pa-4 my-4">
        <MudGrid>
            <MudItem sm="6">
                <MudText Typo="Typo.h3">Registrera ny anställd</MudText>
                <MudText Typo="Typo.body1" Class="mt-3">
                    Vänligen fyll i informationen för att lägga till en ny anställd i systemet.
                    Du kan ange för- och efternamn, samt välja roll, avdelning, status och senioritetsnivå från listorna.
                    Du kan också specificera hur stor arbetsbelastning den anställde förväntas ha inom sitt tilldelade projekt.
                    När all information är korrekt ifylld, klicka på "Spara" för att slutföra registreringen.
                </MudText>
            </MudItem>
            <MudItem sm="6" Class="p-4 mt-4">
                <EditForm Spacing="6" Model="Employee">
                    <MudGrid>
                        <MudItem sm="6">
                            <MudTextField T="string" @bind-Value="Employee.FirstName" Label="Förnamn"/>
                        </MudItem>
                        <MudItem sm="6">
                            <MudTextField T="string" @bind-Value="Employee.Lastname" Label="Efternamn"/>
                        </MudItem>
                    </MudGrid>
                    <MudSelect T="ProjectRole" @bind-Value="Employee.Role" Label="Ange roll" ToStringFunc="GetUserFriendlyTextWithoutDefaultValue">
                        @foreach (var role in Enum.GetValues(typeof(ProjectRole)).Cast<ProjectRole>().Where(e => !e.Equals(default(ProjectRole))))
                        {
                            <MudSelectItem Value="@role">@role</MudSelectItem>
                        }
                    </MudSelect>
                    <MudSelect T="Department" @bind-Value="Employee.Department" Label="Ange avdelning" ToStringFunc="GetUserFriendlyTextWithoutDefaultValue">
                        @foreach (var department in Enum.GetValues(typeof(Department)).Cast<Department>().Where(e => !e.Equals(default(Department))))
                        {
                            <MudSelectItem Value="@department">@department</MudSelectItem>
                        }
                    </MudSelect>
                    <MudSelect T="EmployeeStatus" @bind-Value="Employee.Status" Label="Ange status" ToStringFunc="GetUserFriendlyTextWithoutDefaultValue">
                        @foreach (var status in Enum.GetValues(typeof(EmployeeStatus)).Cast<EmployeeStatus>().Where(e => !e.Equals(default(EmployeeStatus))))
                        {
                            <MudSelectItem Value="@status">@status</MudSelectItem>
                        }
                    </MudSelect>
                    <MudSelect T="SeniorityLevel" @bind-Value="Employee.SeniorityLevel" Label="Ange senioritet" ToStringFunc="GetUserFriendlyTextWithoutDefaultValue">
                        @foreach (var seniorityLevel in Enum.GetValues(typeof(SeniorityLevel)).Cast<SeniorityLevel>().Where(e => !e.Equals(default(SeniorityLevel))))
                        {
                            <MudSelectItem Value="@seniorityLevel">@seniorityLevel</MudSelectItem>
                        }
                    </MudSelect>
                    <MudSelect T="Availability" @bind-Value="Employee.Availability" Label="Ange tillgänglighet" ToStringFunc="GetUserFriendlyTextWithoutDefaultValue">
                        @foreach (var workload in Enum.GetValues(typeof(Availability)).Cast<Availability>().Where(e => !e.Equals(default(Availability))))
                        {
                            <MudSelectItem Value="@workload">@workload</MudSelectItem>
                        }
                    </MudSelect>
                    <MudGrid>
                        <MudItem sm="12" Class="d-flex justify-end">
                            <MudButton Variant="Variant.Filled" Color="Color.Primary" Class="mt-4">Spara</MudButton>
                        </MudItem>
                    </MudGrid>

                </EditForm>
            </MudItem>
        </MudGrid>
    </MudPaper>
</MudContainer>

@code {
    private Employee Employee { get; set; } = new();

    private static string GetUserFriendlyTextWithoutDefaultValue(ProjectRole enumValue)
        => enumValue == 0 ? string.Empty : GetDisplayName(enumValue);

    private static string GetUserFriendlyTextWithoutDefaultValue(Department enumValue)
        => enumValue == 0 ? string.Empty : GetDisplayName(enumValue);

    private static string GetUserFriendlyTextWithoutDefaultValue(EmployeeStatus enumValue)
        => enumValue == 0 ? string.Empty : GetDisplayName(enumValue);

    private static string GetUserFriendlyTextWithoutDefaultValue(SeniorityLevel enumValue)
        => enumValue == 0 ? string.Empty : GetDisplayName(enumValue);

    private static string GetUserFriendlyTextWithoutDefaultValue(Availability enumValue)
        => enumValue == 0 ? string.Empty : GetDisplayName(enumValue);

    private static string GetDisplayName(ProjectRole enumValue)
    {
        var displayAttribute = enumValue.GetType()
            .GetMember(enumValue.ToString())
            .FirstOrDefault()?
            .GetCustomAttributes(false)
            .OfType<DisplayAttribute>()
            .FirstOrDefault();

        return displayAttribute?.Name ?? enumValue.ToString();
    }

    private static string GetDisplayName(Department enumValue)
    {
        var displayAttribute = enumValue.GetType()
            .GetMember(enumValue.ToString())
            .FirstOrDefault()?
            .GetCustomAttributes(false)
            .OfType<DisplayAttribute>()
            .FirstOrDefault();

        return displayAttribute?.Name ?? enumValue.ToString();
    }

    private static string GetDisplayName(EmployeeStatus enumValue)
    {
        var displayAttribute = enumValue.GetType()
            .GetMember(enumValue.ToString())
            .FirstOrDefault()?
            .GetCustomAttributes(false)
            .OfType<DisplayAttribute>()
            .FirstOrDefault();

        return displayAttribute?.Name ?? enumValue.ToString();
    }

    private static string GetDisplayName(SeniorityLevel enumValue)
    {
        var displayAttribute = enumValue.GetType()
            .GetMember(enumValue.ToString())
            .FirstOrDefault()?
            .GetCustomAttributes(false)
            .OfType<DisplayAttribute>()
            .FirstOrDefault();

        return displayAttribute?.Name ?? enumValue.ToString();
    }

    private static string GetDisplayName(Availability enumValue)
    {
        var displayAttribute = enumValue.GetType()
            .GetMember(enumValue.ToString())
            .FirstOrDefault()?
            .GetCustomAttributes(false)
            .OfType<DisplayAttribute>()
            .FirstOrDefault();

        return displayAttribute?.Name ?? enumValue.ToString();
    }
}
        
```

När vi renderar komponenten så kommer employee ha default värde av tidigare i inlägget nämnd anledning gällande hur enums fungerar. Vi använder MudBlazors inbyggda funktion ToStringFunc för att styra vad som visas i dropdownen. Genom att använda den kan vi dynamiskt returnera en tom sträng (string.Empty) när enum-värdet är InvalidState, vilket gör att label-texten ("Ange roll", "Ange avdelning", etc.) visas istället för InvalidState. Dessutom filtrerar vi bort InvalidState i foreach-loopen så att det inte visas som ett alternativ i listan och därmed inte kan väljas av misstag.

### Refaktorering

Nu när vi har en färdig komponent så kan vi se tydligt att det är väldigt mycket duplicering av kod, och visst kan man bryta ner varje metod som tar emot en generisk enum istället för varje specifik enum, men vi kan göra det ännu bättre och skapa en generisk enum-select att använda oss av. 

Vi börjar med att skapa en ny mapp och döper den till ReusableComponents, i den skapar vi en ny komponent som vi döper till EnumSelect.razor

Vi lägger in följande kod i EnumSelect.razor:

```csharp
@typeparam TEnum where TEnum : struct, Enum

<MudSelect T="TEnum" Label="@Label" ValueChanged="OnSelectedEnumChanged" ToStringFunc="GetUserFriendlyTextWithoutDefaultValue">
    @foreach (var enumValue in GetAllValuesExceptDefault())
    {
        <MudSelectItem Value="enumValue">@GetDisplayName(enumValue)</MudSelectItem>
    }
</MudSelect> 

@code {
    [Parameter] public required TEnum SelectedEnum { get; set; }
    [Parameter] public required EventCallback<TEnum> SelectedEnumChanged { get; set; }
    [Parameter] public required string Label { get; set; } 

    private async void OnSelectedEnumChanged(TEnum enumValue)
    {
        if (!Enum.IsDefined(typeof(TEnum), enumValue))
        {
            throw new ArgumentOutOfRangeException("Enumvärdet är ej definerat");
        }
        
        SelectedEnum = enumValue;
        await SelectedEnumChanged.InvokeAsync(enumValue);
    }

    private static IEnumerable<TEnum> GetAllValuesExceptDefault()
        => Enum.GetValues(typeof(TEnum)).Cast<TEnum>().Where(e => !e.Equals(default(TEnum)));
    
    private static string GetDisplayName(TEnum enumValue)
    {
        var displayAttribute = enumValue.GetType()
            .GetMember(enumValue.ToString())
            .FirstOrDefault()?
            .GetCustomAttributes(false)
            .OfType<DisplayAttribute>()
            .FirstOrDefault();

        if (displayAttribute is null)
        {
            return string.Empty;
        }

        return displayAttribute.Name ?? enumValue.ToString();
    }
    
    private static string GetUserFriendlyTextWithoutDefaultValue(TEnum enumValue)
        => enumValue.Equals(default(TEnum)) ? string.Empty : GetDisplayName(enumValue);
}
```

När vi i Blazor skapar en generisk komponent behöver vi specificera vilken typ vi vill arbeta med på ett dynamiskt sätt. Målet med denna komponent är att kunna använda vilken enum som helst från vårt system, utan att behöva skapa en separat MudSelect för varje enum och loopa igenom dess värden.

### Vad är @typeparam?

@typeparam låter oss definiera en generisk typ för en Blazor-komponent, vilket gör att vi kan återanvända komponenten för olika typer utan att duplicera kod. I vårt fall är målet att komponenten ska kunna hantera vilken enum som helst, vilket är anledningen till att vi specificerar @typeparam TEnum.

Det är värt att notera att TEnum är namnet på typparametern, och den kan faktiskt heta vad som helst. Jag använder TEnum för att det tydligt signalerar att vi arbetar med en enum-typ, vilket gör koden lättare att förstå.

### Begränsningar med where TEnum : struct, Enum

Vi använder where TEnum : struct, Enum för att säkerställa att den generiska typen som skickas till komponenten måste vara en enum. Detta fungerar som en typbegränsning som gör att Blazor-komponenten endast accepterar enum-typer och inte andra datatyper som klasser eller gränssnitt. struct-kravet anger att det måste vara en värdetyp, och Enum specificerar att det specifikt måste vara en uppräkningstyp.

#### Varför är detta nödvändigt?

Eftersom vi vill skapa en generisk komponent som kan hantera olika typer av enums, måste vi använda generiska typer. Detta gör komponenten mycket mer flexibel och återanvändbar. Istället för att skriva en separat komponent för varje enum (t.ex. ProjectRole, Department, etc.), kan vi skapa en enda generisk komponent som fungerar för alla enums.

När du arbetar med generiska typer i C# och vill begränsa att din generiska parameter bara accepterar enum-typer (eller värdetyper i allmänhet), behöver du inkludera struct i where-villkoret. Detta beror på att en enum är en värdetyp och alla värdetyper i C# är derivater av struct. Här är några anledningar till varför du måste använda struct:

- **Begränsa typen till värdetyper (struct)**: I C# är en enum en värdetyp, och värdetyper är alltid struct. Genom att lägga till where TEnum : struct säkerställer du att den generiska typen måste vara en värdetyp (som enum), och inte en referenstyp (som klasser).
  
- **Undvika referenstyper**: Om du inte inkluderar struct i din typbegränsning kan den generiska typen acceptera både värdetyper och referenstyper. Det skulle innebära att din komponent kunde ta emot alla möjliga typer, vilket inte är vad du vill när du specifikt vill jobba med enums.

- **Specifik för enums**: En enum i C# är en speciell typ av struct, men det finns ingen direkt where TEnum : enum-begränsning i C#. Därför använder man struct-begränsningen tillsammans med Enum-baserade operationer för att hantera enum-typer korrekt.

Så här ser deklarationen ut:

```csharp
@typeparam TEnum where TEnum : struct, Enum
```
Vidare kan vi se att koden har blivit refaktorerad så vi nu har en metod för att hämta ett användarvänligt namn att visa, där vi plockar bort InvalidState till ToStringFunc i MudSelect. Vi har också en metod för att hämta DisplayName-attributet från enum
, vilket förenklar kodstrukturen.

Jag har även refaktorerat sättet vi hämtar alla enums förutom default-värdet som tidigare fanns i varje foreach-loop:

```csharp
@foreach (var department in Enum.GetValues(typeof(Department)).Cast<Department>().Where(e => !e.Equals(default(Department))))
```
Denna kod kan vara svår att förstå vid första ögonkastet, så vi har brutit ner den till en egen metod med ett beskrivande namn, vilket minskar komplexiteten:

```csharp
private static IEnumerable<TEnum> GetAllValuesExceptDefault()
        => Enum.GetValues(typeof(TEnum)).Cast<TEnum>().Where(e => !e.Equals(default(TEnum)));
```
Vi använder nu denna metod i loopen så här:

```csharp
@foreach (var enumValue in GetAllValuesExceptDefault())
```
Detta gör det mycket enklare att se och förstå vad som händer i denna foreach-loop.

Komponenten har följande parametrar:

```csharp
[Parameter] public required TEnum SelectedEnum { get; set; }
[Parameter] public required EventCallback<TEnum> SelectedEnumChanged { get; set; }
[Parameter] public required string Label { get; set; }
```

- **SelectedEnum:** En TEnum som binder det valda värdet som skickas in till komponenten.

- **SelectedEnumChanged:** En EventCallback<TEnum> som möjliggör tvåvägsbindning. När SelectedEnum ändras i komponenten, kommer det nya värdet att skickas tillbaka till den anropande komponenten. Det är viktigt att notera att denna parameter har samma namn som SelectedEnum, med tillägget "Changed", för att Blazor ska kunna hantera bindningen korrekt.

- **Label:** En string som används för att ange en Label för MudSelect-komponenten. Detta hjälper användarna att förstå vad de ska välja i dropdown-menyn.
Denna refaktorisering och tydliga parameterbeskrivningar förbättrar läsbarheten och användbarheten av komponenten.

- **required:** Genom att använda required säkerställer vi att dessa parameter alltid måste anges vid användning av komponenten.

### OnSelectedEnumChanged-metoden:

Denna metod hanterar förändringar av det valda enum-värdet när användaren gör ett val i dropdownen. Den anropas automatiskt vid ValueChanged-händelsen. Här är dess implementation:

```csharp
private async void OnSelectedEnumChanged(TEnum enumValue)
{
    if (!Enum.IsDefined(typeof(TEnum), enumValue))
    {
        throw new ArgumentOutOfRangeException("Enumvärdet är ej definerat");
    }
    
    SelectedEnum = enumValue;
    await SelectedEnumChanged.InvokeAsync(enumValue);
}
```

Validering: Först kontrollerar metoden om det nya enumValue är definierat i den aktuella enum-typen. Om värdet inte är definierat kastas ett ArgumentOutOfRangeException för att indikera att det valda värdet inte är giltigt.

Uppdatering av SelectedEnum: Om värdet är giltigt uppdateras SelectedEnum med det nya värdet.

Händelsetrigger: Slutligen anropas SelectedEnumChanged.InvokeAsync(enumValue) för att meddela den anropande komponenten om den nya värdet, vilket möjliggör tvåvägsbindning.
Genom att använda EventCallback<TEnum>-mönstret säkerställer vi att eventuella ändringar i enum-värdet återspeglas korrekt i den anropande komponenten, vilket ger en sömlös användarupplevelse.

### Använda vår generiska enum-select

Om vi går tillbaka till AddEmployee.razor och lägger in följande kod:

```csharp
@page "/add-resource"
@using BlazorPlaygroundWeb.ReusableComponents

<MudContainer>
    <MudPaper Elevation="4" Class="pa-4 my-4">
        <MudGrid>
            <MudItem sm="6">
                <MudText Typo="Typo.h3">Registrera ny anställd</MudText>
                <MudText Typo="Typo.body1" Class="mt-3">
                    Vänligen fyll i informationen för att lägga till en ny anställd i systemet.
                    Du kan ange för- och efternamn, samt välja roll, avdelning, status och senioritetsnivå från listorna.
                    Du kan också specificera hur stor arbetsbelastning den anställde förväntas ha inom sitt tilldelade projekt.
                    När all information är korrekt ifylld, klicka på "Spara" för att slutföra registreringen.
                    <br/><br/>
                    <MudText Typo="Typo.body2">
                        <b>
                            Namn:
                        </b>
                        @Employee.FirstName @Employee.Lastname
                    </MudText>
                    <MudText Typo="Typo.body2" Class="mt-3">
                        <b>
                            Roll:
                        </b>
                        @(Employee.Role == ProjectRole.InvalidState ? "Ingen roll tilldelad" : GetDisplayName(Employee.Role))
                    </MudText>
                    <MudText Typo="Typo.body2" Class="mt-3">
                        <b>
                            Avdelning:
                        </b>
                        @(Employee.Department == Department.InvalidState ? "Ingen avdelning tilldelad" : GetDisplayName(Employee.Department))
                    </MudText>
                    <MudText Typo="Typo.body2" Class="mt-3">
                        <b>
                            Status
                        </b>
                        @(Employee.Status == EmployeeStatus.InvalidState ? "Ingen status tilldelad" : GetDisplayName(Employee.Status))
                    </MudText>
                    <MudText Typo="Typo.body2" Class="mt-3">
                        <b>
                            Senioritet:
                        </b>
                        @(Employee.SeniorityLevel == SeniorityLevel.InvalidState ? "Ingen senioritet tilldelad" : GetDisplayName(Employee.SeniorityLevel))
                    </MudText>
                    <MudText Typo="Typo.body2" Class="mt-3">
                        <b>
                            Tillgänglighet:
                        </b>
                        @(Employee.Availability == Availability.InvalidState ? "Ingen tillgänglighet tilldelad" : GetDisplayName(Employee.Availability))
                    </MudText>
                </MudText>
            </MudItem>
            <MudItem sm="6" Class="p-4 mt-4">
                <EditForm Spacing="6" Model="Employee">
                    <MudGrid>
                        <MudItem sm="6">
                            <MudTextField T="string" @bind-Value="Employee.FirstName" Label="Förnamn"/>
                        </MudItem>
                        <MudItem sm="6">
                            <MudTextField T="string" @bind-Value="Employee.Lastname" Label="Efternamn"/>
                        </MudItem>
                    </MudGrid>
                    <EnumSelect TEnum="ProjectRole" Label="Ange roll" @bind-SelectedEnum="Employee.Role"/>
                    <EnumSelect TEnum="Department" Label="Ange avdelning" @bind-SelectedEnum="Employee.Department"/>
                    <EnumSelect TEnum="EmployeeStatus" Label="Ange status" @bind-SelectedEnum="Employee.Status"/>
                    <EnumSelect TEnum="SeniorityLevel" Label="Ange senioritet" @bind-SelectedEnum="Employee.SeniorityLevel"/>
                    <EnumSelect TEnum="Availability" Label="Ange tillgänglighet" @bind-SelectedEnum="Employee.Availability"/>
                    <MudGrid>
                        <MudItem sm="12" Class="d-flex justify-end">
                            <MudButton Variant="Variant.Filled" Color="Color.Primary" Class="mt-4">Spara</MudButton>
                        </MudItem>
                    </MudGrid>
                </EditForm>
            </MudItem>
        </MudGrid>
    </MudPaper>
</MudContainer>

@code {
    private Employee Employee { get; set; } = new();
    
    private static string GetDisplayName(Enum enumValue)
    {
        var displayAttribute = enumValue.GetType()
            .GetMember(enumValue.ToString())
            .FirstOrDefault()?
            .GetCustomAttributes(false)
            .OfType<DisplayAttribute>()
            .FirstOrDefault();

        if (displayAttribute is null)
        {
            return string.Empty;
        }

        return displayAttribute.Name ?? enumValue.ToString();
    }
}
```

Det jag vill notera här är att jag byggt ut den vänstra delen i kortet med att skriva ut vilket val man har gjort i varje dropdown för att demonstrera att tvåvägsbindningen fungerar som den ska. Detta är även anledningen till att GetDisplayName metoden ligger här. 

![Inputform](/images/form1.png)

### Förklaring av EnumSelect-komponenten
I vår AddEmployee.razor-fil har vi implementerat en ny generisk komponent, EnumSelect, som effektivt förenklar hanteringen av enums i vårt formulär. Istället för att skapa en separat MudSelect-komponent för varje enum (t.ex. ProjectRole, Department, EmployeeStatus, etc.) och därmed behöva loopa genom varje enum-värde, kan vi nu använda EnumSelect-komponenten för att göra denna process mer elegant och lättläst.

### Användning av EnumSelect

Vi använder EnumSelect på följande sätt:

```csharp
<EnumSelect TEnum="ProjectRole" Label="Ange roll" @bind-SelectedEnum="Employee.Role"/>
```
- **TEnum:** Vi specificerar vilken enum vi vill att komponenten ska använda. I detta fall anger vi ProjectRole.

- **Label:** Här definierar vi den text som ska visas som etikett för dropdown-menyn. I detta exempel använder vi "Ange roll".

- **@bind-SelectedEnum:** Vi binder vår Employee.Role-egenskap till SelectedEnum i komponenten. Detta möjliggör tvåvägsbindning, vilket innebär att när användaren väljer ett värde från dropdown-menyn, uppdateras Employee.Role automatiskt, och om Employee.Role ändras i koden kommer dropdown-menyn att återspegla detta val.

## Fördelar med vår generiska EnumSelect
- **Mindre kod:** Genom att använda EnumSelect kan vi avsevärt minska mängden kod i vår komponent, vilket gör den mer hanterbar och lättläst.

- **Återanvändbarhet:** Komponentens generiska natur gör den lätt att återanvända för olika enums, vilket ökar kodåteranvändningen i hela applikationen.

- **Tydlighet:** Användningen av en gemensam komponent för enum-val gör att vi kan tydliggöra syftet med varje dropdown-menyn på ett konsistent sätt.

### Sammanfattning
Genom att implementera EnumSelect har vi förenklat processen för att hantera enum-värden i vårt formulär. Denna lösning är både effektiv och elegant, vilket förbättrar kodens läsbarhet och underhållbarhet.