# Analyse-Resumer-du-projet-fil-rouge
Ce repository contient un resumer claire du projet fil-rouge fait en Dev

# 🏗️ AUDIT TECHNIQUE - Event Agenda API

## 📋 Table des matières
1. [Résumé du projet](#résumé-du-projet)
2. [Analyse de l'architecture](#analyse-de-larchitecture)
3. [Points forts](#points-forts)
4. [Points faibles / Axes d'amélioration](#points-faibles--axes-damélioration)
5. [Ordre d'implémentation recommandé](#ordre-dimplémentation-recommandé)
6. [Création du projet from scratch](#création-du-projet-from-scratch)
7. [Schéma explicatif de l'architecture](#schéma-explicatif-de-larchitecture)

---

## 📋 Résumé du projet

### Objectif principal
Application de gestion d'agenda d'événements permettant de :
- Créer, consulter, modifier et supprimer des événements
- Filtrer les événements par date ou plage de dates
- Catégoriser les événements
- Gérer des membres et une FAQ

### Problème résolu
Fournir une API backend robuste et maintenable pour gérer un système d'événements avec :
- Validation métier stricte (ex: événements créés minimum 1 jour à l'avance)
- Gestion centralisée des erreurs
- Séparation claire entre les responsabilités techniques et métier

### Stack technique
- **Backend Framework**: ASP.NET Core 10.0 (Web API)
- **Base de données**: MS SQL Server
- **ORM**: Entity Framework Core 10.0.2
- **Architecture**: Clean Architecture (4 couches)
- **Documentation API**: OpenAPI/Swagger avec Scalar
- **Containerisation**: Docker (Dockerfile inclus)
- **Sécurité**: Argon2 pour le hashing (via Soenneker.Hashing.Argon2)

### Type d'application
**API REST** - Backend pour applications clientes (web, mobile, desktop)

---

## 🏛️ Analyse de l'architecture

### Structure des couches

Le projet est organisé en **4 projets distincts** suivant la Clean Architecture :

```
Demo_WebAPI_EventAgenda
├── Domain                    ← Couche centrale (aucune dépendance)
├── ApplicationCore          ← Logique métier (dépend de Domain)
├── Infrastructure.Database  ← Accès données (dépend de Domain + ApplicationCore)
└── Presentation.WebAPI      ← API REST (dépend de tous)
```

#### 1️⃣ **Domain** (Noyau métier - 0 dépendance)
**Responsabilité** : Définir les concepts métier purs, indépendants de toute technologie.

**Contenu** :
```
Domain/
├── Models/                   ← Entités métier (AgendaEvent, EventCategory, Member, Faq)
└── BusinessExceptions/       ← Exceptions métier personnalisées
```

**Principes appliqués** :
- **DDD (Domain-Driven Design)** : 
  - Setters privés pour encapsulation forte
  - Validation dans les constructeurs
  - Méthodes métier pour les modifications (ex: `ChangeDate()`)
  - Constructeur privé sans paramètres pour EF Core
- **Entités riches** : Les objets contiennent leur propre logique de validation
- **Immutabilité partielle** : Modification contrôlée via méthodes dédiées

**Exemple de modèle** :
```csharp
public class AgendaEvent 
{
    public long Id { get; private set; }  // Setter privé
    public string Name { get; private set; }
    
    private AgendaEvent() { }  // Pour EF Core
    
    public AgendaEvent(string name, ...) 
    {
        if(string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("...");
        Name = name;
    }
    
    public AgendaEvent ChangeDate(DateTime start, DateTime? end) 
    {
        // Validation + modification
        return this; // Fluent interface
    }
}
```

#### 2️⃣ **ApplicationCore** (Logique applicative)
**Responsabilité** : Orchestrer les use cases métier et définir les contrats d'infrastructure.

**Contenu** :
```
ApplicationCore/
├── Interfaces/
│   ├── Services/         ← Contrats de services métier (IAgendaEventService)
│   └── Repositories/     ← Contrats de persistance (IAgendaEventRepository)
└── Services/             ← Implémentations des services
```

**Principes appliqués** :
- **Inversion de dépendance** : Les services dépendent d'interfaces, pas d'implémentations
- **Pattern Facade** : Services exposent une API simple aux couches externes
- **Séparation des préoccupations** : 
  - Les services orchestrent la logique métier
  - Les repositories gèrent la persistance
- **Gestion des règles métier** :
  - Validation : "Un événement doit être créé minimum 1 jour à l'avance"
  - Calculs : Pagination (offset/limit)
  - Exceptions métier lancées en cas de violation

**Exemple de service** :
```csharp
public class AgendaEventService : IAgendaEventService
{
    private IAgendaEventRepository _repo;  // Interface, pas implémentation
    
    public AgendaEvent Create(AgendaEvent data)
    {
        if (data.StartDate < DateTime.Today.AddDays(1))
            throw new AgendaEventCreateException(data);
            
        return _repo.Insert(data);
    }
}
```

#### 3️⃣ **Infrastructure.Database** (Accès aux données)
**Responsabilité** : Implémenter les contrats définis par ApplicationCore pour accéder à SQL Server.

**Contenu** :
```
Infrastructure.Database/
├── Configs/              ← Entity configurations (IEntityTypeConfiguration)
├── Repositories/         ← Implémentations des repositories
├── Migrations/           ← Migrations EF Core
└── AppDbContext.cs       ← DbContext principal
```

**Principes appliqués** :
- **Pattern Repository** : Abstraction de l'accès aux données
- **Entity Framework Core** : 
  - Code First approach
  - Fluent API pour configuration avancée (dans Configs/)
  - Auto-discovery des configurations via `ApplyConfigurationsFromAssembly`
- **Optimisations** :
  - `AsNoTracking()` pour les lectures sans modification
  - `Include()` pour eager loading des relations
- **Gestion intelligente** :
  - Vérification de l'existence de catégories avant insertion
  - Requêtes complexes pour recherche par plage de dates

**Points techniques notables** :
```csharp
// Recherche d'événements par plage de dates (logique complexe)
public IEnumerable<AgendaEvent> GetByDate(DateTime start, DateTime? end)
{
    // Gestion de 3 cas : 
    // - Événement commence avant et finit après
    // - Événement termine après mais commence avant
    // - Événement complètement inclus
    // (Voir commentaires ligne 126-138 du fichier)
}
```

#### 4️⃣ **Presentation.WebAPI** (Interface utilisateur)
**Responsabilité** : Exposer l'API REST et gérer les communications HTTP.

**Contenu** :
```
Presentation.WebAPI/
├── Controllers/          ← Endpoints REST (AgendaEventController, AuthController)
├── Dto/
│   ├── Request/         ← DTOs pour les requêtes entrantes
│   ├── Response/        ← DTOs pour les réponses
│   └── Mappers/         ← Conversions Domain ↔ DTO
├── ExceptionHandlers/   ← Gestion centralisée des erreurs
├── Token/               ← Gestion de l'authentification
└── Program.cs           ← Configuration DI + Middleware
```

**Principes appliqués** :
- **Pattern DTO** : Séparation entre modèles API et domaine
- **Pattern Mapper** : Méthodes d'extension pour conversion Domain ↔ DTO
- **Exception Handler** : Gestion centralisée des erreurs métier
- **Validation** : Data Annotations sur les DTOs
- **REST** : Respect des conventions HTTP (200, 201, 204, 404, 422)

**Architecture typique d'un endpoint** :
```
Request (JSON) 
  ↓
[Controller] Validation automatique des Data Annotations
  ↓
[Mapper] DTO → Domain Model
  ↓
[Service] Logique métier + validation
  ↓
[Repository] Persistance
  ↓
[Mapper] Domain Model → DTO
  ↓
Response (JSON)
```

### Respect des principes de Clean Architecture

#### ✅ Dépendances unidirectionnelles
```
Presentation → ApplicationCore → Domain
     ↓              ↓
Infrastructure ←────┘
```

- **Domain** : 0 dépendance (projet autonome)
- **ApplicationCore** : Dépend uniquement de Domain
- **Infrastructure** : Dépend de ApplicationCore + Domain
- **Presentation** : Dépend de toutes les couches (composition root)

#### ✅ Inversion de dépendance (Dependency Inversion Principle)
Les services ne connaissent **que les interfaces**, jamais les implémentations :
```csharp
// Service dépend d'une interface
private IAgendaEventRepository _repo;  // ✅ Interface

// Implémentation fournie via Injection de Dépendance
public AgendaEventService(IAgendaEventRepository repo) 
{
    _repo = repo;  // Implémentation injectée au runtime
}
```

Configuration dans `Program.cs` :
```csharp
// Services (ApplicationCore)
builder.Services.AddScoped<IAgendaEventService, AgendaEventService>();

// Repositories (Infrastructure)
builder.Services.AddScoped<IAgendaEventRepository, AgendaEventRepository>();
```

#### ✅ Séparation des responsabilités

| Couche | Responsabilité | Exemples |
|--------|---------------|----------|
| **Domain** | Règles métier pures | Validation des dates, encapsulation |
| **ApplicationCore** | Orchestration + règles applicatives | "Créer 1 jour avant", pagination |
| **Infrastructure** | Accès technique aux données | SQL, EF Core, migrations |
| **Presentation** | Interface utilisateur | HTTP, JSON, validation DTOs |

### Patterns utilisés

#### 1. **Repository Pattern**
Abstraction de la persistance des données.

```csharp
// Interface dans ApplicationCore
public interface IAgendaEventRepository
{
    AgendaEvent? GetById(long id);
    AgendaEvent Insert(AgendaEvent data);
}

// Implémentation dans Infrastructure
public class AgendaEventRepository : IAgendaEventRepository
{
    private readonly AppDbContext _dbContext;
    
    public AgendaEvent? GetById(long id) => 
        _dbContext.AgendaEvents
            .Include(ae => ae.Category)
            .SingleOrDefault(ae => ae.Id == id);
}
```

**Avantages** :
- Testabilité (possibilité de mocker les repositories)
- Changement de technologie de persistance sans impact sur la logique métier

#### 2. **DTO Pattern + Mapper**
Séparation entre modèles internes et externes.

```csharp
// DTO Request avec validation
public class AgendaEventRequestDto
{
    [Required, MinLength(3), MaxLength(50)]
    public required string Name { get; set; }
}

// Mapper (méthodes d'extension)
public static class AgendaEventMapper
{
    public static AgendaEvent ToDomain(this AgendaEventRequestDto dto) 
        => new AgendaEvent(dto.Name, ...);
    
    public static AgendaEventResponseDto ToResponseDto(this AgendaEvent data) 
        => new AgendaEventResponseDto { Id = data.Id, ... };
}

// Usage dans le contrôleur
AgendaEvent domain = requestDto.ToDomain();      // DTO → Domain
AgendaEventResponseDto response = result.ToResponseDto();  // Domain → DTO
```

**Avantages** :
- Contrôle total sur ce qui est exposé via l'API
- Validation indépendante de la logique métier
- Évolution de l'API sans casser le domaine

#### 3. **Exception Handler Pattern**
Centralisation de la gestion des erreurs.

```csharp
public class AgendaEventExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext ctx, Exception ex, CancellationToken ct)
    {
        if (ex is not AgendaEventException) return false;
        
        int status = ex is AgendaEventNotFoundException ? 404
            : ex is AgendaEventCreateException ? 422
            : 400;
        
        ctx.Response.StatusCode = status;
        await ctx.Response.WriteAsJsonAsync(new ProblemDetails {...});
        return true;
    }
}
```

Enregistrement dans `Program.cs` :
```csharp
builder.Services.AddExceptionHandler<AgendaEventExceptionHandler>();
```

**Avantages** :
- Gestion uniforme des erreurs
- Séparation logique métier / gestion HTTP
- Respect du principe DRY (Don't Repeat Yourself)

#### 4. **Domain-Driven Design (DDD)**
- Entités avec validation intégrée
- Méthodes métier sur les entités (`ChangeDate()`)
- Exceptions métier personnalisées (`AgendaEventCreateException`)
- Encapsulation forte (setters privés)

#### 5. **Dependency Injection (DI)**
Injection automatique des dépendances via le conteneur IoC d'ASP.NET Core.

```csharp
// Configuration
builder.Services.AddScoped<IAgendaEventService, AgendaEventService>();

// Injection
public AgendaEventController(IAgendaEventService service) 
{
    _service = service;  // Injecté automatiquement
}
```

**Portées utilisées** :
- `Singleton` : Outils (TokenTool) - une instance pour l'application
- `Scoped` : Services et Repositories - une instance par requête HTTP
- `Transient` : Non utilisé dans ce projet

#### 6. **Fluent Interface** (Optionnel)
Méthodes chaînables dans le Domain.

```csharp
public AgendaEvent ChangeDate(DateTime start, DateTime? end)
{
    // Validation et modification
    return this;  // Permet le chaînage
}

// Usage potentiel
event.ChangeDate(newStart, newEnd)
     .SomeOtherMethod()
     .AnotherMethod();
```

---

## ✅ Points forts

### 1. **Qualité du découpage architectural**
- ✅ **Séparation nette des responsabilités** : Chaque couche a un rôle clair et distinct
- ✅ **Dépendances unidirectionnelles** : Le flux respecte strictement Domain ← ApplicationCore ← Infrastructure/Presentation
- ✅ **Modules cohésifs** : Les fichiers sont organisés logiquement par domaine (AgendaEvent, Member, Faq)
- ✅ **Nomenclature cohérente** : Conventions de nommage claires et homogènes

### 2. **Testabilité**
- ✅ **Interfaces partout** : Services et Repositories sont mockables
- ✅ **Injection de dépendances** : Facilite le remplacement des dépendances en tests
- ✅ **Logique métier isolée** : Le Domain et ApplicationCore sont testables sans base de données
- ✅ **DTOs séparés** : Permet de tester les contrôleurs indépendamment

### 3. **Maintenabilité**
- ✅ **Code bien commenté** : Commentaires pédagogiques utiles pour les développeurs juniors
- ✅ **Exceptions métier explicites** : `AgendaEventNotFoundException`, `AgendaEventCreateException`
- ✅ **Gestion centralisée des erreurs** : Exception Handlers évitent la duplication
- ✅ **Configuration externalisée** : `appsettings.json` pour les connexions DB

### 4. **Extensibilité**
- ✅ **Ajout facile de nouvelles entités** : Structure reproductible (Model → Service → Repository → Controller)
- ✅ **Changement de technologie facilité** :
  - Remplacer SQL Server par PostgreSQL : uniquement l'Infrastructure change
  - Ajouter une API GraphQL : nouveau projet Presentation sans toucher au Core
- ✅ **Auto-discovery des configurations EF** : `ApplyConfigurationsFromAssembly()` évite l'ajout manuel

### 5. **Lisibilité du code**
- ✅ **Mappers sous forme de méthodes d'extension** : `entity.ToResponseDto()` très lisible
- ✅ **Noms explicites** : `AgendaEventCreateException`, `IAgendaEventRepository`
- ✅ **Code métier simple** : Pas de sur-ingénierie, logique directe
- ✅ **Exemples commentés** : Cas d'usage de `GetByDate()` illustrés en commentaires

### 6. **Validation multi-niveaux**
- ✅ **Niveau 1 - DTOs** : Data Annotations (`[Required]`, `[MinLength]`)
- ✅ **Niveau 2 - Domain** : Validation dans les constructeurs (règles métier pures)
- ✅ **Niveau 3 - Services** : Validation applicative (ex: "créer 1 jour avant")

---

## ⚠️ Points faibles / Axes d'amélioration

### 1. **Violations de Clean Architecture**

#### 🔴 Infrastructure dépend d'ApplicationCore
```xml
<!-- Infrastructure.Database.csproj -->
<ProjectReference Include="..\Demo_WebAPI_EventAgenda.ApplicationCore\..." />
```

**Problème** : L'Infrastructure **ne devrait dépendre QUE du Domain**, pas d'ApplicationCore.

**Impact** :
- Couplage inutile : l'Infrastructure connaît les interfaces de services
- Empêche la réutilisation de l'Infrastructure dans d'autres contextes

**Solution** :
- Déplacer les interfaces de Repository (`IAgendaEventRepository`) du `ApplicationCore/Interfaces/Repositories/` vers `Domain/Interfaces/`
- Infrastructure n'aurait alors qu'une référence à Domain

**Après correction** :
```
Infrastructure.Database/ (dépend uniquement de Domain)
├── Repositories/
│   └── AgendaEventRepository.cs  // Implémente Domain.Interfaces.IAgendaEventRepository
```

### 2. **Couplage excessif**

#### 🟡 Présentation référence directement Infrastructure
```xml
<!-- Presentation.WebAPI.csproj -->
<ProjectReference Include="..\Infrastructure.Database\..." />
```

**Problème** : Nécessaire pour la configuration DI, mais crée un couplage technique.

**Impact limité** : Acceptable dans un monolithe, mais bloque l'évolution vers des microservices.

**Solution avancée** :
- Pattern **Composition Root** : Créer un projet `Bootstrapper` dédié à la configuration DI
- La Presentation ne connaîtrait que les interfaces

#### 🟡 Présentation importe Domain directement
Dans `AgendaEventController.cs` :
```csharp
using Demo_WebAPI_EventAgenda.Domain.Models;  // Import direct
```

**Problème** : Le contrôleur manipule les entités du Domain alors qu'il devrait uniquement manipuler des DTOs.

**Impact** : 
- Risque de fuite d'informations sensibles
- Couplage fort entre API et modèle interne

**Solution** :
```csharp
// ❌ Actuellement
AgendaEvent result = _service.GetById(id);  // Entité Domain dans le contrôleur
AgendaEventResponseDto dto = result.ToResponseDto();

// ✅ Meilleure approche
AgendaEventResponseDto dto = _service.GetById(id);  // Service retourne directement un DTO
```

### 3. **Code smells**

#### 🟡 Logique de mapping dans Presentation
**Problème** : Les mappers (`ToDomain()`, `ToResponseDto()`) sont dans la couche Presentation, mais sont utilisés par les Services.

**Meilleure pratique** :
- Créer un projet `Application.Contracts` contenant :
  - Les DTOs
  - Les mappers
  - Les interfaces de services
- ApplicationCore et Presentation dépendraient tous deux de `Application.Contracts`

#### 🟡 Duplication de logique de pagination
```csharp
// Dans le Service
int offset = (page - 1) * nbElement;
int limit = nbElement;
```

**Solution** : Créer un objet `PaginationParameters` réutilisable.

```csharp
public record PaginationParameters(int Page, int PageSize)
{
    public int Offset => (Page - 1) * PageSize;
    public int Limit => PageSize;
}
```

#### 🟡 Gestion des transactions absente
**Problème** : Pas de pattern **Unit of Work** pour gérer les transactions.

**Impact** : 
```csharp
// Scénario problématique
_repo1.Insert(data1);
_dbContext.SaveChanges();  // ✅ Réussit
_repo2.Insert(data2);
_dbContext.SaveChanges();  // ❌ Échoue → Données incohérentes
```

**Solution** : Implémenter un Unit of Work :
```csharp
public interface IUnitOfWork
{
    IAgendaEventRepository AgendaEvents { get; }
    IMemberRepository Members { get; }
    Task<int> SaveChangesAsync();
}
```

### 4. **Manque de tests**

**Absence totale de tests** :
- ❌ Pas de projet de tests unitaires
- ❌ Pas de tests d'intégration
- ❌ Pas de tests de contrats (contract testing)

**Recommandations** :
```
Solution/
├── Tests/
│   ├── Domain.Tests/              ← Tests des entités et validations
│   ├── ApplicationCore.Tests/     ← Tests des services (mocking des repos)
│   ├── Infrastructure.Tests/      ← Tests d'intégration avec InMemory DB
│   └── Presentation.Tests/        ← Tests des contrôleurs (WebApplicationFactory)
```

**Exemples de tests à ajouter** :
```csharp
// Domain.Tests
[Fact]
public void AgendaEvent_Constructor_ThrowsException_WhenNameIsEmpty()
{
    Assert.Throws<ArgumentException>(() => 
        new AgendaEvent("", null, null, DateTime.Now, null, category));
}

// ApplicationCore.Tests
[Fact]
public void Create_ThrowsException_WhenEventDateIsToday()
{
    var mockRepo = new Mock<IAgendaEventRepository>();
    var service = new AgendaEventService(mockRepo.Object);
    
    var eventToday = new AgendaEvent("Test", null, null, DateTime.Today, null, category);
    
    Assert.Throws<AgendaEventCreateException>(() => service.Create(eventToday));
}
```

### 5. **Complexité inutile**

#### 🟡 Double DTO pour les réponses
```csharp
public class AgendaEventResponseDto { /* 7 propriétés */ }
public class AgendaEventListItemResponseDto { /* 4 propriétés */ }
```

**Problème** : Duplication de code pour un gain marginal.

**Alternative** :
- Utiliser JSON.NET ou System.Text.Json avec `[JsonIgnore]` conditionnel
- Ou accepter de renvoyer plus de données dans les listes (si acceptable)

#### 🟡 Constructeur privé dans Domain
```csharp
private AgendaEvent() { }  // Pour EF Core uniquement
```

**Impact limité**, mais montre une concession au framework ORM.

**Alternative** : Utiliser un ORM sans cette contrainte (ex: Dapper) ou accepter ce compromis.

### 6. **Améliorations techniques**

#### 📊 Logging absent
Aucun logging structuré (Serilog, NLog).

**Ajout recommandé** :
```csharp
builder.Services.AddLogging(logging =>
{
    logging.AddSerilog();
});
```

#### 🔒 Authentification/Autorisation limitée
Un `TokenTool` existe mais n'est pas utilisé dans le projet.

**À implémenter** :
- JWT Tokens pour authentification
- Policies pour autorisation (Admin, User, etc.)

#### 🔄 Absence de CQRS
Toutes les opérations (Commandes et Requêtes) passent par les mêmes services.

**Pour des projets plus complexes**, considérer :
- **MediatR** pour CQRS
- Séparation `Commands/` et `Queries/`

#### 📝 Validation FluentValidation
Data Annotations limitées pour des règles complexes.

**Alternative** :
```csharp
public class AgendaEventRequestDtoValidator : AbstractValidator<AgendaEventRequestDto>
{
    public AgendaEventRequestDtoValidator()
    {
        RuleFor(x => x.Name).NotEmpty().Length(3, 50);
        RuleFor(x => x.StartDate).GreaterThan(DateTime.Today);
    }
}
```

#### 🔐 Secrets en clair
Connexion DB dans `appsettings.json` visible en clair.

**Solution** :
- Utiliser **User Secrets** en développement
- Utiliser **Azure Key Vault** ou variables d'environnement en production

---

## 🛠️ Ordre d'implémentation recommandé

### Phase 1️⃣ : **Fondations (Domain)**
**Par quoi commencer** : Toujours par le cœur métier.

```
1. Créer le projet Domain (Class Library)
   └─ Définir les entités métier (AgendaEvent, EventCategory)
       ├─ Propriétés avec setters privés
       ├─ Constructeurs avec validation
       └─ Méthodes métier (ChangeDate)
   
2. Définir les exceptions métier
   └─ AgendaEventException
       ├─ AgendaEventNotFoundException
       └─ AgendaEventCreateException

3. Créer les interfaces de repositories DANS LE DOMAIN (correction)
   └─ Domain/Interfaces/IAgendaEventRepository.cs
```

**Ordre de création des entités** :
1. `EventCategory` (entité simple sans dépendance)
2. `AgendaEvent` (dépend d'EventCategory)
3. `Member` (indépendant)
4. `Faq` (indépendant)

**Validation** : Lancer des tests unitaires sur le Domain dès cette étape.

---

### Phase 2️⃣ : **Logique métier (ApplicationCore)**
**Construction progressive des couches** : Créer les services une fois le Domain stable.

```
1. Créer le projet ApplicationCore (Class Library)
   └─ Référence : Domain uniquement

2. Définir les interfaces de services
   └─ Interfaces/Services/IAgendaEventService.cs

3. Implémenter les services
   └─ Services/AgendaEventService.cs
       ├─ Injection du IAgendaEventRepository (interface)
       ├─ Implémentation des use cases
       └─ Gestion des règles métier (ex: créer 1 jour avant)

4. Tester les services (mocking des repositories)
```

**Principes** :
- Les services manipulent **uniquement** des objets du Domain
- Ils ne connaissent **aucune** implémentation concrète (EF Core, SQL, etc.)

---

### Phase 3️⃣ : **Infrastructure (Accès données)**
**Mise en place des interfaces et contrats** : Implémenter ce que le Core a défini.

```
1. Créer le projet Infrastructure.Database (Class Library)
   └─ Références : Domain (uniquement)

2. Installer les packages NuGet
   └─ Microsoft.EntityFrameworkCore
   └─ Microsoft.EntityFrameworkCore.SqlServer

3. Créer le DbContext
   └─ AppDbContext.cs
       └─ DbSet<AgendaEvent>, DbSet<EventCategory>, etc.

4. Configurer les entités (Fluent API)
   └─ Configs/AgendaEventConfig.cs
       └─ Implémenter IEntityTypeConfiguration<AgendaEvent>

5. Implémenter les repositories
   └─ Repositories/AgendaEventRepository.cs
       └─ Implémente IAgendaEventRepository

6. Créer les migrations
   └─ dotnet ef migrations add InitialCreate
```

**Ordre de configuration EF Core** :
1. Entités sans relations (EventCategory, Member)
2. Entités avec relations (AgendaEvent → EventCategory)

---

### Phase 4️⃣ : **Presentation (API Web)**
**Ajout de l'infrastructure** : Exposer l'API REST.

```
1. Créer le projet Presentation.WebAPI (Web API)
   └─ Références : ApplicationCore, Infrastructure, Domain

2. Configurer l'injection de dépendances (Program.cs)
   └─ Services, Repositories, DbContext

3. Créer les DTOs
   └─ Dto/Request/AgendaEventRequestDto.cs
   └─ Dto/Response/AgendaEventResponseDto.cs

4. Créer les Mappers
   └─ Dto/Mappers/AgendaEventMapper.cs

5. Créer les contrôleurs
   └─ Controllers/AgendaEventController.cs

6. Implémenter les Exception Handlers
   └─ ExceptionHandlers/AgendaEventExceptionHandler.cs

7. Tester via Swagger
```

**Configuration DI dans Program.cs** (ordre recommandé) :
```csharp
// 1. DbContext
builder.Services.AddDbContext<AppDbContext>(options => 
    options.UseSqlServer(connectionString));

// 2. Repositories
builder.Services.AddScoped<IAgendaEventRepository, AgendaEventRepository>();

// 3. Services
builder.Services.AddScoped<IAgendaEventService, AgendaEventService>();

// 4. Controllers
builder.Services.AddControllers();

// 5. Exception Handlers
builder.Services.AddExceptionHandler<AgendaEventExceptionHandler>();
```

---

## 🏗️ Création du projet from scratch

### Étape 1 : Création de la structure de solution

```bash
# Créer le dossier racine
mkdir EventAgenda && cd EventAgenda

# Créer la solution
dotnet new sln -n EventAgenda

# Créer les projets
dotnet new classlib -n EventAgenda.Domain
dotnet new classlib -n EventAgenda.ApplicationCore
dotnet new classlib -n EventAgenda.Infrastructure.Database
dotnet new webapi -n EventAgenda.Presentation.WebAPI

# Ajouter les projets à la solution
dotnet sln add EventAgenda.Domain/EventAgenda.Domain.csproj
dotnet sln add EventAgenda.ApplicationCore/EventAgenda.ApplicationCore.csproj
dotnet sln add EventAgenda.Infrastructure.Database/EventAgenda.Infrastructure.Database.csproj
dotnet sln add EventAgenda.Presentation.WebAPI/EventAgenda.Presentation.WebAPI.csproj
```

### Étape 2 : Configuration des dépendances entre projets

```bash
# ApplicationCore dépend de Domain
cd EventAgenda.ApplicationCore
dotnet add reference ../EventAgenda.Domain/EventAgenda.Domain.csproj

# Infrastructure dépend de Domain (correction Clean Archi)
cd ../EventAgenda.Infrastructure.Database
dotnet add reference ../EventAgenda.Domain/EventAgenda.Domain.csproj

# Presentation dépend de tout (composition root)
cd ../EventAgenda.Presentation.WebAPI
dotnet add reference ../EventAgenda.Domain/EventAgenda.Domain.csproj
dotnet add reference ../EventAgenda.ApplicationCore/EventAgenda.ApplicationCore.csproj
dotnet add reference ../EventAgenda.Infrastructure.Database/EventAgenda.Infrastructure.Database.csproj
```

### Étape 3 : Installation des packages NuGet

```bash
# Infrastructure - Entity Framework Core
cd ../EventAgenda.Infrastructure.Database
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design

# Presentation - API Documentation
cd ../EventAgenda.Presentation.WebAPI
dotnet add package Microsoft.AspNetCore.OpenApi
dotnet add package Scalar.AspNetCore
```

### Étape 4 : Organisation des dossiers

```bash
# Domain
cd ../EventAgenda.Domain
mkdir Models
mkdir BusinessExceptions
mkdir Interfaces  # NOUVEAU (correction Clean Archi)

# ApplicationCore
cd ../EventAgenda.ApplicationCore
mkdir Interfaces
mkdir Interfaces/Services
mkdir Services

# Infrastructure
cd ../EventAgenda.Infrastructure.Database
mkdir Repositories
mkdir Configs
mkdir Migrations

# Presentation
cd ../EventAgenda.Presentation.WebAPI
mkdir Controllers
mkdir Dto
mkdir Dto/Request
mkdir Dto/Response
mkdir Dto/Mappers
mkdir ExceptionHandlers
```

### Étape 5 : Création des fichiers de base

#### 5.1 Domain - Modèle AgendaEvent
```csharp
// EventAgenda.Domain/Models/AgendaEvent.cs
namespace EventAgenda.Domain.Models
{
    public class AgendaEvent
    {
        public long Id { get; private set; }
        public string Name { get; private set; } = default!;
        public string? Description { get; private set; }
        public DateTime StartDate { get; private set; }
        public DateTime? EndDate { get; private set; }
        public EventCategory Category { get; private set; } = default!;

        private AgendaEvent() { } // EF Core

        public AgendaEvent(string name, string? description, 
            DateTime startDate, DateTime? endDate, EventCategory category)
        {
            if (string.IsNullOrWhiteSpace(name))
                throw new ArgumentException("Name required", nameof(name));
                
            if (endDate.HasValue && endDate < startDate)
                throw new ArgumentException("Invalid dates");

            Name = name;
            Description = description;
            StartDate = startDate;
            EndDate = endDate;
            Category = category;
        }

        public AgendaEvent ChangeDate(DateTime start, DateTime? end)
        {
            if (end.HasValue && end < start)
                throw new ArgumentException("Invalid dates");

            StartDate = start;
            EndDate = end;
            return this;
        }
    }
}
```

#### 5.2 Domain - Interface Repository (CORRECTION)
```csharp
// EventAgenda.Domain/Interfaces/IAgendaEventRepository.cs
namespace EventAgenda.Domain.Interfaces
{
    public interface IAgendaEventRepository
    {
        AgendaEvent? GetById(long id);
        IEnumerable<AgendaEvent> GetMany(int offset, int limit);
        AgendaEvent Insert(AgendaEvent data);
        AgendaEvent Update(AgendaEvent data);
        bool Delete(long id);
    }
}
```

#### 5.3 ApplicationCore - Interface Service
```csharp
// EventAgenda.ApplicationCore/Interfaces/Services/IAgendaEventService.cs
using EventAgenda.Domain.Models;

namespace EventAgenda.ApplicationCore.Interfaces.Services
{
    public interface IAgendaEventService
    {
        AgendaEvent GetById(long id);
        IEnumerable<AgendaEvent> GetMany(int page, int pageSize);
        AgendaEvent Create(AgendaEvent data);
        void Delete(long id);
    }
}
```

#### 5.4 ApplicationCore - Implémentation Service
```csharp
// EventAgenda.ApplicationCore/Services/AgendaEventService.cs
using EventAgenda.Domain.Interfaces;
using EventAgenda.Domain.Models;

namespace EventAgenda.ApplicationCore.Services
{
    public class AgendaEventService : IAgendaEventService
    {
        private readonly IAgendaEventRepository _repository;

        public AgendaEventService(IAgendaEventRepository repository)
        {
            _repository = repository;
        }

        public AgendaEvent Create(AgendaEvent data)
        {
            // Règle métier : créer au moins 1 jour avant
            if (data.StartDate < DateTime.Today.AddDays(1))
                throw new InvalidOperationException("Event must be created 1 day in advance");

            return _repository.Insert(data);
        }

        public AgendaEvent GetById(long id)
        {
            return _repository.GetById(id) 
                ?? throw new KeyNotFoundException("Event not found");
        }

        // ... autres méthodes
    }
}
```

#### 5.5 Infrastructure - DbContext
```csharp
// EventAgenda.Infrastructure.Database/AppDbContext.cs
using EventAgenda.Domain.Models;
using Microsoft.EntityFrameworkCore;

namespace EventAgenda.Infrastructure.Database
{
    public class AppDbContext : DbContext
    {
        public DbSet<AgendaEvent> AgendaEvents { get; set; }
        public DbSet<EventCategory> EventCategories { get; set; }

        public AppDbContext(DbContextOptions options) : base(options) { }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            modelBuilder.ApplyConfigurationsFromAssembly(
                Assembly.GetExecutingAssembly());
        }
    }
}
```

#### 5.6 Infrastructure - Repository
```csharp
// EventAgenda.Infrastructure.Database/Repositories/AgendaEventRepository.cs
using EventAgenda.Domain.Interfaces;
using EventAgenda.Domain.Models;

namespace EventAgenda.Infrastructure.Database.Repositories
{
    public class AgendaEventRepository : IAgendaEventRepository
    {
        private readonly AppDbContext _dbContext;

        public AgendaEventRepository(AppDbContext dbContext)
        {
            _dbContext = dbContext;
        }

        public AgendaEvent? GetById(long id)
        {
            return _dbContext.AgendaEvents
                .Include(e => e.Category)
                .SingleOrDefault(e => e.Id == id);
        }

        public AgendaEvent Insert(AgendaEvent data)
        {
            var entry = _dbContext.AgendaEvents.Add(data);
            _dbContext.SaveChanges();
            return entry.Entity;
        }

        // ... autres méthodes
    }
}
```

#### 5.7 Presentation - DTO
```csharp
// EventAgenda.Presentation.WebAPI/Dto/Request/AgendaEventRequestDto.cs
using System.ComponentModel.DataAnnotations;

namespace EventAgenda.Presentation.WebAPI.Dto.Request
{
    public class AgendaEventRequestDto
    {
        [Required, MinLength(3), MaxLength(50)]
        public required string Name { get; set; }
        
        [MaxLength(500)]
        public string? Description { get; set; }
        
        [Required]
        public required DateTime StartDate { get; set; }
        
        public DateTime? EndDate { get; set; }
        
        [Required]
        public required string Category { get; set; }
    }
}
```

#### 5.8 Presentation - Mapper
```csharp
// EventAgenda.Presentation.WebAPI/Dto/Mappers/AgendaEventMapper.cs
using EventAgenda.Domain.Models;

namespace EventAgenda.Presentation.WebAPI.Dto.Mappers
{
    public static class AgendaEventMapper
    {
        public static AgendaEvent ToDomain(this AgendaEventRequestDto dto)
        {
            return new AgendaEvent(
                dto.Name,
                dto.Description,
                dto.StartDate,
                dto.EndDate,
                new EventCategory(dto.Category)
            );
        }

        public static AgendaEventResponseDto ToResponseDto(this AgendaEvent entity)
        {
            return new AgendaEventResponseDto
            {
                Id = entity.Id,
                Name = entity.Name,
                Description = entity.Description,
                StartDate = entity.StartDate,
                EndDate = entity.EndDate,
                Category = entity.Category.Name
            };
        }
    }
}
```

#### 5.9 Presentation - Controller
```csharp
// EventAgenda.Presentation.WebAPI/Controllers/AgendaEventController.cs
using EventAgenda.ApplicationCore.Interfaces.Services;
using Microsoft.AspNetCore.Mvc;

namespace EventAgenda.Presentation.WebAPI.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class AgendaEventController : ControllerBase
    {
        private readonly IAgendaEventService _service;

        public AgendaEventController(IAgendaEventService service)
        {
            _service = service;
        }

        [HttpGet("{id}")]
        public IActionResult GetById(long id)
        {
            var result = _service.GetById(id);
            var dto = result.ToResponseDto();
            return Ok(dto);
        }

        [HttpPost]
        public IActionResult Create(AgendaEventRequestDto request)
        {
            var entity = request.ToDomain();
            var result = _service.Create(entity);
            var dto = result.ToResponseDto();
            
            return CreatedAtAction(
                nameof(GetById), 
                new { id = result.Id }, 
                dto);
        }
    }
}
```

#### 5.10 Presentation - Program.cs
```csharp
// EventAgenda.Presentation.WebAPI/Program.cs
using EventAgenda.ApplicationCore.Interfaces.Services;
using EventAgenda.ApplicationCore.Services;
using EventAgenda.Domain.Interfaces;
using EventAgenda.Infrastructure.Database;
using EventAgenda.Infrastructure.Database.Repositories;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Database
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

// Repositories
builder.Services.AddScoped<IAgendaEventRepository, AgendaEventRepository>();

// Services
builder.Services.AddScoped<IAgendaEventService, AgendaEventService>();

// Controllers
builder.Services.AddControllers();
builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
    app.MapScalarApiReference();
}

app.UseHttpsRedirection();
app.MapControllers();
app.Run();
```

### Étape 6 : Configuration appsettings.json
```json
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=EventAgenda;Integrated Security=True;TrustServerCertificate=True"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  }
}
```

### Étape 7 : Migrations et création de la base
```bash
# Dans le dossier Infrastructure
cd EventAgenda.Infrastructure.Database

# Créer la migration initiale
dotnet ef migrations add InitialCreate --startup-project ../EventAgenda.Presentation.WebAPI

# Appliquer la migration
dotnet ef database update --startup-project ../EventAgenda.Presentation.WebAPI
```

### Étape 8 : Lancement et test
```bash
# Lancer l'API
cd ../EventAgenda.Presentation.WebAPI
dotnet run

# Ouvrir le navigateur
# https://localhost:5001/scalar/v1
```

### Étape 9 : Mise en place des tests (recommandé)
```bash
# Créer les projets de tests
dotnet new xunit -n EventAgenda.Domain.Tests
dotnet new xunit -n EventAgenda.ApplicationCore.Tests
dotnet new xunit -n EventAgenda.Presentation.Tests

# Ajouter à la solution
dotnet sln add EventAgenda.Domain.Tests/EventAgenda.Domain.Tests.csproj
dotnet sln add EventAgenda.ApplicationCore.Tests/EventAgenda.ApplicationCore.Tests.csproj
dotnet sln add EventAgenda.Presentation.Tests/EventAgenda.Presentation.Tests.csproj

# Ajouter les références
cd EventAgenda.Domain.Tests
dotnet add reference ../EventAgenda.Domain/EventAgenda.Domain.csproj

cd ../EventAgenda.ApplicationCore.Tests
dotnet add reference ../EventAgenda.ApplicationCore/EventAgenda.ApplicationCore.csproj
dotnet add package Moq  # Pour le mocking

cd ../EventAgenda.Presentation.Tests
dotnet add reference ../EventAgenda.Presentation.WebAPI/EventAgenda.Presentation.WebAPI.csproj
dotnet add package Microsoft.AspNetCore.Mvc.Testing
```

---

## 📊 Schéma explicatif de l'architecture

### Vue d'ensemble - Flux de données

```
┌─────────────────────────────────────────────────────────────────┐
│                       CLIENT (Browser/Mobile)                    │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ HTTP Request (JSON)
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                            │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ AgendaEventController                                     │  │
│  │  ├─ [HttpPost] Create(AgendaEventRequestDto)            │  │
│  │  ├─ [HttpGet] GetById(long id)                          │  │
│  │  └─ [HttpDelete] Delete(long id)                        │  │
│  └───────────────────────────────────────────────────────────┘  │
│                             │                                    │
│       1. Validation DTO (Data Annotations)                      │
│       2. DTO → Domain (Mapper)                                  │
│                             ↓                                    │
└─────────────────────────────┼────────────────────────────────────┘
                              │
                              │ AgendaEvent (Domain Object)
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                   APPLICATION CORE LAYER                         │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ IAgendaEventService (Interface)                          │  │
│  │  ├─ AgendaEvent Create(AgendaEvent)                      │  │
│  │  ├─ AgendaEvent GetById(long id)                         │  │
│  │  └─ void Delete(long id)                                 │  │
│  └───────────────────────────────────────────────────────────┘  │
│                             │                                    │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ AgendaEventService (Implementation)                      │  │
│  │  • Règle métier : Créer 1 jour avant                     │  │
│  │  • Validation applicative                                │  │
│  │  • Orchestration des repositories                        │  │
│  └───────────────────────────────────────────────────────────┘  │
│                             │                                    │
│                             │ Appel via IAgendaEventRepository   │
│                             ↓                                    │
└─────────────────────────────┼────────────────────────────────────┘
                              │
                              │ Interface (Dependency Inversion)
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                   INFRASTRUCTURE LAYER                           │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ AgendaEventRepository (Implementation)                   │  │
│  │  • GetById(long id) → Entity Framework Query            │  │
│  │  • Insert(AgendaEvent) → DbContext.Add()                │  │
│  │  • Delete(long id) → DbContext.Remove()                 │  │
│  └───────────────────────────────────────────────────────────┘  │
│                             │                                    │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ AppDbContext : DbContext                                 │  │
│  │  • DbSet<AgendaEvent> AgendaEvents                       │  │
│  │  • DbSet<EventCategory> EventCategories                  │  │
│  │  • OnModelCreating() → Fluent API Configs               │  │
│  └───────────────────────────────────────────────────────────┘  │
│                             │                                    │
│                             │ SQL Queries                        │
│                             ↓                                    │
└─────────────────────────────┼────────────────────────────────────┘
                              │
                              ↓
                      ┌───────────────┐
                      │  SQL SERVER   │
                      │   Database    │
                      └───────────────┘
```

### Graphe de dépendances entre projets

```
                    ┌─────────────────────┐
                    │      DOMAIN         │
                    │  (Models, Excp)     │
                    │  NO DEPENDENCIES    │
                    └──────────┬──────────┘
                               │
                ┌──────────────┼──────────────┐
                │              │              │
                ↓              ↓              ↓
     ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
     │ APPLICATION  │  │ INFRASTRUCTURE│  │ PRESENTATION │
     │     CORE     │  │   .Database   │  │    .WebAPI   │
     │              │  │               │  │              │
     │  Services    │  │  Repositories │  │  Controllers │
     │  Interfaces  │  │  DbContext    │  │      DTOs    │
     └──────────────┘  └──────────────┘  └──────┬───────┘
                                                 │
                       ┌─────────────────────────┘
                       │  (Composition Root)
                       │  Injection Dépendances
                       └─ Program.cs
```

### Flux d'une requête HTTP complète

```
┌──────────────────────────────────────────────────────────────────┐
│ REQUEST: POST /api/agendaevent                                   │
│ Body: { "name": "Concert", "startDate": "2026-03-15", ... }     │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│ 1. CONTROLLER - AgendaEventController.Create()                   │
│    • Validation automatique du DTO (Data Annotations)           │
│    • Conversion DTO → Domain via Mapper                         │
│      AgendaEvent event = requestDto.ToDomain();                 │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│ 2. SERVICE - AgendaEventService.Create(event)                    │
│    • Vérification règle métier:                                 │
│      if (event.StartDate < DateTime.Today.AddDays(1))           │
│          throw new AgendaEventCreateException();                │
│    • Appel au repository:                                       │
│      return _repository.Insert(event);                          │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│ 3. REPOSITORY - AgendaEventRepository.Insert(event)              │
│    • Vérification catégorie existante:                          │
│      category = _dbContext.EventCategories                      │
│                   .SingleOrDefault(c => c.Name == event.Cat);   │
│    • Création entité:                                           │
│      var entry = _dbContext.AgendaEvents.Add(event);            │
│    • Persistance:                                               │
│      _dbContext.SaveChanges();                                  │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│ 4. DATABASE - SQL Server                                         │
│    • INSERT INTO AgendaEvents (Name, StartDate, ...)            │
│      VALUES ('Concert', '2026-03-15', ...);                     │
│    • RETURN Id = 42                                             │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                    ↓ ← ← ← ← Retour
┌──────────────────────────────────────────────────────────────────┐
│ 5. REPOSITORY → SERVICE                                          │
│    • AgendaEvent result = { Id: 42, Name: "Concert", ... }      │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                    ↓ ← ← ← ←
┌──────────────────────────────────────────────────────────────────┐
│ 6. SERVICE → CONTROLLER                                          │
│    • AgendaEvent result = ...                                   │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                    ↓ ← ← ← ←
┌──────────────────────────────────────────────────────────────────┐
│ 7. CONTROLLER - Conversion Domain → DTO                          │
│    • AgendaEventResponseDto dto = result.ToResponseDto();       │
│    • return CreatedAtAction(..., dto);                          │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│ RESPONSE: 201 Created                                            │
│ Location: /api/agendaevent/42                                   │
│ Body: { "id": 42, "name": "Concert", ... }                      │
└──────────────────────────────────────────────────────────────────┘
```

### Gestion des erreurs (Exception Handler)

```
┌──────────────────────────────────────────────────────────────────┐
│ REQUEST: POST /api/agendaevent                                   │
│ Body: { "startDate": "2026-02-11" }  ← Aujourd'hui !             │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ↓
                      [Controller]
                             │
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│ SERVICE - AgendaEventService.Create()                            │
│    if (event.StartDate < DateTime.Today.AddDays(1))             │
│        throw new AgendaEventCreateException(event);  ← BOOM!    │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             │ Exception remonte
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│ MIDDLEWARE - Exception Handler Pipeline                         │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ AgendaEventExceptionHandler.TryHandleAsync()              │ │
│  │  • Détecte: exception is AgendaEventCreateException       │ │
│  │  • Détermine le status code: 422 Unprocessable Entity     │ │
│  │  • Crée ProblemDetails:                                   │ │
│  │    {                                                      │ │
│  │      "title": "AgendaEvent error !",                      │ │
│  │      "detail": "Erreur lors de la création...",           │ │
│  │      "status": 422                                        │ │
│  │    }                                                      │ │
│  │  • Envoie la réponse                                      │ │
│  └────────────────────────────────────────────────────────────┘ │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│ RESPONSE: 422 Unprocessable Entity                               │
│ Body: {                                                          │
│   "title": "AgendaEvent error !",                                │
│   "detail": "Erreur lors de la création de l'événement",         │
│   "status": 422                                                  │
│ }                                                                │
└──────────────────────────────────────────────────────────────────┘
```

### Injection de dépendances (DI Container)

```
┌─────────────────────────────────────────────────────────────────┐
│                      STARTUP (Program.cs)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  // Configuration du Container IoC                              │
│  builder.Services.AddScoped<IAgendaEventService,                │
│                              AgendaEventService>();             │
│  builder.Services.AddScoped<IAgendaEventRepository,             │
│                              AgendaEventRepository>();          │
│  builder.Services.AddDbContext<AppDbContext>(...)               │
│                                                                  │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            │ Lors d'une requête HTTP
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│                    DI CONTAINER RÉSOUT                           │
│                                                                  │
│  Request → AgendaEventController                                │
│              ↓ (besoin de IAgendaEventService)                  │
│              Container crée AgendaEventService                  │
│                ↓ (besoin de IAgendaEventRepository)             │
│                Container crée AgendaEventRepository             │
│                  ↓ (besoin de AppDbContext)                     │
│                  Container crée AppDbContext                    │
│                                                                  │
│  Hiérarchie construite:                                         │
│  AgendaEventController                                          │
│    └─ AgendaEventService                                        │
│          └─ AgendaEventRepository                               │
│                └─ AppDbContext                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Récapitulatif des patterns

```
┌─────────────────────────────────────────────────────────────────┐
│                         PATTERNS UTILISÉS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Clean Architecture                                          │
│     └─ Séparation en couches avec dépendances unidirectionnelles│
│                                                                  │
│  2. Repository Pattern                                          │
│     └─ Abstraction de l'accès aux données                       │
│                                                                  │
│  3. DTO Pattern + Mapper                                        │
│     └─ Séparation modèles API / Domain                          │
│                                                                  │
│  4. Exception Handler Pattern                                   │
│     └─ Gestion centralisée des erreurs                          │
│                                                                  │
│  5. Domain-Driven Design (DDD)                                  │
│     └─ Entités riches avec validation                           │
│                                                                  │
│  6. Dependency Injection (DI)                                   │
│     └─ Inversion de contrôle                                    │
│                                                                  │
│  7. Facade Pattern                                              │
│     └─ Services comme façade de la logique métier               │
│                                                                  │
│  8. Fluent Interface                                            │
│     └─ Méthodes chaînables (optionnel)                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📈 Récapitulatif - Matrice d'évaluation

| Critère | Note | Commentaire |
|---------|------|-------------|
| **Architecture** | ⭐⭐⭐⭐ | Bonne séparation, une violation (Infra → AppCore) |
| **Testabilité** | ⭐⭐⭐ | Interfaces présentes, mais aucun test |
| **Maintenabilité** | ⭐⭐⭐⭐⭐ | Code clair, bien commenté, cohérent |
| **Extensibilité** | ⭐⭐⭐⭐ | Facile d'ajouter des entités, mais pas de CQRS |
| **Performance** | ⭐⭐⭐⭐ | `AsNoTracking()`, eager loading, bonnes pratiques EF |
| **Sécurité** | ⭐⭐ | Authentification partielle, secrets en clair |
| **Documentation** | ⭐⭐⭐⭐ | Commentaires pédagogiques, OpenAPI/Swagger |

---

## 🎯 Recommandations prioritaires

### Court terme (Quick wins)
1. ✅ Déplacer les interfaces de Repository dans Domain
2. ✅ Ajouter des tests unitaires de base (Domain + Services)
3. ✅ Implémenter FluentValidation pour remplacer Data Annotations
4. ✅ Ajouter un logging structuré (Serilog)

### Moyen terme (Améliorations)
5. ✅ Implémenter le pattern Unit of Work
6. ✅ Créer un projet Application.Contracts pour les DTOs
7. ✅ Ajouter des health checks
8. ✅ Implémenter l'authentification JWT complète

### Long terme (Évolution)
9. ✅ Migrer vers CQRS avec MediatR
10. ✅ Ajouter des projets de tests d'intégration
11. ✅ Implémenter Event Sourcing pour l'audit
12. ✅ Migrer vers une architecture microservices si croissance

---

## 📚 Ressources complémentaires

- **Clean Architecture** : "Clean Architecture" par Robert C. Martin
- **DDD** : "Domain-Driven Design" par Eric Evans
- **ASP.NET Core** : Documentation Microsoft officielle
- **EF Core** : Entity Framework Core Best Practices
- **Testing** : "The Art of Unit Testing" par Roy Osherove

---

## ✅ Conclusion

Ce projet est une **excellente démonstration pédagogique** de la Clean Architecture en .NET. Il respecte la majorité des principes fondamentaux et constitue une base solide pour un projet d'entreprise.

**Points remarquables** :
- Architecture bien pensée et documentée
- Code maintenable et extensible
- Bonne séparation des responsabilités
- Patterns modernes appliqués correctement

**Axes d'amélioration principaux** :
- Ajouter des tests (manque critique)
- Corriger la dépendance Infrastructure → ApplicationCore
- Renforcer la sécurité et le logging

**Verdict final** : 📊 **8/10** - Projet très bien conçu, prêt pour la production après ajout des tests et corrections mineures.
