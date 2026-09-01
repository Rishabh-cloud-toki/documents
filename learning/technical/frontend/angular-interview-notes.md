1. Core Angular Concepts (High Priority - Must Know)
    - Component-based architecture (Decorators, Lifecycle Hooks) - DONE
    - Modules & Dependency Injection - DONE
    - Directives (Structural & Attribute Directives) - DONE
    - Pipes & Custom Pipes
    - Services & RxJS for state management - DONE
    - Forms (Template-driven & Reactive Forms) - DONE
2. RxJS & State Management (Critical for Senior Developers) - DONE
    - Observables & Subjects
    - Operators (mergeMap, switchMap, concatMap, debounceTime, etc.)
    - BehaviorSubject vs. ReplaySubject
    - NgRx vs. Services for state management
3. Performance Optimization (Commonly Asked)
    - Change Detection Strategy (OnPush vs. Default)
    - Lazy Loading & Route Preloading
    - Virtual Scrolling & Efficient DOM Updates
    - Using TrackBy with *ngFor
4. Angular Routing & Guards
    - RouterModule, Lazy Loading
    - Route Guards (CanActivate, CanDeactivate)
    - Resolvers for Preloading Data
5. Testing in Angular
    - Unit Testing with Jasmine/Karma
    - Mocking services with TestBed
    - E2E Testing with Cypress or Protractor
6. Monorepos & Large-Scale Angular Applications
    - Nx Workspaces
    - Micro Frontends with Angular
7. Security & Best Practices
    - XSS & CSRF Protection
    - JWT Authentication (Interceptors)
    - CORS Handling
8. Real-world Problems (Behavioral & Architectural Questions)
    - How to handle large forms dynamically?
    - How to optimize an Angular app for performance?
    - How to debug memory leaks in Angular?



**Angular Interview Notes (Detailed and Structured)**

---

### 1. Component-Based Architecture

- Angular breaks the app into reusable units called **components**.
- Each component has a `@Component` decorator that defines:
 - `selector`: custom HTML tag
 - `template`: UI layout
 - `styles`: CSS
 - `class`: TypeScript logic

**Common Decorators:**

- `@NgModule`: Defines a module
- `@Injectable`: Marks a service for DI
- `@Input` / `@Output`: For parent-child communication

**Lifecycle Hooks:**

- `ngOnInit()`: Init logic, API calls
- `ngOnDestroy()`: Cleanup (unsubscribe)
- Others: `ngOnChanges()`, `ngDoCheck()`, `ngAfterViewInit()` etc.

---

### 2. Change Detection

- Angular checks and updates DOM when data changes.
- Triggered by:
 - Events, async tasks, or input property changes.

**Strategies:**

- `Default`: Checks entire component tree
- `OnPush`: Only checks on input change or internal event

**Manual Triggers:**

- `ChangeDetectorRef.detectChanges()`
- `ChangeDetectorRef.markForCheck()` (preferred with OnPush)

**Best Practice:**

- Use `async` pipe: handles subscription and triggers CD automatically

---

### 3. DOM Updates (Angular vs React)

- React: Uses Virtual DOM
- Angular: Uses Incremental DOM (Ivy)

**How Angular Updates the DOM:**
1. **Change Detection Phase**: Angular detects when component state changes.
2. **Template Evaluation**: Angular re-evaluates the component's template to identify any changes in bindings (e.g., `{{ value }}`).
3. **Instruction Execution**: Angular executes efficient instructions (e.g., `??textInterpolate1`) to directly update only the parts of the DOM that changed.

**Key Benefits:**
- Angular doesn't re-render the entire virtual tree. Instead, it surgically updates DOM elements.
- It maps expressions in the template to exact DOM nodes and reuses them efficiently.

**Ivy Rendering Engine:**
- Templates compile to a set of instructions during build time.
- Uses low-level operations to efficiently patch the DOM.
- Example internal instruction: `??property("value", ctx.inputValue)`

**Why Angular Doesn't Need a Virtual DOM:**
- Angular knows exactly what part of the DOM each expression updates.
- Updates are deterministic and fine-grained.

**Conclusion:**
- Angular provides better performance in many real-world cases due to precise DOM updates compared to React's diffing strategy.

---

### 4. Modules & Dependency Injection

**NgModules:**

- Use `@NgModule` to declare:
 - `declarations`: Components, directives
 - `imports`: Other modules
 - `providers`: Services for DI
 - `bootstrap`: Root component

**DI Basics:**

- Use `@Injectable({ providedIn: 'root' })` for singleton service
- Inject using constructor
- Angular has hierarchical DI (app ? module ? component)

**Interview Tips:**

- `providedIn: 'root'` vs local providers
- Lazy module injectors = isolated instances

---

### 5. Lazy Loading Modules

- Defers loading feature modules until needed
- Reduces initial bundle size

**Steps:**

1. Create feature module with routing
2. In `AppRoutingModule`:

```ts
{ path: 'feature', loadChildren: () => import('./feature.module').then(m => m.FeatureModule) }
```

3. Feature routing: define default route inside module

**Benefits:** Faster load, better UX

---

### 6. RxJS State Management

**Core Concepts:**

- `Observable`: Emits data over time
- `Subject`: Emits to multiple subscribers
- `BehaviorSubject`: Emits current + future values
- `ReplaySubject`: Re-emits last `n` values
- `AsyncSubject`: Emits only last value on `complete()`

**Operators:**

- Creation: `of`, `from`, `interval`
- Transformation: `map`, `switchMap`, `concatMap`
- Filtering: `filter`, `debounceTime`
- Error Handling: `catchError`, `retry`

**State in Services:**

- Use `BehaviorSubject` to store state
- Components subscribe to `state$`

---

### 7. NgRx State Management

**Why NgRx:**

- Scalable state management using Redux principles
- Predictable state transitions and debugging

**Core Concepts:**

- `Store`: Holds state
- `Actions`: Describe events
- `Reducers`: Pure functions that return new state
- `Effects`: Handle async operations
- `Selectors`: Extract specific slices of state

**Flow:**

1. Dispatch `loadTodos` ?
2. `Effect` makes API call ?
3. On success, dispatch `loadTodosSuccess` ?
4. `Reducer` updates state ?
5. `Selector` gets data for component

**Advantages vs RxJS-only:**

- Centralization, tooling, predictability
- But more boilerplate

---

### 8. Immutability

- Key for predictability, efficient CD
- Use spread operator (`...`) or libraries like Immer

**Benefits:**

- Enables Angular to detect changes via object reference
- Supports undo/redo, time-travel debugging

---

### 9. Error Handling in NgRx

**Steps:**

1. Define error actions: `loadItemsFailure({ error })`
2. Use `catchError()` in effects
3. Handle errors in reducers
4. Create selectors for error state
5. Display in components via `error$ | async`

**Best Practices:**

- Centralize logic, log errors, user-friendly messages

---

### 10. Logging & Debugging

**Logging:**

- Create `LoggingService`
- Use levels: `logInfo`, `logWarn`, `logError`
- Use libraries like `ngx-logger`

**Debugging Tools:**

- Angular DevTools: inspect components, CD cycles
- Augury: visualize component tree, routes
- Chrome DevTools + console

---

### 11. Interview-Ready NgRx Questions

**Conceptual:**

1. How does NgRx enforce a single source of truth?
2. What's the benefit of using Actions and Reducers?
3. How does NgRx differ from RxJS-only state?
4. Explain a complete flow: dispatch ? effect ? reducer ? selector ? view

**Performance:**
5. How to structure large apps in NgRx?
6. What is `OnPush` + NgRx interaction?

**Advanced:**
7. What is optimistic update?
8. How do `StoreModule.forRoot` vs `forFeature` differ?
9. EntityState usage and benefits?
10. Best practices for testing effects/selectors/reducers?

---

Let me know if you'd like:

- Flashcards
- One-pager cheat sheet
- Printable format
- Q&A style walkthrough
- Mind map or visual breakdown



-------------------------------------
Angular Interview Notes

Directives in Angular
Types of Directives:
- Component Directives: Have a template (e.g., AppComponent)
- Structural Directives: Change DOM structure (*ngIf, *ngFor)
- Attribute Directives: Change appearance or behavior (ngClass, custom highlight directive)

TemplateRef and ViewContainerRef
- TemplateRef: Represents an <ng-template>. Used to capture embedded views.
- ViewContainerRef: Acts as a container to add/remove views dynamically.
Example:
viewContainerRef.createEmbeddedView(templateRef);

Directive Lifecycle
Lifecycle Hooks:
- ngOnInit(), ngOnChanges(), ngDoCheck(), ngAfterViewInit(), ngOnDestroy

Manual Change Detection in ngDoCheck()
- Use for custom change logic (not to call detectChanges directly inside).
- If needed, use ChangeDetectorRef.detectChanges() outside of ngDoCheck.

Angular Forms

What are Forms in Angular?
Used for input handling and validation.
Types:
- Template-driven forms (ngModel)
- Reactive forms (FormGroup, FormControl, FormArray)

Why Reactive Forms?
- Dynamic form creation, scalable validation, easier testing, clean separation

Add/Remove Form Controls Dynamically
form.addControl('phone', new FormControl(''));
form.removeControl('phone');
skills.push(new FormControl('Skill'));
skills.removeAt(index);

Enable/Disable Controls
form.get('email')?.disable();
form.get('email')?.enable();
Use form.getRawValue() to get disabled fields.

Integrate Forms with Services (No NgRx)
ngOnInit() {
 userService.getUser(id).subscribe(user => form.patchValue(user));
}
onSubmit() {
 userService.saveUser(form.value).subscribe();
}

Forms with NgRx & Async Pipe (Student Form Example)
Fields: name, class, section, department

Store Setup
interface StudentState {
 student: Student | null;
 loading: boolean;
 error: any;
}

Actions
loadStudent, loadStudentSuccess, saveStudent, saveStudentSuccess, saveStudentFailure

Reducer
Handles state update, student data, error handling

Effects
loadStudent$ = createEffect(() => this.actions$.pipe(
 ofType(loadStudent),
 mergeMap(action => this.service.getStudent(action.id).pipe(
   map(student => loadStudentSuccess({ student })),
   catchError(error => of(saveStudentFailure({ error })))
 )));

Service
getStudent(id): Observable<Student>
saveStudent(student): Observable<Student>

Component
this.form = this.fb.group({...});
this.store.dispatch(loadStudent({ id: 1 }));
this.student$.subscribe(student => form.patchValue(student));
onSubmit() => dispatch(saveStudent({ student: form.value }))

Template (with async pipe)
<form [formGroup]="form" (ngSubmit)="onSubmit()">
 <div *ngIf="loading$ | async">Loading...</div>
 <input formControlName="name" />
 <input formControlName="class" />
 <input formControlName="section" />
 <input formControlName="department" />
 <button type="submit">Save</button>
</form>

Interview Tips
- patchValue for partial updates
- async pipe to auto-unsubscribe
- FormBuilder simplifies setup
- Explain reactive validation + NgRx separation of concerns


