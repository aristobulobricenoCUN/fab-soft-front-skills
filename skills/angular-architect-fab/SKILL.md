---
name: angular-architect-fab
description: Senior Angular architecture specialist for FAB (Fabrica Architecture Base) projects. Responsible for designing, generating, analyzing and extending Angular 17+ enterprise applications following CUN architectural conventions. Applies standalone architecture, scalable feature modularization, shared UI patterns, signals-based reactive state management, provider-based HTTP configuration, reusable mock strategies, strict typing, Jest testing standards, form validation utilities, interceptor orchestration, environment isolation and maintainable folder structures. Ensures consistency, scalability, testability and long-term maintainability across the entire Angular ecosystem.
---

# Angular Architect FAB - Skill

## Propósito

Esta skill guía la creación de proyectos web Angular siguiendo la **Arquitectura FAB** (Fabrica Architecture Base). Un agente debe usar esta skill para generar, analizar o extender proyectos Angular delstack CUN.

---

## Stack Tecnológico Base

| Tecnología        | Propósito                                   |
| ----------------- | ------------------------------------------- |
| Angular 17+       | Framework principal (standalone components) |
| Angular Material  | Componentes UI base                         |
| Bootstrap 5       | Grid y utilidades CSS                       |
| Jest              | Unit testing                                |
| TypeScript strict | Tipado estricto                             |

---

## Arquitectura de Carpetas

```
src/
├── app/
│   ├── core/                          # Infraestructura singleton
│   │   ├── config/
│   │   │   └── interceptors.ts        # Registro de interceptores HTTP
│   │   └── interceptors/
│   │       ├── loading/               # Spinner interceptor (request counter)
│   │       ├── auth/                  # Token injection + 401 refresh
│   │       └── error/                 # 5xx error handling
│   │
│   ├── shared/                        # Reutilizable en todos los módulos
│   │   ├── components/
│   │   │   ├── shared-button/
│   │   │   ├── shared-input/
│   │   │   ├── shared-select/
│   │   │   ├── shared-datepicker/
│   │   │   ├── spinner/
│   │   │   └── template-modal-alert/
│   │   ├── directives/
│   │   ├── pipes/
│   │   │   └── form-control.pipe.ts   # [formControl]="control | formControl"
│   │   ├── interfaces/
│   │   │   ├── auth.interface.ts
│   │   │   ├── user.interface.ts
│   │   │   └── ...
│   │   ├── mocks/
│   │   │   └── *.mock.ts              # Faker-js factories
│   │   ├── services/
│   │   │   ├── auth/                  # AuthService (token management)
│   │   │   ├── loading/               # Loading signal service
│   │   │   ├── error-service/         # Error handling
│   │   │   └── status-alert/          # SweetAlert custom alerts
│   │   └── utils/
│   │       ├── custom-validators.ts
│   │       ├── form-errors.ts
│   │       └── string-date.utils.ts
│   │
│   ├── modules/                       # Features de la aplicación
│   │   └── [feature-name]/
│   │       ├── layout/
│   │       │   └── [feature]-layout/  # Shell component (header + router-outlet + footer)
│   │       ├── routes/
│   │       │   └── [feature].routes.ts
│   │       └── modules/
│   │           ├── [sub-feature]/
│   │           │   ├── components/
│   │           │   ├── views/
│   │           │   ├── services/
│   │           │   ├── interfaces/
│   │           │   ├── mocks/
│   │           │   └── routes/
│   │           └── [sub-feature]/
│   │
│   ├── app.routes.ts                 # Rutas principales (Lazy loading)
│   ├── app.config.ts                 # Providers globales
│   └── app.ts                        # Componente raíz
│
├── environments/
│   └── environments.ts               # URLs de APIs, configs externas
└── styles.css                       # Estilos globales
```

---

## Patrones de Arquitectura

### 1. Componentes Standalone

```typescript
@Component({
  selector: "app-my-component",
  standalone: true,
  imports: [CommonModule, MatButtonModule, FormControlPipe],
  templateUrl: "./my-component.html",
  styleUrl: "./my-component.css",
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class MyComponent implements OnInit {
  private readonly service = inject(MyService);
}
```

### 2. Signal-Based State Management

```typescript
@Injectable({ providedIn: "root" })
export class MyService {
  // Writable signal con prefijo _ para mutable
  private readonly _itemsControl = signal<Item[]>([]);

  // Readonly accessor
  public readonly items: Signal<Item[]> = this._itemsControl.asReadonly();

  // Computed signal para estado derivado
  public readonly itemCount: Signal<number> = computed(
    () => this._itemsControl().length,
  );

  // httpResource para HTTP automático
  private readonly itemsResource = httpResource<Item[]>(
    () => `${this.url}/items`,
    { defaultValue: [] },
  );
}
```

### 3. Functional HTTP Interceptors

**loading.interceptor.ts:**

```typescript
export const loadingInterceptor: HttpInterceptorFn = (req, next) => {
  const loading = inject(Loading);
  loading.show();
  return next(req).pipe(finalize(() => loading.hide()));
};
```

**auth.interceptor.ts:**

```typescript
export const AuthInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthService);

  if (isAuthApi(req.url)) {
    return next(req);
  }

  const token = auth.getToken();
  const authReq = req.clone({
    headers: req.headers.set("Authorization", `Bearer ${token}`),
  });

  return next(authReq).pipe(
    catchError((error) => {
      if (error.status === 401) {
        return auth.refreshToken().pipe(
          switchMap((newToken) =>
            next(
              req.clone({
                headers: req.headers.set("Authorization", `Bearer ${newToken}`),
              }),
            ),
          ),
        );
      }
      if (error.status >= 500) {
        inject(ErrorService).handle(error);
      }
      return throwError(() => error);
    }),
  );
};
```

### 4. Lazy-Loaded Routes

```typescript
// [feature].routes.ts
export const MyFeatureRoutes: Routes = [
  {
    path: "",
    loadComponent: () =>
      import("../views/my-feature/my-feature").then(
        (m) => m.MyFeatureComponent,
      ),
  },
];

// app.routes.ts
export const routes: Routes = [
  { path: "", redirectTo: "feature", pathMatch: "full" },
  ...MyFeatureRoutes,
  { path: "**", redirectTo: "feature" },
];
```

### 5. Layout con Hijos

```typescript
{
  path: 'feature',
  component: FeatureLayoutComponent,
  canActivate: [authGuard],
  children: [
    ...SubFeatureRoutes,
    { path: '**', redirectTo: 'sub-feature' }
  ]
}
```

### 6. inject() Over Constructor

```typescript
@Injectable({ providedIn: "root" })
export class AuthService {
  private readonly http = inject(HttpClient);
  private readonly appConfig = inject(APP_CONFIG);
}
```

### 7. DestroyRef para Cleanup

```typescript
export class MyComponent implements OnInit {
  private readonly destroy = inject(DestroyRef);

  ngOnInit(): void {
    this.route.queryParams.pipe(takeUntilDestroyed(this.destroy)).subscribe();
  }
}
```

### 8. lastValueFrom para HTTP

```typescript
async getItems(): Promise<void> {
  try {
    const data = await lastValueFrom(
      this.http.get<Item[]>(`${this.url}/items`)
    );
    this._itemsControl.set(data);
  } catch (e) {
    this._itemsControl.set([]);
    if (e instanceof HttpErrorResponse) {
      controlAlertHttp(e);
    }
  }
}
```

---

## Convenciones de Código

### TypeScript

- `strict: true`
- `noImplicitOverride: true`
- `strictTemplates: true`
- NO usar `any`
- Prefijo `_` para miembros privados escribibles: `_itemsControl`
- Interfaces SIN prefijo "I": `User` no `IUser`

### Archivos

| Tipo        | Convención              | Ejemplo                     |
| ----------- | ----------------------- | --------------------------- |
| Componentes | PascalCase.component.ts | `my-component.component.ts` |
| Servicios   | PascalCase.service.ts   | `auth.service.ts`           |
| Interfaces  | PascalCase.interface.ts | `user.interface.ts`         |
| Mocks       | PascalCase.mock.ts      | `user.mock.ts`              |
| Utils       | kebab-case.utils.ts     | `form-errors.utils.ts`      |
| Pipes       | PascalCase.pipe.ts      | `form-control.pipe.ts`      |
| Tests       | \*.spec.ts              | `auth.service.spec.ts`      |
| Templates   | kebab-case.html         | `my-component.html`         |

### Imports (orden)

1. Angular core/external
2. Componentes/pipes standalone de Angular
3. Third-party imports
4. Internal `@shared`, `@core`
5. Relative imports

---

## Mock Pattern

```typescript
// user.mock.ts
import { faker } from "@faker-js/faker";

faker.seed(123);

export const createMockUser = (overrides?: Partial<User>): User => ({
  id: faker.string.uuid(),
  name: faker.person.fullName(),
  email: faker.internet.email(),
  ...overrides,
});

export const createMockUsers = (count: number = 5): User[] =>
  Array.from({ length: count }, () => createMockUser());
```

---

## Servicios Core Requeridos

### AuthService

- `login(credentials): Promise<TokenResponse>`
- `logout(): void`
- `getToken(): string`
- `refreshToken(): Observable<Token>`
- `isAuthenticated(): boolean`

### LoadingService

```typescript
// signals-based
const _loading = signal(false);
export const loading = _loading.asReadonly();
export const show = () => _loading.set(true);
export const hide = () => _loading.set(false);
```

### ErrorService

- `handle(error: HttpErrorResponse): void`
- `handleAlertHttp(error: HttpErrorResponse): void`

---

## Guards

| Guard                  | Propósito                    |
| ---------------------- | ---------------------------- |
| `auth.guard.ts`        | Protege rutas autenticadas   |
| `auth-logged.guard.ts` | Redirige si ya está logueado |
| `role.guard.ts`        | Acceso basado en roles       |

```typescript
export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);

  if (auth.isAuthenticated()) {
    return true;
  }

  return router.createUrlTree(["/login"]);
};
```

---

## Environments

```typescript
export const environment = {
  production: false,
  cdn: "https://apicdn.cunapp.pro/fabrica-utils",
  apiBase: {
    url: "https://api.example.com/api/v1",
    username: "USER",
    password: "PASS",
  },
  // ... APIs específicas del proyecto
};
```

---

## Testing

### Jest Setup

- Usar `@faker-js/faker` para mocks
- `jest.clearAllMocks()` en `afterEach`
- `httpMock.verify()` para verificar requests
- Mocks en `**/*.mock.ts`

### Test Pattern

```typescript
describe("MyService", () => {
  let service: MyService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [provideHttpClient(), provideHttpClientTesting(), MyService],
    });
    service = TestBed.inject(MyService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  it("should fetch items", () => {
    const mockItems = createMockItems();

    service.getItems();

    const req = httpMock.expectOne("/api/items");
    req.flush(mockItems);

    expect(service.items()).toEqual(mockItems);
  });
});
```

---

## Form Validation

Usar utilidades compartidas en `@shared/utils/form/form-errors.ts`:

```typescript
// Template
<div *ngIf="isInvalidField('email', form)">
  {{ getFieldError('email', form) }}
</div>

// No acceder directamente a control.errors
```

---

## Angular Material Components Comunes

- `MatIconModule` - Iconos
- `MatButtonModule` - Botones
- `MatInputModule` - Inputs
- `MatSelectModule` - Selects
- `MatPaginatorModule` - Paginación
- `MatExpansionModule` - Paneles expandibles
- `MatCheckboxModule` - Checkboxes
- `MatChipsModule` - Chips
- `MatDialogModule` - Diálogos
- `MatSnackBarModule` - Notificaciones

---

## Lista de Patrones a Implementar por Proyecto

Al crear un nuevo proyecto, verificar:

- [ ] Estructura de carpetas según plantilla
- [ ] Interceptor de loading (request counter)
- [ ] Interceptor de auth (token injection + 401 refresh)
- [ ] Interceptor de error (5xx handling)
- [ ] AuthService con token management
- [ ] LoadingService con signals
- [ ] ErrorService con SweetAlert
- [ ] FormControlPipe
- [ ] Shared components (button, input, select, spinner, modal-alert)
- [ ] Guards (auth, auth-logged)
- [ ] Environments.ts con APIs del proyecto
- [ ] Lazy-loaded routes
- [ ] Layout component con header/footer
- [ ] OnPush en todos los componentes
- [ ] Mocks con faker-js
- [ ] Jest configured

---

## Contexto Requerido del Proyecto

Para generar un proyecto específico, obtener:

1. **Nombre del proyecto**
2. **Descripción breve**
3. **Lista de APIs** (URLs, credenciales)
4. **Lista de módulos/features**
5. **Integraciones externas** (Firebase, Zoho, etc.)
6. **Componentes shared requeridos**
7. **Guards específicos**
8. **Rutas principales**
9. **Reglas adicionales** (si aplica)
