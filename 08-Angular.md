# 8. Angular — Deep Dive

> **Goal:** Understand how Angular's building blocks fit together, master component/service patterns, RxJS, forms, routing, and be productive in any Angular codebase.

---

## 8.0 What is Angular & Why It Exists

Angular is Google's opinionated framework for building **Single Page Applications (SPAs)** — apps that live in the browser and behave like desktop apps (no full page reloads).

**Key value:**
- **TypeScript** (typed JS) — catches errors before runtime
- **Component-based** — reusable UI blocks
- **Built-in** everything — routing, forms, HTTP, DI, testing (no need to pick libraries)
- **RxJS** for reactive programming
- **CLI-driven** — `ng new`, `ng generate component`, `ng test`

### Angular vs AngularJS

| AngularJS (1.x) | Angular (2+) |
|-----------------|--------------|
| JavaScript | **TypeScript** |
| Controllers + $scope | Components |
| 2-way binding default | One-way by default; explicit 2-way with `[(...)]` |
| $watch loops | Zone.js + Change Detection |
| No CLI | Full-featured CLI (`@angular/cli`) |

Angular 2 was a complete rewrite. "Angular" today means version 2+ (currently 17+).

---

## 8.1 Getting Started

```bash
npm install -g @angular/cli
ng new my-app                  # scaffold
cd my-app
ng serve                        # dev server → http://localhost:4200

ng generate component book      # or: ng g c book
ng generate service book        # ng g s book
ng generate module admin        # ng g m admin
```

### Project structure (relevant parts)

```
src/
├── app/
│   ├── app.module.ts           (root module)
│   ├── app.component.ts        (root component)
│   ├── app.component.html
│   ├── app.component.css
│   ├── app-routing.module.ts
│   └── book/
│       ├── book.component.ts
│       └── book.component.html
├── assets/
├── environments/                (env-specific configs)
├── main.ts                      (bootstrap entry)
└── index.html
```

---

## 8.2 Components — The Building Blocks

Every Angular UI element is a **component**. Each component has:
1. **Template** (HTML)
2. **Class** (TypeScript logic)
3. **Styles** (CSS/SCSS)
4. **Metadata** (`@Component` decorator)

### Anatomy

```typescript
// book.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-book',                 // <app-book></app-book> in parent HTML
  templateUrl: './book.component.html',
  styleUrls: ['./book.component.css']
})
export class BookComponent {
  title = 'My Book';                    // property
  count = 0;

  onClick() {                            // method
    this.count++;
  }
}
```

```html
<!-- book.component.html -->
<h2>{{ title }}</h2>                     <!-- interpolation -->
<p>Clicked {{ count }} times</p>
<button (click)="onClick()">Click me</button>
```

Use it in another template:
```html
<app-book></app-book>
```

### Data Binding — 4 flavors

| Flavor | Syntax | Direction | Example |
|--------|--------|-----------|---------|
| Interpolation | `{{ value }}` | TS → HTML | `<h1>{{ name }}</h1>` |
| Property binding | `[prop]="value"` | TS → HTML | `<img [src]="url">` |
| Event binding | `(event)="handler()"` | HTML → TS | `<button (click)="save()">` |
| Two-way binding | `[(ngModel)]="value"` | Both | `<input [(ngModel)]="name">` |

Two-way binding requires `FormsModule`.

### Input & Output — Parent ↔ Child

```typescript
// child.component.ts
export class BookComponent {
  @Input() book!: Book;                          // parent → child
  @Output() delete = new EventEmitter<number>(); // child → parent

  remove() {
    this.delete.emit(this.book.id);
  }
}
```

```html
<!-- parent.component.html -->
<app-book
  [book]="selectedBook"
  (delete)="onDelete($event)">
</app-book>
```

### View Encapsulation

By default, styles in a component are **scoped** to that component (via generated attributes). Don't leak globally!

```typescript
@Component({
  ...
  encapsulation: ViewEncapsulation.None      // if you want global CSS
})
```

---

## 8.3 Modules — Grouping Related Things

Modules bundle related components, directives, pipes, and services.

```typescript
@NgModule({
  declarations: [AppComponent, BookComponent, BookListComponent],
  imports: [BrowserModule, FormsModule, HttpClientModule, AppRoutingModule],
  providers: [BookService],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

- **declarations** — components/directives/pipes owned by this module
- **imports** — other modules whose exports we use
- **providers** — services available in this module
- **exports** — what other modules importing us can use
- **bootstrap** — root component (only in AppModule)

### Feature modules

Split your app by feature (admin, user, checkout):

```
app/
├── app.module.ts
├── admin/
│   ├── admin.module.ts
│   ├── admin-routing.module.ts
│   └── components/
├── shared/                  (reusable components/pipes)
│   └── shared.module.ts
└── core/                    (singletons: auth, interceptors)
    └── core.module.ts
```

**Lazy-loaded modules** — loaded only when route is visited (smaller initial bundle):
```typescript
{ path: 'admin', loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule) }
```

### Standalone Components (Angular 14+)

Modern Angular allows components without NgModule:

```typescript
@Component({
  standalone: true,
  selector: 'app-book',
  imports: [CommonModule, FormsModule],
  template: `<h1>{{ title }}</h1>`
})
export class BookComponent {
  title = 'Java';
}
```

Bootstrap standalone:
```typescript
// main.ts
bootstrapApplication(AppComponent, {
  providers: [provideRouter(routes), provideHttpClient()]
});
```

---

## 8.4 Services & Dependency Injection

Services hold business logic and shared state — kept out of components for cleanness and testability.

```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })    // singleton app-wide
export class BookService {
  private apiUrl = '/api/books';

  constructor(private http: HttpClient) {}      // DI via constructor

  getBooks(): Observable<Book[]> {
    return this.http.get<Book[]>(this.apiUrl);
  }

  getBook(id: number): Observable<Book> {
    return this.http.get<Book>(`${this.apiUrl}/${id}`);
  }

  create(book: Book): Observable<Book> {
    return this.http.post<Book>(this.apiUrl, book);
  }

  update(id: number, book: Book): Observable<Book> {
    return this.http.put<Book>(`${this.apiUrl}/${id}`, book);
  }

  delete(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`);
  }
}
```

### Provider scopes

- `providedIn: 'root'` — singleton, whole app (most common)
- `providedIn: 'any'` — separate instance per lazy module
- `providedIn: SomeModule` — per module
- Component-level `providers: []` — new instance per component

### Using a service

```typescript
export class BookListComponent implements OnInit {
  books: Book[] = [];

  constructor(private bookService: BookService) {}     // injected

  ngOnInit() {
    this.bookService.getBooks().subscribe(data => this.books = data);
  }
}
```

---

## 8.5 Routing — Multiple "Pages"

```typescript
// app-routing.module.ts
const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'books', component: BookListComponent },
  { path: 'books/:id', component: BookDetailComponent },
  { path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule),
    canActivate: [AuthGuard] },
  { path: '**', component: NotFoundComponent }              // catch-all
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule { }
```

Template:
```html
<nav>
  <a routerLink="/" routerLinkActive="active">Home</a>
  <a routerLink="/books" routerLinkActive="active">Books</a>
</nav>

<router-outlet></router-outlet>                            <!-- component renders here -->
```

### Route parameters

```typescript
export class BookDetailComponent implements OnInit {
  book?: Book;

  constructor(
    private route: ActivatedRoute,
    private router: Router,
    private bookService: BookService
  ) {}

  ngOnInit() {
    // observable — updates if URL param changes
    this.route.paramMap.subscribe(pm => {
      const id = +pm.get('id')!;
      this.bookService.getBook(id).subscribe(b => this.book = b);
    });
  }

  goBack() {
    this.router.navigate(['/books']);
  }

  goToEdit() {
    this.router.navigate(['/books', this.book!.id, 'edit'], {
      queryParams: { returnUrl: '/books' }
    });
  }
}
```

### Route Guards

Protect routes. Guards return `boolean | UrlTree | Observable<boolean>`.

```typescript
@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate {
  constructor(private auth: AuthService, private router: Router) {}

  canActivate(): boolean | UrlTree {
    if (this.auth.isLoggedIn()) return true;
    return this.router.createUrlTree(['/login']);
  }
}
```

Types: `CanActivate`, `CanActivateChild`, `CanDeactivate` (leaving page — prompt for unsaved changes), `Resolve` (fetch data before route activates), `CanLoad` (block lazy load).

### Lazy loading benefits

- Smaller initial bundle
- Faster first load
- Modules loaded only when needed

---

## 8.6 RxJS — Reactive Programming in Angular

Angular is built on RxJS. `HttpClient`, `Router`, forms all return **Observables**.

### Observable = stream of values over time

```typescript
import { Observable, of, from, interval, fromEvent } from 'rxjs';

// From values
of(1, 2, 3);                              // emits 1, 2, 3, then completes
from([1, 2, 3]);                          // same
from(fetch('/api'));                      // wraps Promise

// From timing
interval(1000);                           // emits 0, 1, 2, ... every 1s

// From events
fromEvent(document.getElementById('btn')!, 'click');

// Custom
const obs$ = new Observable(sub => {
  sub.next(1);
  sub.next(2);
  setTimeout(() => { sub.next(3); sub.complete(); }, 1000);
});
```

Subscribe:
```typescript
obs$.subscribe({
  next: v => console.log(v),
  error: e => console.error(e),
  complete: () => console.log('Done')
});
```

### Operators — transform / combine streams

```typescript
import { map, filter, switchMap, mergeMap, debounceTime,
         distinctUntilChanged, catchError, retry, tap, take } from 'rxjs/operators';

// Common pipeline
this.searchInput.valueChanges.pipe(
    debounceTime(300),                     // wait 300ms of inactivity
    distinctUntilChanged(),                // only if changed
    switchMap(q => this.bookService.search(q))  // cancel previous, start new
).subscribe(results => this.results = results);
```

### Key operators

| Operator | Purpose |
|----------|---------|
| `map(fn)` | Transform each value |
| `filter(fn)` | Only pass values matching predicate |
| `tap(fn)` | Side effect (log, debug) without changing stream |
| `take(n)` | Take first N and complete |
| `debounceTime(ms)` | Emit only after quiet period |
| `distinctUntilChanged` | Emit only when value differs from last |
| `catchError(fn)` | Handle errors |
| `retry(n)` | Auto-retry on error |

### Flattening operators — the big four

When you have an Observable that produces Observables (nested), you flatten:

| Operator | Behavior |
|----------|----------|
| **switchMap** | Cancel previous inner obs when new arrives (SEARCH boxes!) |
| **mergeMap** | Run all in parallel; results interleaved |
| **concatMap** | Queue; run one after another, keep order |
| **exhaustMap** | Ignore new emissions while one is running (LOGIN — prevents double-clicks) |

```typescript
// Search — cancel stale queries
input.valueChanges.pipe(
  switchMap(q => this.service.search(q))
);

// Save multiple items — one at a time in order
saveButton.pipe(
  concatMap(item => this.service.save(item))
);

// Login — prevent double submit
loginButton.pipe(
  exhaustMap(creds => this.auth.login(creds))
);
```

### Combining streams

```typescript
combineLatest([a$, b$]).subscribe(([a, b]) => {});      // whenever either emits
forkJoin([a$, b$, c$]).subscribe(([a, b, c]) => {});    // wait for all to complete
merge(a$, b$);                                           // just merge into one stream
concat(a$, b$);                                          // finish a$ then start b$
```

### Subjects — Observable + Observer

Multi-cast events. You can call `next()` on a Subject.

| Type | Behavior |
|------|----------|
| **Subject** | No initial value, new subscribers get only future emissions |
| **BehaviorSubject** | Has initial value; new subscribers get **last value** immediately |
| **ReplaySubject** | New subscribers get N previous values |
| **AsyncSubject** | Emits only the **last** value on complete |

**Shared state via BehaviorSubject:**
```typescript
@Injectable({ providedIn: 'root' })
export class CartService {
  private itemsSubject = new BehaviorSubject<Item[]>([]);
  readonly items$ = this.itemsSubject.asObservable();

  add(item: Item) {
    const current = this.itemsSubject.value;
    this.itemsSubject.next([...current, item]);
  }

  get itemCount$() {
    return this.items$.pipe(map(items => items.length));
  }
}
```

Usage:
```html
<span>Cart: {{ cart.itemCount$ | async }} items</span>
```

### Memory leaks & unsubscribing

If you `.subscribe()`, you must eventually unsubscribe or you'll leak.

**Option 1: manual**
```typescript
export class BookListComponent implements OnInit, OnDestroy {
  private sub!: Subscription;

  ngOnInit() {
    this.sub = this.bookService.getBooks().subscribe(...);
  }

  ngOnDestroy() {
    this.sub.unsubscribe();
  }
}
```

**Option 2: takeUntil pattern (best for many subs)**
```typescript
export class BookListComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit() {
    this.bookService.getBooks()
      .pipe(takeUntil(this.destroy$))
      .subscribe(...);
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**Option 3: async pipe (best!)**
Auto-subscribes AND unsubscribes.
```typescript
books$ = this.bookService.getBooks();      // just assign, don't subscribe
```
```html
<li *ngFor="let b of books$ | async">{{ b.title }}</li>
```

**Angular 16+:** `takeUntilDestroyed()` — cleanup without needing OnDestroy!

---

## 8.7 Forms — Two Approaches

### Template-Driven Forms — for simple forms

Uses directives in HTML.

```html
<form #f="ngForm" (ngSubmit)="onSubmit(f)">
  <input name="title" [(ngModel)]="book.title" required minlength="3">
  <input name="price" [(ngModel)]="book.price" type="number" required>
  <button type="submit" [disabled]="f.invalid">Save</button>
</form>
```

Needs `FormsModule` in module.

### Reactive Forms — for complex forms

Programmatic, type-safe, testable.

```typescript
import { FormBuilder, Validators } from '@angular/forms';

export class BookFormComponent {
  form = this.fb.group({
    title: ['', [Validators.required, Validators.minLength(3)]],
    price: [0, [Validators.required, Validators.min(0)]],
    author: this.fb.group({
      name:  ['', Validators.required],
      email: ['', [Validators.required, Validators.email]]
    }),
    tags: this.fb.array([this.fb.control('')])       // dynamic array
  });

  constructor(private fb: FormBuilder) {}

  get tags() {
    return this.form.get('tags') as FormArray;
  }

  addTag() { this.tags.push(this.fb.control('')); }
  removeTag(i: number) { this.tags.removeAt(i); }

  onSubmit() {
    if (this.form.valid) {
      console.log(this.form.value);
    }
  }
}
```

```html
<form [formGroup]="form" (ngSubmit)="onSubmit()">
  <input formControlName="title">
  <div *ngIf="form.get('title')?.errors?.['required']">Title required</div>
  <div *ngIf="form.get('title')?.errors?.['minlength']">Too short</div>

  <div formGroupName="author">
    <input formControlName="name">
    <input formControlName="email">
  </div>

  <div formArrayName="tags">
    <div *ngFor="let t of tags.controls; let i = index">
      <input [formControlName]="i">
      <button type="button" (click)="removeTag(i)">Remove</button>
    </div>
    <button type="button" (click)="addTag()">Add tag</button>
  </div>

  <button [disabled]="form.invalid">Save</button>
</form>
```

Needs `ReactiveFormsModule`.

### Template-Driven vs Reactive

| | Template | Reactive |
|-|----------|----------|
| Setup | Directives in HTML | TS-based |
| Model | Auto (from ngModel) | Explicit FormGroup |
| Complexity | Easy for simple | Better for complex |
| Testing | Harder | Easier |
| Dynamic controls | Harder | FormArray easy |

**Rule:** default to Reactive. Template-Driven is fine for tiny forms.

### Custom Validators

```typescript
function forbiddenName(name: RegExp): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const forbidden = name.test(control.value);
    return forbidden ? { forbiddenName: { value: control.value } } : null;
  };
}

// Use
title: ['', [Validators.required, forbiddenName(/admin/i)]]
```

Async validator (checks server):
```typescript
function uniqueTitle(service: BookService): AsyncValidatorFn {
  return control =>
    control.valueChanges.pipe(
      debounceTime(300),
      switchMap(title => service.checkUnique(title)),
      map(exists => (exists ? { taken: true } : null)),
      first()
    );
}
```

---

## 8.8 Directives

Change DOM behavior. Two kinds:

### Structural directives — change DOM structure (prefixed with `*`)

```html
<div *ngIf="isVisible">Shown when isVisible is true</div>

<div *ngIf="user; else guest">Hello {{ user.name }}</div>
<ng-template #guest>Please log in</ng-template>

<li *ngFor="let book of books; let i = index; trackBy: trackById">
  {{ i + 1 }}. {{ book.title }}
</li>

<div [ngSwitch]="status">
  <span *ngSwitchCase="'active'">✅</span>
  <span *ngSwitchCase="'pending'">⏳</span>
  <span *ngSwitchDefault>❌</span>
</div>
```

**trackBy** improves `*ngFor` performance — Angular reuses DOM instead of destroying/recreating:
```typescript
trackById(index: number, item: Book) { return item.id; }
```

### Attribute directives — change appearance/behavior

```html
<div [ngClass]="{active: isActive, error: hasError}"></div>
<div [ngStyle]="{color: color, fontSize: size + 'px'}"></div>
```

### Custom directive

```typescript
@Directive({ selector: '[appHighlight]' })
export class HighlightDirective {
  @Input('appHighlight') color = 'yellow';

  constructor(private el: ElementRef) {}

  @HostListener('mouseenter') onEnter() {
    this.el.nativeElement.style.background = this.color;
  }
  @HostListener('mouseleave') onLeave() {
    this.el.nativeElement.style.background = '';
  }
}
```

```html
<p [appHighlight]="'lightblue'">Hover me</p>
```

---

## 8.9 Pipes — Transform Values in Templates

Built-in pipes:
```html
{{ price | currency:'USD' }}          <!-- $499.00 -->
{{ price | currency:'INR':'symbol' }} <!-- ₹499.00 -->
{{ date | date:'yyyy-MM-dd HH:mm' }}
{{ 'HELLO' | lowercase }}
{{ 'hello' | uppercase }}
{{ items | slice:0:5 }}                <!-- first 5 -->
{{ obj | json }}
{{ observable$ | async }}
```

### Chain pipes
```html
{{ birthDate | date:'medium' | uppercase }}
```

### Custom pipe

```typescript
@Pipe({ name: 'truncate', standalone: true })
export class TruncatePipe implements PipeTransform {
  transform(value: string, limit = 20, suffix = '…'): string {
    if (!value || value.length <= limit) return value;
    return value.substring(0, limit) + suffix;
  }
}
```

```html
{{ description | truncate:50 }}
```

### Pure vs Impure

- **Pure (default)** — recomputes only when input reference changes. Fast.
- **Impure** — recomputes on every change detection cycle. Slow.

```typescript
@Pipe({ name: 'filterActive', pure: false })
```

Rule: keep pipes pure. For dynamic filtering, compute in the component.

---

## 8.10 Lifecycle Hooks

Order of execution:

```
constructor()                            // DI happens here
    ↓
ngOnChanges(changes: SimpleChanges)      // when @Input changes (called with a map)
    ↓
ngOnInit()                                // AFTER first ngOnChanges — do init here!
    ↓
ngDoCheck()                               // custom change detection (advanced)
    ↓
ngAfterContentInit()                      // content projection children ready
ngAfterContentChecked()
    ↓
ngAfterViewInit()                         // view + child views ready (safe to use @ViewChild)
ngAfterViewChecked()
    ↓
… changes happen …
    ↓
ngOnDestroy()                             // cleanup: unsubscribe, cancel timers
```

Practical:
```typescript
export class BookComponent implements OnInit, OnDestroy {
  private sub!: Subscription;

  constructor(private bookService: BookService) {
    // Don't call services here — @Input isn't set yet
  }

  ngOnInit() {
    // Fetch data, subscribe to observables
    this.sub = this.bookService.getBooks().subscribe(...);
  }

  ngOnDestroy() {
    this.sub?.unsubscribe();
  }
}
```

**Rule:** never fetch data in `constructor`. Always in `ngOnInit`.

---

## 8.11 Change Detection

Angular detects when data changes and updates the DOM.

### Zone.js — the magic

Zone.js patches all async APIs (`setTimeout`, `Promise`, XHR, events). When any completes, Angular triggers change detection on the entire component tree.

### Strategies

- **Default** (`ChangeDetectionStrategy.Default`) — checks whole subtree on every event
- **OnPush** — checks only when:
  - An `@Input` reference changes
  - An event fires within the component
  - An observable fires (with `async` pipe)
  - Manual trigger (`ChangeDetectorRef.markForCheck()`)

```typescript
@Component({
  selector: 'app-book',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `...`
})
export class BookComponent { }
```

**Best practice:** use `OnPush` + immutable inputs + `async` pipe for performance.

### Immutable updates

```typescript
// ❌ mutating — OnPush won't detect
this.book.title = 'New';

// ✅ new reference — OnPush detects
this.book = { ...this.book, title: 'New' };
this.books = [...this.books, newBook];
```

---

## 8.12 HTTP Client

```typescript
import { HttpClient, HttpParams, HttpHeaders } from '@angular/common/http';

constructor(private http: HttpClient) {}

// GET
this.http.get<Book[]>('/api/books').subscribe(res => this.books = res);

// With params/headers
const params = new HttpParams()
  .set('page', 0)
  .set('size', 10)
  .set('sort', 'title,asc');
const headers = new HttpHeaders({ 'X-Api-Key': 'abc' });
this.http.get<Book[]>('/api/books', { params, headers });

// POST
this.http.post<Book>('/api/books', newBook).subscribe();

// PUT
this.http.put<Book>(`/api/books/${id}`, book).subscribe();

// DELETE
this.http.delete<void>(`/api/books/${id}`).subscribe();

// Error handling
this.http.get<Book[]>('/api/books').pipe(
  retry(3),
  catchError(err => {
    console.error(err);
    return of([]);      // fallback empty list
  })
).subscribe(...);
```

### HTTP Interceptors — cross-cutting concerns

Add auth headers, log, handle errors globally.

```typescript
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler) {
    const token = localStorage.getItem('token');
    if (token) {
      req = req.clone({
        setHeaders: { Authorization: `Bearer ${token}` }
      });
    }
    return next.handle(req);
  }
}

@Injectable()
export class ErrorInterceptor implements HttpInterceptor {
  constructor(private router: Router) {}
  intercept(req: HttpRequest<any>, next: HttpHandler) {
    return next.handle(req).pipe(
      catchError(err => {
        if (err.status === 401) this.router.navigate(['/login']);
        return throwError(() => err);
      })
    );
  }
}

// Register
providers: [
  { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true },
  { provide: HTTP_INTERCEPTORS, useClass: ErrorInterceptor, multi: true }
]
```

Standalone equivalent (Angular 15+):
```typescript
provideHttpClient(withInterceptors([authInterceptorFn, errorInterceptorFn]))
```

---

## 8.13 State Management (brief)

For small apps: `BehaviorSubject` in services (see 8.6).

For large apps:
- **NgRx** — Redux-inspired, actions/reducers/effects/selectors
- **Akita** — simpler, more OO
- **NGXS** — decorator-based
- **Signals** (Angular 16+) — reactive primitives, may replace RxJS for state

### Signals sneak peek

```typescript
export class Counter {
  count = signal(0);
  double = computed(() => this.count() * 2);

  increment() {
    this.count.update(v => v + 1);
  }
}
```

```html
<p>{{ count() }} — doubled: {{ double() }}</p>
<button (click)="increment()">+</button>
```

Cleaner than BehaviorSubject for local state.

---

## 8.14 Testing

```typescript
// book.service.spec.ts
describe('BookService', () => {
  let service: BookService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [BookService]
    });
    service = TestBed.inject(BookService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  it('should fetch books', () => {
    const mock: Book[] = [{ id: 1, title: 'Java' }];
    service.getBooks().subscribe(books => {
      expect(books).toEqual(mock);
    });
    const req = httpMock.expectOne('/api/books');
    expect(req.request.method).toBe('GET');
    req.flush(mock);
  });

  afterEach(() => httpMock.verify());
});
```

Component testing with `TestBed`, `ComponentFixture`, `DebugElement`.

E2E: Playwright / Cypress.

---

## 8.15 Complete Mini App Structure

```
src/app/
├── app.module.ts / app.config.ts
├── app.component.ts
├── app-routing.module.ts
├── core/                        (singletons — services, interceptors, guards)
│   ├── auth/
│   │   ├── auth.service.ts
│   │   ├── auth.guard.ts
│   │   └── auth.interceptor.ts
│   └── error.interceptor.ts
├── shared/                      (reusable components/pipes/directives)
│   ├── components/
│   ├── pipes/
│   └── shared.module.ts
├── features/                     (feature modules — lazy loaded)
│   ├── book/
│   │   ├── book.module.ts
│   │   ├── book-routing.module.ts
│   │   ├── components/
│   │   │   ├── book-list/
│   │   │   ├── book-detail/
│   │   │   └── book-form/
│   │   ├── services/book.service.ts
│   │   └── models/book.model.ts
│   └── order/
└── models/                       (or per-feature)
```

---

## 8.16 Interview Question Bank

1. **Angular vs React vs Vue?**  
   Angular: opinionated full framework, TypeScript, DI, RxJS. React: library, JSX, more freedom. Vue: middle ground, easy learning curve.

2. **Component vs Directive vs Pipe?**  
   Component: has template. Directive: modifies existing DOM (no template). Pipe: transforms template values.

3. **What is DI in Angular?**  
   Angular's built-in dependency injection — services are injected via constructor from a hierarchical injector.

4. **Template-driven vs Reactive forms?**  
   Template: directives in HTML, easy. Reactive: TS-driven FormGroup, better for complex/dynamic.

5. **What is Observable? Subject types?**  
   Async stream of values. Subject (basic), BehaviorSubject (initial + last), ReplaySubject (buffer), AsyncSubject (only last on complete).

6. **switchMap vs mergeMap vs concatMap vs exhaustMap?**  
   switchMap: cancel prev. mergeMap: parallel. concatMap: sequential. exhaustMap: ignore new while busy.

7. **Explain lifecycle hooks.**  
   ngOnChanges → ngOnInit → ngDoCheck → ngAfterContentInit/Checked → ngAfterViewInit/Checked → ngOnDestroy.

8. **Change Detection strategies?**  
   Default (check whole tree on any event). OnPush (only when Input reference changes or event/observable inside).

9. **What is Lazy Loading?**  
   Load feature module only when its route is visited. `loadChildren: () => import(...)`.

10. **How to communicate between components?**  
    Parent→Child: `@Input`. Child→Parent: `@Output` EventEmitter. Siblings/anywhere: shared service with Subject.

11. **What is a Route Guard?**  
    Interface (CanActivate, CanDeactivate, Resolve, CanLoad) that controls route access.

12. **What is `ngModel`?**  
    Two-way binding directive from `FormsModule`.

13. **What is a Resolver?**  
    Fetches data before the route activates so the component has it on load.

14. **What is `async` pipe?**  
    Auto-subscribes to Observable/Promise in template and unsubscribes on destroy.

15. **How to prevent memory leaks?**  
    Unsubscribe in ngOnDestroy, use `takeUntil` pattern, or better — `async` pipe.

16. **What is Zone.js?**  
    Monkey-patches async APIs; triggers change detection after any async op completes.

17. **What are Standalone Components?**  
    Components without NgModule (Angular 14+). Simpler; the future direction.

18. **Signals vs RxJS?**  
    Signals: sync reactive primitive for local state. RxJS: async streams. Complement each other.

19. **HTTP Interceptor use cases?**  
    Auth tokens, logging, error handling, retries, cache.

20. **What is `trackBy` in `*ngFor`?**  
    Tells Angular how to identify items → reuses DOM instead of destroying/recreating on list changes. Big perf win.

---

## 8.17 Cheat Sheet

```
Component: template + class + styles + @Component
Data binding: {{ }} | [prop] | (event) | [(ngModel)]
Parent → Child: @Input; Child → Parent: @Output EventEmitter
Service: @Injectable({ providedIn: 'root' }); DI via constructor
Router: routes[] + <router-outlet>, use guards + lazy loading
RxJS: switchMap for search, exhaustMap for login
Unsub: takeUntil / async pipe / takeUntilDestroyed
Reactive forms > Template-driven for complex forms
OnPush + immutable inputs + async pipe = fast app
Signals for local state (Angular 16+)
```

---

## Practical Assignments

1. Build a book list app: list, detail (route param), add/edit/delete via reactive forms and HTTP.
2. Implement a search input with `debounceTime` + `switchMap`.
3. Add `AuthGuard` and JWT `HttpInterceptor`.
4. Convert a component to `OnPush` and make its inputs immutable.
5. Create a shared `CartService` with BehaviorSubject; display badge with `async` pipe.
6. Add a custom `truncate` pipe and a `highlight` directive.

Master these and Angular interviews are yours. 🚀