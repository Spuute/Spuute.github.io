---
title: "Skapa en generisk Enum select i Blazor"
meta_title: ""
description: "Generisk Enum select i Blazor"
date: 2024-09-28
image: "/images/clean-architecture-diagram"
categories: ["Blazor"]
author: "Patric Bergkvist"
tags: ["Blazor"]
draft: true
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

och för att få namnet från [Display]-attributet kan du skapa en hjälpfunktion:
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

--SCREENSHOT--

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
                    <MudSelect T="ProjectRole" @bind-Value="Employee.Role" Label="Ange roll" ToStringFunc="GetAllWithoutDefaultValue">
                        @foreach (var role in Enum.GetValues(typeof(ProjectRole)).Cast<ProjectRole>().Where(e => !e.Equals(default(ProjectRole))))
                        {
                            <MudSelectItem Value="@role">@role</MudSelectItem>
                        }
                    </MudSelect>
                    <MudSelect T="Department" @bind-Value="Employee.Department" Label="Ange avdelning" ToStringFunc="GetAllWithoutDefaultValue">
                        @foreach (var department in Enum.GetValues(typeof(Department)).Cast<Department>().Where(e => !e.Equals(default(Department))))
                        {
                            <MudSelectItem Value="@department">@department</MudSelectItem>
                        }
                    </MudSelect>
                    <MudSelect T="EmployeeStatus" @bind-Value="Employee.Status" Label="Ange status" ToStringFunc="GetAllWithoutDefaultValue">
                        @foreach (var status in Enum.GetValues(typeof(EmployeeStatus)).Cast<EmployeeStatus>().Where(e => !e.Equals(default(EmployeeStatus))))
                        {
                            <MudSelectItem Value="@status">@status</MudSelectItem>
                        }
                    </MudSelect>
                    <MudSelect T="SeniorityLevel" @bind-Value="Employee.SeniorityLevel" Label="Ange senioritet" ToStringFunc="GetAllWithoutDefaultValue">
                        @foreach (var seniorityLevel in Enum.GetValues(typeof(SeniorityLevel)).Cast<SeniorityLevel>().Where(e => !e.Equals(default(SeniorityLevel))))
                        {
                            <MudSelectItem Value="@seniorityLevel">@seniorityLevel</MudSelectItem>
                        }
                    </MudSelect>
                    <MudSelect T="Availability" @bind-Value="Employee.Availability" Label="Ange tillgänglighet" ToStringFunc="GetAllWithoutDefaultValue">
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

    private static string GetAllWithoutDefaultValue(ProjectRole enumValue)
        => enumValue == 0 ? string.Empty : GetDisplayName(enumValue);

    private static string GetAllWithoutDefaultValue(Department enumValue)
        => enumValue == 0 ? string.Empty : GetDisplayName(enumValue);

    private static string GetAllWithoutDefaultValue(EmployeeStatus enumValue)
        => enumValue == 0 ? string.Empty : GetDisplayName(enumValue);

    private static string GetAllWithoutDefaultValue(SeniorityLevel enumValue)
        => enumValue == 0 ? string.Empty : GetDisplayName(enumValue);

    private static string GetAllWithoutDefaultValue(Availability enumValue)
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
I detta kodexempel används MudSelect-komponenter för att skapa dropdown-menyer som tillåter användaren att välja värden från de enum-typer vi definierat tidigare. För att förbättra användarupplevelsen och säkerställa att ogiltiga val inte kan göras, använder vi flera tekniker såsom ToStringFunc och filtrering av ogiltiga enum-värden.

När vi renderar komponenten så kommer employee ha default värde av den anledning jag förklarat tidigare gällande hur enum fungerar. Vi använder MudBlazors inbyggda funktion ToStringFunc för att styra vad som visas i dropdownen. Genom att använda den kan vi dynamiskt returnera en tom sträng (string.Empty) när enum-värdet är InvalidState, vilket gör att label-texten ("Ange roll", "Ange avdelning", etc.) visas istället för ett ogiltigt värde. Dessutom filtrerar vi bort InvalidState i foreach-loopen så att det inte visas som ett alternativ i listan och därmed inte kan väljas av misstag.