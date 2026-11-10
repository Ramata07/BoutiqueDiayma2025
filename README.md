# Boutique Diayma

TD C# et Technologies .NET : Visual Studio et débogage (ESP, 2026/2027)
Réalisé par : Ramata Niane

- Dépôt : https://github.com/Ramata07/BoutiqueDiayma2025
- Exécutable Windows (Google Drive) : https://drive.google.com/file/d/1OecWQmEtUmw_eqsivT9UfLvkBS7TSEna/view?usp=drive_link
- Outil utilisé : Visual Studio Code avec l'extension C# (débogueur coreclr, configuration dans `.vscode/launch.json`)
- Lancement en local : `dotnet run --project P2FixAnAppDotNetCode`, puis http://localhost:62929

## 1. Projets de la solution

La solution `Diayma.sln` contient **un seul projet** :

| Projet | Fichier | Type |
|---|---|---|
| Diayma | `P2FixAnAppDotNetCode/Diayma.csproj` | Application web ASP.NET Core MVC (`Microsoft.NET.Sdk.Web`) |

Le dossier s'appelle `P2FixAnAppDotNetCode` et le namespace racine porte ce même nom, mais le projet s'appelle `Diayma`.

## 2. Version du SDK .NET

- Framework cible, dans `Diayma.csproj` : `<TargetFramework>netcoreapp2.0</TargetFramework>`, soit **.NET Core 2.0**.
- SDK installé : **.NET Core SDK 2.1.202**, dernier SDK de la branche .NET Core 2.0 (il inclut le runtime 2.0.9).
- Téléchargement : https://dotnet.microsoft.com/download/dotnet/2.0

Le numéro du SDK (2.1.x) est différent de celui du runtime (2.0.x) : le SDK sert à compiler, le runtime à exécuter.

## 3. Bugs trouvés

### Bug 1 : le total du panier ignore les quantités

- **Où :** `Models/Cart.cs`, méthodes `GetTotalValue()` (ligne 66) et `GetAverageValue()` (ligne 77).
- **Reproduire :** ajouter deux fois le même produit au panier, par exemple le XIAOMI Redmi 13C à 88 000.
- **Constaté :** quantité 2, sous-total 176 000, mais total 88 000.
- **Cause :** le calcul additionne `Product.Price` sans le multiplier par `Quantity`.
- **Correction :** `Sum(x => x.Product.Price * x.Quantity)`, et moyenne égale au total divisé par la quantité totale.

### Bug 2 : le stock ne diminue jamais après une commande

- **Où :** `Models/Repositories/ProductRepository.cs`, constructeur (lignes 13 à 17).
- **Reproduire :** commander un produit, puis revenir à la liste des produits.
- **Constaté :** le stock affiché reste identique.
- **Cause :** `ProductRepository` est enregistré en `AddTransient`, donc recréé à chaque requête. Son constructeur réinitialise à chaque fois la liste statique `_products`, ce qui efface la mise à jour du stock.
- **Correction :** n'initialiser la liste que si `_products == null`.

### Autre anomalie : les prix sont affichés en euros

Les prix sont en FCFA, mais la page affiche « 88 000,00 € ». Le format `ToString("c")` utilise la monnaie de la culture `fr-FR`.

## 4. Points d'arrêt

| Repère | Fichier et ligne | Instruction | Atteint à l'affichage de l'accueil ? |
|---|---|---|---|
| e | `Startup.cs:20` | `Configuration = configuration;` | Exécuté au démarrage, avant toute requête (voir la remarque ci-dessous) |
| b | `Controllers/ProductController.cs:15` | `_productService = productService;` | Oui, arrêt observé à l'ouverture de http://localhost:62929 |
| a | `Components/CartSummaryViewComponent.cs:12` | `_cart = cart as Cart;` | Oui, après b, au rendu du layout |
| d | `Controllers/CartController.cs:15` | `_cart = pCart;` | Non, seulement sur `/Cart` |
| c | `Controllers/OrderController.cs:17` | `_cart = pCart;` | Non, seulement sur `/Order` |

**Remarque sur `Startup` :** sous VS Code, le constructeur s'exécute dès le lancement, avant que les points d'arrêt soient tous actifs, et le premier essai ne s'y est pas arrêté. Pour l'observer, ajouter `"stopAtEntry": true` dans `.vscode/launch.json` : le programme s'arrête au début de `Program.Main`, et une reprise (F5) l'amène ensuite à `Startup.cs:20`.

Les arrêts c et d ne sont pas atteints car ces contrôleurs ne sont créés que lorsqu'une requête les cible. La route par défaut, `{controller=Product}/{action=Index}/{id?}`, envoie `/` vers `ProductController.Index`.

## 5. Parcours avant l'affichage des produits

Modes de pas à pas utilisés :

| Mode | Touche | Quand je l'utilise |
|---|---|---|
| Pas à pas détaillé | F11 | Pour entrer dans une méthode de l'application (contrôleur, service, repository) |
| Pas à pas principal | F10 | Pour passer un appel du framework ou une ligne sans intérêt (`CreateDefaultBuilder`, `services.Add...`, `_products.Add`) |
| Pas à pas sortant | Maj+F11 | Pour remonter à l'appelant une fois vu l'essentiel (`GenerateProductData`, constructeur ou accesseurs de `Product`) |

### Phase 1 : démarrage de l'application

| # | Namespace | Classe | Méthode | Mode |
|---|---|---|---|---|
| 1 | `P2FixAnAppDotNetCode` | `Program` | `Main(string[])` | F11 |
| 2 | `P2FixAnAppDotNetCode` | `Program` | `BuildWebHost(string[])` | F11 |
| 3 | `Microsoft.AspNetCore` | `WebHost` | `CreateDefaultBuilder(args)` | F10 |
| 4 | `Microsoft.AspNetCore.Hosting` | `WebHostBuilderExtensions` | `UseStartup<Startup>()` | F10 |
| 5 | `Microsoft.AspNetCore.Hosting` | `IWebHostBuilder` | `Build()` | déclenche 6 et 7 |
| 6 | `P2FixAnAppDotNetCode` | `Startup` | constructeur `Startup(IConfiguration)`, **point d'arrêt e** | arrêt |
| 7 | `P2FixAnAppDotNetCode` | `Startup` | `ConfigureServices(IServiceCollection)` | F10 ligne par ligne |
| 8 | `Microsoft.AspNetCore.Hosting` | `WebHostExtensions` | `Run()` | déclenche 9 |
| 9 | `P2FixAnAppDotNetCode` | `Startup` | `Configure(IApplicationBuilder, IHostingEnvironment)` | F10 ligne par ligne |
| 10 | `P2FixAnAppDotNetCode` | `Startup` | expression lambda de `services.Configure<RequestLocalizationOptions>`, exécutée à la lecture de `options.Value` | F10 |

Le constructeur et `ConfigureServices` s'exécutent pendant `Build()`, alors que `Configure` s'exécute au lancement du serveur, dans `Run()`.

### Phase 2 : requête GET `/` et création du contrôleur

L'injection de dépendances crée d'abord les dépendances, puis le contrôleur.

| # | Namespace | Classe | Méthode | Mode |
|---|---|---|---|---|
| 11 | `P2FixAnAppDotNetCode.Models.Repositories` | `ProductRepository` | constructeur | F11 |
| 12 | `P2FixAnAppDotNetCode.Models.Repositories` | `ProductRepository` | `GenerateProductData()` | F11 |
| 13 | `P2FixAnAppDotNetCode.Models` | `Product` | constructeur, 9 appels | F11 puis Maj+F11 |
| 14 | `P2FixAnAppDotNetCode.Models.Repositories` | `OrderRepository` | constructeur | F10 |
| 15 | `P2FixAnAppDotNetCode.Models.Services` | `ProductService` | constructeur | F10 |
| 16 | `P2FixAnAppDotNetCode.Models.Services` | `LanguageService` | constructeur implicite (singleton) | non visible |
| 17 | `P2FixAnAppDotNetCode.Controllers` | `ProductController` | constructeur, **point d'arrêt b** | arrêt |

Pour voir les étapes 11 à 16, ajouter des points d'arrêt dans `ProductRepository.cs:15` et `ProductService.cs:17`, car elles se produisent avant l'arrêt b.

### Phase 3 : action du contrôleur

| # | Namespace | Classe | Méthode | Mode |
|---|---|---|---|---|
| 18 | `P2FixAnAppDotNetCode.Controllers` | `ProductController` | `Index()` | F11 |
| 19 | `P2FixAnAppDotNetCode.Models.Services` | `ProductService` | `GetAllProducts()` | F11 |
| 20 | `P2FixAnAppDotNetCode.Models.Repositories` | `ProductRepository` | `GetAllProducts()` | F11 |
| 21 | `System.Linq` | `Enumerable` | `Where`, `OrderBy`, `ToList`, `ToArray` | F10 |
| 22 | `Microsoft.AspNetCore.Mvc` | `Controller` | `View(object)` | F10 |

### Phase 4 : rendu de la page

Les vues Razor sont compilées dans le namespace `AspNetCore`, et chacune est exécutée par sa méthode `ExecuteAsync()`.

| # | Vue | Classe générée | Détail |
|---|---|---|---|
| 23 | `Views/_ViewStart.cshtml` | `_Views__ViewStart_cshtml` | Définit le layout |
| 24 | `Views/Product/Index.cshtml` | `_Views_Product_Index_cshtml` | Boucle sur les produits, accesseurs de `Product` (`Name`, `Description`, `Price`, `Stock`, `Id`) |
| 25 | `Views/Shared/_Layout.cshtml` | `_Views_Shared__Layout_cshtml` | Appelle `Component.InvokeAsync("CartSummary")` |
| 26 | - | `P2FixAnAppDotNetCode.Components.CartSummaryViewComponent` | Constructeur, **point d'arrêt a**, reçoit le `Cart` singleton créé à ce moment |
| 27 | - | `P2FixAnAppDotNetCode.Components.CartSummaryViewComponent` | `Invoke()` |
| 28 | `Views/Shared/Components/CartSummary/Default.cshtml` | `_Views_Shared_Components_CartSummary_Default_cshtml` | Lit `Cart.Lines` |
| 29 | - | `P2FixAnAppDotNetCode.Components.LanguageSelectorViewComponent` | `Invoke(ILanguageService)`, appelé depuis le layout |
| 30 | `Views/Shared/Components/LanguageSelector/Default.cshtml` | `_Views_Shared_Components_LanguageSelector_Default_cshtml` | Affiche le choix de la langue |

Le corps de la page (`Product/Index`) est rendu avant le layout, ce qui explique que le point d'arrêt b soit atteint avant le point d'arrêt a.

### Récapitulatif des namespaces de l'application visités

- `P2FixAnAppDotNetCode` : `Program`, `Startup`
- `P2FixAnAppDotNetCode.Controllers` : `ProductController`
- `P2FixAnAppDotNetCode.Models.Services` : `ProductService`, `LanguageService`
- `P2FixAnAppDotNetCode.Models.Repositories` : `ProductRepository`, `OrderRepository`
- `P2FixAnAppDotNetCode.Models` : `Product`, `Cart`
- `P2FixAnAppDotNetCode.Components` : `CartSummaryViewComponent`, `LanguageSelectorViewComponent`

## 6. Déploiement en exécutable Windows

Publication autonome pour Windows 64 bits, qui ne demande pas d'installer .NET sur la machine cible (commandes PowerShell) :

```powershell
dotnet publish P2FixAnAppDotNetCode\Diayma.csproj -c Release -r win-x64 --self-contained -o $HOME\Diayma-publish
Compress-Archive $HOME\Diayma-publish\* $HOME\Diayma.zip
```

Le dossier publié contient `Diayma.exe` et toutes ses dépendances. L'archive `Diayma.zip` est partagée sur Google Drive (lien en haut de ce README).

Utilisation :
1. Télécharger et décompresser `Diayma.zip`.
2. Lancer `Diayma.exe` depuis le dossier décompressé.
3. Ouvrir http://localhost:5000 dans le navigateur.

L'exécutable publié n'utilise pas `launchSettings.json` : il écoute donc sur le port par défaut 5000, et non sur le port 62929 utilisé par `dotnet run`.