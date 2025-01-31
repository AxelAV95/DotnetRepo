### Hexagonal Architectura - .NET - Cheatsheet
---

### **1. Estructura de Carpetas**
La estructura de carpetas sigue las capas de Clean Architecture:

```
src/
├── Core/
│   ├── Entities/            # Entidades del dominio
│   ├── Interfaces/          # Interfaces para repositorios y servicios
│   └── UseCases/            # Casos de uso (lógica de negocio)
├── Infrastructure/
│   ├── Data/                # Contexto de EF Core y migraciones
│   ├── Repositories/        # Implementaciones de repositorios
│   └── Services/            # Implementaciones de servicios externos
├── Application/
│   ├── DTOs/                # DTOs (Data Transfer Objects)
│   ├── Mappings/            # Configuración de AutoMapper
│   ├── Commands/            # Comandos para operaciones CRUD
│   ├── Queries/             # Consultas para operaciones CRUD
│   ├── Validators/          # Validaciones con FluentValidation
│   ├── Handlers/            # Manejadores de comandos y consultas
│   └── Services/            # Servicios de aplicación
└── WebAPI/
    ├── Controllers/         # Controladores RESTful
    ├── Extensions/          # Extensiones para configurar servicios
    └── Program.cs           # Punto de entrada de la aplicación
```

---

### **2. Paso a Paso Completo**

#### **2.1. Crear el Proyecto**
1. Crea una solución en .NET:
   ```bash
   dotnet new sln -n CleanArchitectureDemo
   ```
2. Agrega proyectos individuales para cada capa:
   ```bash
   dotnet new classlib -n Core
   dotnet new classlib -n Infrastructure
   dotnet new classlib -n Application
   dotnet new webapi -n WebAPI
   ```

3. Añade los proyectos a la solución:
   ```bash
   dotnet sln add Core
   dotnet sln add Infrastructure
   dotnet sln add Application
   dotnet sln add WebAPI
   ```

4. Configura las referencias entre proyectos:
   ```bash
   dotnet add WebAPI reference Application
   dotnet add Application reference Core
   dotnet add Infrastructure reference Core
   dotnet add Application reference Infrastructure
   ```

---

#### **2.2. Configurar Core**
1. **Entidad Producto** (`Core/Entities/Product.cs`):
   ```csharp
   namespace Core.Entities;

   public class Product
   {
       public int Id { get; set; }
       public string Name { get; set; }
       public decimal Price { get; set; }
       public string Description { get; set; }
   }
   ```

2. **Interfaz del Repositorio** (`Core/Interfaces/IProductRepository.cs`):
   ```csharp
   using Core.Entities;

   namespace Core.Interfaces;

   public interface IProductRepository
   {
       Task<IEnumerable<Product>> GetAllAsync();
       Task<Product> GetByIdAsync(int id);
       Task AddAsync(Product product);
       Task UpdateAsync(Product product);
       Task DeleteAsync(int id);
   }
   ```

3. **Interfaz del Servicio Externo** (`Core/Interfaces/IExternalApiService.cs`):
   ```csharp
   namespace Core.Interfaces;

   public interface IExternalApiService
   {
       Task<string> GetDataFromExternalApiAsync(string endpoint);
       Task<bool> SendDataToExternalApiAsync(string endpoint, object data);
   }
   ```

4. **Use Case para Obtener Productos** (`Core/UseCases/GetAllProductsUseCase.cs`):
   ```csharp
   using Core.Entities;
   using Core.Interfaces;

   namespace Core.UseCases;

   public class GetAllProductsUseCase
   {
       private readonly IProductRepository _repository;

       public GetAllProductsUseCase(IProductRepository repository)
       {
           _repository = repository;
       }

       public async Task<IEnumerable<Product>> ExecuteAsync()
       {
           return await _repository.GetAllAsync();
       }
   }
   ```

5. **Use Case para Consumir API Externa** (`Core/UseCases/ConsumeExternalApiUseCase.cs`):
   ```csharp
   using Core.Interfaces;

   namespace Core.UseCases;

   public class ConsumeExternalApiUseCase
   {
       private readonly IExternalApiService _externalApiService;

       public ConsumeExternalApiUseCase(IExternalApiService externalApiService)
       {
           _externalApiService = externalApiService;
       }

       public async Task<string> ExecuteAsync(string endpoint)
       {
           return await _externalApiService.GetDataFromExternalApiAsync(endpoint);
       }
   }
   ```

---

#### **2.3. Configurar Infrastructure**
1. **Contexto de EF Core** (`Infrastructure/Data/AppDbContext.cs`):
   ```csharp
   using Core.Entities;
   using Microsoft.EntityFrameworkCore;

   namespace Infrastructure.Data;

   public class AppDbContext : DbContext
   {
       public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

       public DbSet<Product> Products { get; set; }
   }
   ```

2. **Implementación del Repositorio** (`Infrastructure/Repositories/ProductRepository.cs`):
   ```csharp
   using Core.Entities;
   using Core.Interfaces;
   using Infrastructure.Data;
   using Microsoft.EntityFrameworkCore;

   namespace Infrastructure.Repositories;

   public class ProductRepository : IProductRepository
   {
       private readonly AppDbContext _context;

       public ProductRepository(AppDbContext context)
       {
           _context = context;
       }

       public async Task<IEnumerable<Product>> GetAllAsync()
       {
           return await _context.Products.ToListAsync();
       }

       public async Task<Product> GetByIdAsync(int id)
       {
           return await _context.Products.FindAsync(id);
       }

       public async Task AddAsync(Product product)
       {
           await _context.Products.AddAsync(product);
           await _context.SaveChangesAsync();
       }

       public async Task UpdateAsync(Product product)
       {
           _context.Products.Update(product);
           await _context.SaveChangesAsync();
       }

       public async Task DeleteAsync(int id)
       {
           var product = await _context.Products.FindAsync(id);
           if (product != null)
           {
               _context.Products.Remove(product);
               await _context.SaveChangesAsync();
           }
       }
   }
   ```

3. **Implementación del Servicio Externo** (`Infrastructure/Services/ExternalApiService.cs`):
   ```csharp
   using System.Net.Http;
   using System.Text.Json;
   using System.Threading.Tasks;
   using Core.Interfaces;

   namespace Infrastructure.Services;

   public class ExternalApiService : IExternalApiService
   {
       private readonly HttpClient _httpClient;

       public ExternalApiService(HttpClient httpClient)
       {
           _httpClient = httpClient;
       }

       public async Task<string> GetDataFromExternalApiAsync(string endpoint)
       {
           var response = await _httpClient.GetAsync(endpoint);
           response.EnsureSuccessStatusCode();
           return await response.Content.ReadAsStringAsync();
       }

       public async Task<bool> SendDataToExternalApiAsync(string endpoint, object data)
       {
           var jsonContent = new StringContent(
               JsonSerializer.Serialize(data),
               System.Text.Encoding.UTF8,
               "application/json");

           var response = await _httpClient.PostAsync(endpoint, jsonContent);
           return response.IsSuccessStatusCode;
       }
   }
   ```

---

#### **2.4. Configurar Application**
1. **DTOs** (`Application/DTOs/ProductDto.cs`):
   ```csharp
   namespace Application.DTOs;

   public class ProductDto
   {
       public int Id { get; set; }
       public string Name { get; set; }
       public decimal Price { get; set; }
       public string Description { get; set; }
   }
   ```

2. **Configuración de AutoMapper** (`Application/Mappings/ProductProfile.cs`):
   ```csharp
   using AutoMapper;
   using Core.Entities;
   using Application.DTOs;

   namespace Application.Mappings;

   public class ProductProfile : Profile
   {
       public ProductProfile()
       {
           CreateMap<Product, ProductDto>();
           CreateMap<ProductDto, Product>();
       }
   }
   ```

3. **Query para Obtener Todos los Productos** (`Application/Queries/GetAllProductsQuery.cs`):
   ```csharp
   using MediatR;
   using Application.DTOs;

   namespace Application.Queries;

   public record GetAllProductsQuery : IRequest<IEnumerable<ProductDto>>;
   ```

4. **Handler para el Query** (`Application/Handlers/GetAllProductsHandler.cs`):
   ```csharp
   using MediatR;
   using AutoMapper;
   using Core.Entities;
   using Application.DTOs;
   using Application.Queries;
   using Core.UseCases;

   namespace Application.Handlers;

   public class GetAllProductsHandler : IRequestHandler<GetAllProductsQuery, IEnumerable<ProductDto>>
   {
       private readonly GetAllProductsUseCase _useCase;
       private readonly IMapper _mapper;

       public GetAllProductsHandler(GetAllProductsUseCase useCase, IMapper mapper)
       {
           _useCase = useCase;
           _mapper = mapper;
       }

       public async Task<IEnumerable<ProductDto>> Handle(GetAllProductsQuery request, CancellationToken cancellationToken)
       {
           var products = await _useCase.ExecuteAsync();
           return _mapper.Map<IEnumerable<ProductDto>>(products);
       }
   }
   ```

5. **Command para Crear un Producto** (`Application/Commands/CreateProductCommand.cs`):
   ```csharp
   using MediatR;
   using Application.DTOs;

   namespace Application.Commands;

   public record CreateProductCommand(ProductDto ProductDto) : IRequest<int>;
   ```

6. **Handler para el Command** (`Application/Handlers/CreateProductHandler.cs`):
   ```csharp
   using MediatR;
   using AutoMapper;
   using Core.Entities;
   using Application.DTOs;
   using Application.Commands;
   using Core.UseCases;

   namespace Application.Handlers;

   public class CreateProductHandler : IRequestHandler<CreateProductCommand, int>
   {
       private readonly CreateProductUseCase _useCase;
       private readonly IMapper _mapper;

       public CreateProductHandler(CreateProductUseCase useCase, IMapper mapper)
       {
           _useCase = useCase;
           _mapper = mapper;
       }

       public async Task<int> Handle(CreateProductCommand request, CancellationToken cancellationToken)
       {
           var product = _mapper.Map<Product>(request.ProductDto);
           return await _useCase.ExecuteAsync(product);
       }
   }
   ```

7. **Validaciones con FluentValidation** (`Application/Validators/ProductValidator.cs`):
   ```csharp
   using FluentValidation;
   using Application.DTOs;

   namespace Application.Validators;

   public class ProductValidator : AbstractValidator<ProductDto>
   {
       public ProductValidator()
       {
           RuleFor(p => p.Name).NotEmpty().MaximumLength(100);
           RuleFor(p => p.Price).GreaterThan(0);
       }
   }
   ```

---

#### **2.5. Configurar WebAPI**
1. **Controlador de Productos** (`WebAPI/Controllers/ProductController.cs`):
   ```csharp
   using Microsoft.AspNetCore.Mvc;
   using MediatR;
   using Application.DTOs;
   using Application.Commands;
   using Application.Queries;

   namespace WebAPI.Controllers;

   [ApiController]
   [Route("api/[controller]")]
   public class ProductController : ControllerBase
   {
       private readonly IMediator _mediator;

       public ProductController(IMediator mediator)
       {
           _mediator = mediator;
       }

       [HttpGet]
       public async Task<IActionResult> GetAllProducts()
       {
           var query = new GetAllProductsQuery();
           var products = await _mediator.Send(query);
           return Ok(products);
       }

       [HttpPost]
       public async Task<IActionResult> CreateProduct([FromBody] ProductDto productDto)
       {
           var command = new CreateProductCommand(productDto);
           var productId = await _mediator.Send(command);
           return CreatedAtAction(nameof(GetAllProducts), new { id = productId }, productDto);
       }
   }
   ```

2. **Controlador para Consumir API Externa** (`WebAPI/Controllers/ExternalApiController.cs`):
   ```csharp
   using Microsoft.AspNetCore.Mvc;
   using MediatR;
   using Application.Queries;

   namespace WebAPI.Controllers;

   [ApiController]
   [Route("api/[controller]")]
   public class ExternalApiController : ControllerBase
   {
       private readonly IMediator _mediator;

       public ExternalApiController(IMediator mediator)
       {
           _mediator = mediator;
       }

       [HttpGet("{endpoint}")]
       public async Task<IActionResult> GetExternalData(string endpoint)
       {
           var query = new GetExternalDataQuery(endpoint);
           var result = await _mediator.Send(query);
           return Ok(result);
       }
   }
   ```

3. **Configuración de Servicios** (`WebAPI/Extensions/ServiceCollectionExtensions.cs`):
   ```csharp
   using AutoMapper;
   using FluentValidation;
   using MediatR;
   using Infrastructure.Data;
   using Infrastructure.Repositories;
   using Infrastructure.Services;
   using Core.Interfaces;
   using Application.Mappings;

   namespace WebAPI.Extensions;

   public static class ServiceCollectionExtensions
   {
       public static IServiceCollection AddApplicationServices(this IServiceCollection services)
       {
           services.AddAutoMapper(typeof(ProductProfile));
           services.AddMediatR(typeof(CreateProductCommand).Assembly);
           services.AddValidatorsFromAssembly(typeof(ProductValidator).Assembly);

           services.AddScoped<IProductRepository, ProductRepository>();
           services.AddHttpClient<IExternalApiService, ExternalApiService>();

           services.AddDbContext<AppDbContext>(options =>
               options.UseSqlServer("YourConnectionStringHere"));

           return services;
       }
   }
   ```

4. **Program.cs**:
   ```csharp
   var builder = WebApplication.CreateBuilder(args);

   builder.Services.AddControllers();
   builder.Services.AddEndpointsApiExplorer();
   builder.Services.AddSwaggerGen();

   builder.Services.AddApplicationServices();

   var app = builder.Build();

   if (app.Environment.IsDevelopment())
   {
       app.UseSwagger();
       app.UseSwaggerUI();
   }

   app.UseHttpsRedirection();
   app.UseAuthorization();
   app.MapControllers();

   app.Run();
   ```

---

### **3. Conclusión**
Este cheatsheet integra todas las piezas necesarias para implementar un CRUD de productos con servicios externos en una arquitectura limpia. Las herramientas como **AutoMapper**, **FluentValidation**, **MediatR** y **Entity Framework Core** ayudan a mantener el código modular, limpio y fácil de mantener. 😊

