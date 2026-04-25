# Angular Code Examples

## Standalone Component Setup

```typescript
@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [DatePipe],
  templateUrl: './user-list.component.html',
  styleUrls: ['./user-list.component.scss'],
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserListComponent {
  readonly users = input.required<User[]>();
  readonly filter = input<string>('');
  readonly userSelected = output<User>();

  protected readonly filteredUsers = computed(() =>
    this.users().filter(u => u.name.toLowerCase().includes(this.filter().toLowerCase())),
  );
}
```

```html
<!-- user-list.component.html -->
@if (filteredUsers().length) {
  <ul>
    @for (user of filteredUsers(); track user.id) {
      <li (click)="userSelected.emit(user)">
        {{ user.name }} — {{ user.createdAt | date:'mediumDate' }}
      </li>
    } @empty {
      <li>No users match the filter.</li>
    }
  </ul>
} @else {
  <p>No users loaded.</p>
}

@for (user of filteredUsers(); track user.id) {
  @switch (user.role) {
    @case ('admin') { <app-admin-badge /> }
    @case ('editor') { <app-editor-badge /> }
    @default { <span>Member</span> }
  }
}
```

## Dependency Injection Patterns

### `inject()` in a Functional Guard

```typescript
export const authGuard: CanActivateFn = () => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }
  return router.createUrlTree(['/login']);
};
```

### Constructor Injection in a Service

```typescript
@Injectable({ providedIn: 'root' })
export class OrderService {
  private readonly http: HttpClient;
  private readonly config: AppConfig;

  constructor(http: HttpClient, @Inject(APP_CONFIG) config: AppConfig) {
    this.http = http;
    this.config = config;
  }

  getOrders$(): Observable<Order[]> {
    return this.http.get<Order[]>(`${this.config.apiUrl}/orders`);
  }
}
```

### InjectionToken for Type-Safe Config

```typescript
export interface AppConfig {
  readonly apiBaseUrl: string;
  readonly featureFlags: Record<string, boolean>;
}

export const APP_CONFIG = new InjectionToken<AppConfig>('app.config');

// Loaded at bootstrap via APP_INITIALIZER from /api/config endpoint.
// See "Externalized Configuration" section for full setup.
```

## Signals & Reactive State

### Facade Service

```typescript
@Injectable({ providedIn: 'root' })
export class ProductFacade {
  private readonly http: HttpClient;

  private readonly productsState = signal<Product[]>([]);
  private readonly loadingState = signal(false);
  private readonly errorState = signal<string | null>(null);

  readonly products = this.productsState.asReadonly();
  readonly loading = this.loadingState.asReadonly();
  readonly error = this.errorState.asReadonly();

  readonly totalCount = computed(() => this.productsState().length);
  readonly hasError = computed(() => this.errorState() !== null);

  constructor(http: HttpClient) {
    this.http = http;

    effect(() => {
      if (this.hasError()) {
        console.error('Product load failed:', this.errorState());
      }
    });
  }

  load(): void {
    this.loadingState.set(true);
    this.errorState.set(null);
    this.http.get<Product[]>('/api/products').subscribe({
      next: products => {
        this.productsState.set(products);
        this.loadingState.set(false);
      },
      error: err => {
        this.errorState.set(err.message);
        this.loadingState.set(false);
      },
    });
  }
}
```

### Bridging Observable to Signal

```typescript
@Component({
  selector: 'app-dashboard',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    @if (currentUser()) {
      <h1>Welcome, {{ currentUser()!.name }}</h1>
    }
  `,
})
export class DashboardComponent {
  private readonly authService = inject(AuthService);

  protected readonly currentUser = toSignal(this.authService.currentUser$, {
    initialValue: null,
  });
}
```

## Spring Boot Integration

### `angular.json` (relevant excerpt)

```json
{
  "projects": {
    "my-app": {
      "sourceRoot": "src/main/webapp",
      "architect": {
        "build": {
          "options": {
            "outputPath": "build/resources/main/static"
          }
        }
      }
    }
  }
}
```

### `build.gradle` with `gradle-node-plugin`

```groovy
plugins {
    id 'com.github.node-gradle.node' version '7.1.0'
}

node {
    version = '22.12.0'
    download = true
}

tasks.register('buildAngular', NpxTask) {
    dependsOn npmInstall
    command = 'ng'
    args = ['build', '--configuration=production']
    inputs.dir('src/main/webapp')
    inputs.file('angular.json')
    inputs.file('package.json')
    inputs.file('tsconfig.json')
    outputs.dir('build/resources/main/static')
}

processResources {
    dependsOn buildAngular
}
```

### Forwarding Controller

```java
/**
 * Forwards non-API, non-static requests to the Angular index page.
 */
@Controller
public class AngularForwardController {

    @GetMapping("{path:^(?!api|v3|swagger-ui|actuator|public|error)[^\\.]*}/**")
    public String handleForward() {
        return "forward:/index.html";
    }
}
```

### CORS Config for Local Profile

```java
/**
 * Permits cross-origin requests from the Angular dev server during local development.
 */
@Configuration
@Profile("local")
public class AngularLocalConfig {

    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(final CorsRegistry registry) {
                registry.addMapping("/**").allowedMethods("*").allowedOrigins("http://localhost:4200");
            }
        };
    }
}
```

## Externalized Configuration

### Backend Config Endpoint (Java)

```java
/**
 * Serves externalized configuration to the Angular frontend.
 * Values are sourced from Spring Cloud Config Server.
 */
@RestController
@RequestMapping("/api/config")
@RefreshScope
@Tag(name = "Configuration", description = "Frontend configuration")
public class ConfigController {

    private final FrontendConfigProperties config;

    public ConfigController(FrontendConfigProperties config) {
        this.config = config;
    }

    @Operation(summary = "Get frontend configuration")
    @ApiResponse(responseCode = "200", description = "Configuration retrieved")
    @GetMapping
    public FrontendConfigResponse getConfig() {
        return new FrontendConfigResponse(
            config.apiBaseUrl(),
            config.featureFlags()
        );
    }
}
```

```java
@ConfigurationProperties(prefix = "app.frontend")
public record FrontendConfigProperties(
    String apiBaseUrl,
    Map<String, Boolean> featureFlags
) {}
```

```java
@Schema(description = "Frontend runtime configuration")
public record FrontendConfigResponse(
    @Schema(description = "Base URL for API calls", example = "/api") String apiBaseUrl,
    @Schema(description = "Feature flag map") Map<String, Boolean> featureFlags
) {}
```

### Angular Config Service and Bootstrap

```typescript
export interface AppConfig {
  readonly apiBaseUrl: string;
  readonly featureFlags: Record<string, boolean>;
}

export const APP_CONFIG = new InjectionToken<AppConfig>('app.config');
```

```typescript
@Injectable({ providedIn: 'root' })
export class ConfigService {
  private readonly http = inject(HttpClient);
  private config: AppConfig | null = null;

  load(): Observable<AppConfig> {
    return this.http.get<AppConfig>('/api/config').pipe(
      tap(config => (this.config = config)),
    );
  }

  getConfig(): AppConfig {
    if (!this.config) {
      throw new Error('Config not loaded. Ensure APP_INITIALIZER has completed.');
    }
    return this.config;
  }
}
```

```typescript
// app.config.ts
function initializeApp(configService: ConfigService): () => Observable<AppConfig> {
  return () => configService.load();
}

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withInterceptors([authInterceptor, domainErrorInterceptor])),
    provideRouter(routes),
    {
      provide: APP_INITIALIZER,
      useFactory: initializeApp,
      deps: [ConfigService],
      multi: true,
    },
    {
      provide: APP_CONFIG,
      useFactory: (configService: ConfigService) => configService.getConfig(),
      deps: [ConfigService],
    },
  ],
};
```

```typescript
// Usage in any service
@Injectable({ providedIn: 'root' })
export class OrderService {
  private readonly http = inject(HttpClient);
  private readonly config = inject(APP_CONFIG);

  getOrders$(): Observable<Order[]> {
    return this.http.get<Order[]>(`${this.config.apiBaseUrl}/orders`);
  }
}
```

## HTTP & Interceptors

### Functional Token Interceptor

```typescript
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.accessToken();

  if (token) {
    req = req.clone({ setHeaders: { Authorization: `Bearer ${token}` } });
  }
  return next(req);
};
```

### Error Interceptor

```typescript
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) {
        router.navigate(['/login']);
      }
      return throwError(() => error);
    }),
  );
};
```

### `app.config.ts` Provider Setup

```typescript
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(withInterceptors([authInterceptor, errorInterceptor])),
  ],
};
```

### Service Layer Pattern

```typescript
@Injectable({ providedIn: 'root' })
export class UserService {
  private readonly http: HttpClient;

  constructor(http: HttpClient) {
    this.http = http;
  }

  getAll$(): Observable<User[]> {
    return this.http.get<User[]>('/api/users');
  }

  getById$(id: string): Observable<User> {
    return this.http.get<User>(`/api/users/${id}`);
  }

  create$(request: CreateUserRequest): Observable<User> {
    return this.http.post<User>('/api/users', request);
  }

  delete$(id: string): Observable<void> {
    return this.http.delete<void>(`/api/users/${id}`);
  }
}
```

## Routing

### `app.routes.ts`

```typescript
export const routes: Routes = [
  { path: '', redirectTo: 'dashboard', pathMatch: 'full' },
  {
    path: 'dashboard',
    loadComponent: () => import('./dashboard/dashboard.component').then(m => m.DashboardComponent),
  },
  {
    path: 'admin',
    canActivate: [authGuard],
    loadChildren: () => import('./admin/admin.routes').then(m => m.ADMIN_ROUTES),
  },
  { path: '**', redirectTo: 'dashboard' },
];
```

### Functional Route Guard

```typescript
export const roleGuard: CanActivateFn = (route) => {
  const authService = inject(AuthService);
  const requiredRole = route.data['role'] as string;

  return authService.hasRole(requiredRole)
    ? true
    : inject(Router).createUrlTree(['/unauthorized']);
};
```

### Feature Routes File

```typescript
export const ADMIN_ROUTES: Routes = [
  { path: '', loadComponent: () => import('./admin-layout.component').then(m => m.AdminLayoutComponent),
    children: [
      { path: 'users', loadComponent: () => import('./users/user-list.component').then(m => m.UserListComponent) },
      { path: 'users/:id', loadComponent: () => import('./users/user-detail.component').then(m => m.UserDetailComponent) },
      { path: 'settings', loadComponent: () => import('./settings/settings.component').then(m => m.SettingsComponent) },
    ],
  },
];
```

## Forms

### Typed Reactive Form

```typescript
@Component({
  selector: 'app-user-form',
  standalone: true,
  imports: [ReactiveFormsModule, FormErrorComponent],
  templateUrl: './user-form.component.html',
  styleUrl: './user-form.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserFormComponent {
  private readonly fb = inject(NonNullableFormBuilder);
  readonly submitted = output<CreateUserRequest>();

  protected readonly form = this.fb.group({
    name: ['', [Validators.required, Validators.minLength(2)]],
    email: ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required, Validators.minLength(8), passwordStrengthValidator]],
  });

  protected onSubmit(): void {
    if (this.form.valid) {
      this.submitted.emit(this.form.getRawValue());
    } else {
      this.form.markAllAsTouched();
    }
  }
}
```

### Custom Validator

```typescript
export const passwordStrengthValidator: ValidatorFn = (control: AbstractControl): ValidationErrors | null => {
  const value = control.value as string;
  if (!value) return null;

  const hasUpper = /[A-Z]/.test(value);
  const hasLower = /[a-z]/.test(value);
  const hasDigit = /\d/.test(value);

  if (hasUpper && hasLower && hasDigit) return null;

  return { passwordStrength: { message: 'Must contain uppercase, lowercase, and a digit' } };
};
```

### Form Error Display Component

```typescript
@Component({
  selector: 'app-form-error',
  standalone: true,
  template: `
    @if (control()?.invalid && control()?.touched) {
      <small class="form-error">
        @if (control()?.hasError('required')) { This field is required. }
        @if (control()?.hasError('email')) { Enter a valid email address. }
        @if (control()?.hasError('minlength')) {
          Minimum length: {{ control()?.getError('minlength').requiredLength }}
        }
        @if (control()?.hasError('passwordStrength')) {
          {{ control()?.getError('passwordStrength').message }}
        }
      </small>
    }
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class FormErrorComponent {
  readonly control = input.required<AbstractControl | null>();
}
```

## Testing

### Jest Service Test

```typescript
describe('UserService', () => {
  let service: UserService;
  let httpTesting: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [provideHttpClient(), provideHttpClientTesting()],
    });
    service = TestBed.inject(UserService);
    httpTesting = TestBed.inject(HttpTestingController);
  });

  afterEach(() => httpTesting.verify());

  it('should fetch all users', () => {
    const expected: User[] = [{ id: '1', name: 'Alice', email: 'alice@test.com' }];

    service.getAll$().subscribe(users => {
      expect(users).toEqual(expected);
    });

    httpTesting.expectOne('/api/users').flush(expected);
  });
});
```

### Standalone Component Test

```typescript
describe('UserListComponent', () => {
  it('should render user names', async () => {
    const users: User[] = [
      { id: '1', name: 'Alice', email: 'alice@test.com', createdAt: new Date().toISOString() },
    ];

    await TestBed.configureTestingModule({
      imports: [UserListComponent],
    }).compileComponents();

    const fixture = TestBed.createComponent(UserListComponent);
    fixture.componentRef.setInput('users', users);
    fixture.detectChanges();

    expect(fixture.nativeElement.textContent).toContain('Alice');
  });
});
```

### Signal-Based Facade Test

```typescript
describe('ProductFacade', () => {
  let facade: ProductFacade;
  let httpTesting: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [provideHttpClient(), provideHttpClientTesting()],
    });
    facade = TestBed.inject(ProductFacade);
    httpTesting = TestBed.inject(HttpTestingController);
  });

  afterEach(() => httpTesting.verify());

  it('should set loading state during fetch', () => {
    expect(facade.loading()).toBe(false);

    facade.load();
    expect(facade.loading()).toBe(true);

    httpTesting.expectOne('/api/products').flush([]);
    expect(facade.loading()).toBe(false);
    expect(facade.products()).toEqual([]);
  });

  it('should set error state on failure', () => {
    facade.load();

    httpTesting.expectOne('/api/products').error(new ProgressEvent('error'));

    expect(facade.hasError()).toBe(true);
    expect(facade.loading()).toBe(false);
  });
});
```

### Cypress E2E Test

```typescript
describe('User Management', () => {
  beforeEach(() => {
    cy.intercept('GET', '/api/users', { fixture: 'users.json' }).as('getUsers');
    cy.visit('/admin/users');
    cy.wait('@getUsers');
  });

  it('should display user list', () => {
    cy.get('[data-cy="user-row"]').should('have.length.greaterThan', 0);
  });

  it('should navigate to user detail on click', () => {
    cy.get('[data-cy="user-row"]').first().click();
    cy.url().should('include', '/admin/users/');
    cy.get('[data-cy="user-name"]').should('be.visible');
  });
});
```

## Error Handling

### Global ErrorHandler

```typescript
@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  private readonly errorFacade = inject(ErrorFacade);

  handleError(error: unknown): void {
    const message = error instanceof Error ? error.message : 'An unexpected error occurred';
    this.errorFacade.report(message);
    console.error(error);
  }
}

// Registered in app.config.ts:
// { provide: ErrorHandler, useClass: GlobalErrorHandler }
```

### HTTP Error Interceptor with Domain Error Mapping

```typescript
export interface DomainError {
  readonly code: string;
  readonly message: string;
}

export const domainErrorInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      const domainError: DomainError = {
        code: error.error?.code ?? 'UNKNOWN',
        message: error.error?.message ?? 'An unexpected error occurred',
      };
      return throwError(() => domainError);
    }),
  );
};
```

### Component-Level Error Handling with Signals

```typescript
@Component({
  selector: 'app-product-page',
  standalone: true,
  imports: [ProductListComponent],
  template: `
    @if (facade.error(); as err) {
      <div class="alert alert-danger" role="alert">{{ err }}</div>
    } @else if (facade.loading()) {
      <app-spinner />
    } @else {
      <app-product-list [products]="facade.products()" />
    }
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ProductPageComponent implements OnInit {
  protected readonly facade = inject(ProductFacade);

  ngOnInit(): void {
    this.facade.load();
  }
}
```

## ESLint Configuration

### `eslint.config.js` (flat config)

```javascript
const angular = require('angular-eslint');
const tseslint = require('typescript-eslint');

module.exports = tseslint.config(
  {
    files: ['**/*.ts'],
    extends: [...tseslint.configs.recommended, ...angular.configs.tsRecommended],
    processor: angular.processInlineTemplates,
    rules: {
      '@angular-eslint/component-selector': ['error', { type: 'element', prefix: 'app', style: 'kebab-case' }],
      '@angular-eslint/directive-selector': ['error', { type: 'attribute', prefix: 'app', style: 'camelCase' }],
      '@angular-eslint/prefer-on-push-component-change-detection': 'error',
      '@angular-eslint/prefer-standalone': 'error',
      '@typescript-eslint/explicit-function-return-type': ['error', { allowExpressions: true }],
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
    },
  },
  {
    files: ['**/*.html'],
    extends: [...angular.configs.templateRecommended, ...angular.configs.templateAccessibility],
    rules: {
      '@angular-eslint/template/no-negated-async': 'error',
      '@angular-eslint/template/prefer-control-flow': 'error',
      '@angular-eslint/template/prefer-self-closing-tags': 'error',
    },
  },
);
```
