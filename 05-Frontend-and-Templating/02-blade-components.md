### Detailed Conceptual Overview (70%)

Blade components represent a paradigm shift in Laravel's templating system. They provide a robust mechanism for building reusable, encapsulated UI elements with their own templates, logic, and styles. Understanding components deeply transforms how you architect your application's frontend.

**The Component Architecture**

Blade components consist of two parts: a Blade template file (the view) and a corresponding PHP class (the component logic). The class handles data preparation, dependency injection, and any processing logic. The template receives the processed data and renders HTML. This separation mirrors the Controller-View pattern at a micro level.

Components can be class-based or anonymous. Class-based components have an associated PHP class in `app/View/Components` that provides explicit data handling and constructor dependency injection. Anonymous components are simple Blade files in `resources/views/components` without a class—ideal for stateless presentational elements.

**Component Data Flow and Attributes**

Data flows into components through two mechanisms: the constructor (for class-based components) and the `@props` directive (for anonymous components). The constructor receives data as parameters and stores it as public properties accessible in the template. Anonymous components declare expected props with defaults and types.

HTML attributes passed to components that aren't matched to constructor parameters or props are captured in the `$attributes` object. This enables attribute forwarding—passing through CSS classes, data attributes, ARIA attributes, and Alpine.js directives to the root element of the component template. The `$attributes->merge()` method intelligently combines default and passed classes.

**Slots: Composable Content Areas**

Slots enable components to receive and render content from parent views. The `$slot` variable contains the default slot—any content passed between the component's opening and closing tags. Named slots provide multiple content areas, allowing complex components like cards, modals, and panels to accept different content sections.

The `@slot` directive syntax is the older approach; the new `<x-slot:name>` tag syntax is preferred for clarity. Named slots can have default content in the component template using the `$name ?? 'Default Content'` pattern. Scoped slots pass data from the component back to the parent's slot content.

**Anonymous Index Components**

Laravel automatically resolves components using a dot-notation convention. A `<x-ui.button>` tag resolves to `resources/views/components/ui/button.blade.php`. This convention eliminates explicit component registration and makes the filesystem the single source of truth for component organization.

**Component Attributes and HTML Handling**

The `$attributes` object provides methods for sophisticated attribute manipulation. `->merge(['class' => 'default'])` combines component defaults with passed classes, resolving conflicts intelligently using Tailwind CSS conflict resolution. `->class()` adds classes conditionally. `->filter()` selects specific attribute types. `->except()` removes attributes before forwarding.

**Dynamic Components**

The `<x-dynamic-component>` tag renders a component whose type is determined at runtime. This enables rendering different components based on data: a different form field component for each field type, or different chart components based on chart configuration. The component name comes from a variable or expression.

### Production-Ready Code Snippets (30%)

**1. Class-Based Card Component**
```php
<?php
// app/View/Components/Card.php
namespace App\View\Components;

use Illuminate\View\Component;
use Illuminate\Contracts\View\View;

class Card extends Component
{
    /**
     * Create the component instance.
     */
    public function __construct(
        public ?string $header = null,
        public ?string $footer = null,
        public bool $padding = true,
        public bool $shadow = true,
        public bool $border = false,
        public string $headerClass = '',
    ) {}

    /**
     * Get the view / contents that represent the component.
     */
    public function render(): View
    {
        return view('components.card');
    }

    /**
     * Get the computed CSS classes for the card.
     */
    public function cardClasses(): string
    {
        return implode(' ', array_filter([
            'bg-white rounded-lg',
            $this->shadow ? 'shadow-md' : '',
            $this->border ? 'border border-gray-200' : '',
            $this->padding ? 'p-6' : '',
        ]));
    }

    /**
     * Determine if the card has a header.
     */
    public function hasHeader(): bool
    {
        return $this->header !== null;
    }
}
```

```blade
{{-- resources/views/components/card.blade.php --}}
<div {{ $attributes->merge(['class' => $cardClasses()]) }}>
    {{-- Header section --}}
    @if ($hasHeader())
        <div class="flex items-center justify-between mb-4 pb-4 border-b border-gray-200 
                    {{ $headerClass }}">
            <h3 class="text-lg font-medium text-gray-900">{{ $header }}</h3>
            
            {{-- Actions slot for header buttons --}}
            @isset($actions)
                <div class="flex items-center gap-2">
                    {{ $actions }}
                </div>
            @endisset
        </div>
    @endif
    
    {{-- Default content slot --}}
    <div class="{{ $padding && $hasHeader() ? '' : 'py-4' }}">
        {{ $slot }}
    </div>
    
    {{-- Footer section --}}
    @isset($footer)
        <div class="mt-4 pt-4 border-t border-gray-200">
            {{ $footer }}
        </div>
    @else
        @isset($footerSlot)
            <div class="mt-4 pt-4 border-t border-gray-200">
                {{ $footerSlot }}
            </div>
        @endisset
    @endisset
</div>
```

**2. Anonymous Form Components with Error Integration**
```blade
{{-- resources/views/components/form/select.blade.php --}}
@props([
    'name',
    'label' => null,
    'options' => [],
    'selected' => null,
    'placeholder' => 'Select an option',
    'required' => false,
    'multiple' => false,
])

<div class="mb-4">
    @if ($label)
        <label for="{{ $name }}" class="block text-sm font-medium text-gray-700 mb-1">
            {{ $label }}
            @if ($required)
                <span class="text-red-500">*</span>
            @endif
        </label>
    @endif
    
    <select name="{{ $name }}" 
            id="{{ $name }}"
            @required($required)
            @if ($multiple) multiple @endif
            {{ $attributes->class([
                'block w-full rounded-md shadow-sm',
                'border-gray-300 focus:border-indigo-500 focus:ring-indigo-500' => !$errors->has($name),
                'border-red-300 focus:border-red-500 focus:ring-red-500' => $errors->has($name),
            ]) }}>
        
        @if (!$multiple && $placeholder)
            <option value="">{{ $placeholder }}</option>
        @endif
        
        @foreach ($options as $value => $label)
            @php
                $isSelected = $multiple 
                    ? in_array($value, old($name, $selected ?? []))
                    : old($name, $selected) == $value;
            @endphp
            <option value="{{ $value }}" @selected($isSelected)>
                {{ $label }}
            </option>
        @endforeach
    </select>
    
    @error($name)
        <p class="mt-1 text-sm text-red-600">{{ $message }}</p>
    @enderror
</div>

{{-- resources/views/components/form/button.blade.php --}}
@props([
    'type' => 'submit',
    'variant' => 'primary',
    'size' => 'md',
    'loading' => false,
])

@php
    $baseClasses = 'inline-flex items-center justify-center font-medium rounded-md focus:outline-none focus:ring-2 focus:ring-offset-2 transition duration-150 ease-in-out disabled:opacity-50 disabled:cursor-not-allowed';
    
    $variants = [
        'primary' => 'bg-indigo-600 text-white hover:bg-indigo-700 focus:ring-indigo-500',
        'secondary' => 'bg-white text-gray-700 border border-gray-300 hover:bg-gray-50 focus:ring-indigo-500',
        'danger' => 'bg-red-600 text-white hover:bg-red-700 focus:ring-red-500',
        'success' => 'bg-green-600 text-white hover:bg-green-700 focus:ring-green-500',
        'ghost' => 'bg-transparent text-gray-600 hover:bg-gray-100 focus:ring-gray-500',
    ];
    
    $sizes = [
        'xs' => 'px-2.5 py-1.5 text-xs',
        'sm' => 'px-3 py-2 text-sm leading-4',
        'md' => 'px-4 py-2 text-sm',
        'lg' => 'px-4 py-2 text-base',
        'xl' => 'px-6 py-3 text-base',
    ];
    
    $classes = $baseClasses . ' ' . $variants[$variant] . ' ' . $sizes[$size];
@endphp

<button type="{{ $type }}" 
        {{ $attributes->merge(['class' => $classes]) }}
        @if ($loading) disabled @endif>
    
    @if ($loading)
        <svg class="animate-spin -ml-1 mr-2 h-4 w-4" 
             xmlns="http://www.w3.org/2000/svg" 
             fill="none" 
             viewBox="0 0 24 24">
            <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
            <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
        </svg>
    @endif
    
    {{ $slot }}
</button>
```

**3. Data Table Component with Named Slots and Scoped Data**
```php
<?php
// app/View/Components/DataTable.php
namespace App\View\Components;

use Illuminate\View\Component;
use Illuminate\Contracts\Pagination\LengthAwarePaginator;
use Illuminate\Support\Collection;

class DataTable extends Component
{
    public function __construct(
        public array|Collection $rows,
        public array $columns = [],
        public bool $searchable = true,
        public bool $paginated = false,
        public string $emptyMessage = 'No records found.',
    ) {
        $this->rows = $rows instanceof Collection ? $rows : collect($rows);
    }

    public function render()
    {
        return view('components.data-table');
    }

    public function hasPaginator(): bool
    {
        return $this->paginated && $this->rows instanceof LengthAwarePaginator;
    }
}
```

```blade
{{-- resources/views/components/data-table.blade.php --}}
<div {{ $attributes->merge(['class' => 'overflow-hidden']) }}>
    {{-- Search and actions header --}}
    @if ($searchable || isset($actions))
        <div class="flex items-center justify-between mb-4">
            @if ($searchable)
                <div class="relative flex-1 max-w-xs">
                    <input type="search" 
                           placeholder="Search..." 
                           class="block w-full rounded-md border-gray-300 pl-10"
                           wire:model.live.debounce.300ms="search">
                    <svg class="absolute left-3 top-2.5 h-5 w-5 text-gray-400" viewBox="0 0 20 20" fill="currentColor">
                        <path fill-rule="evenodd" d="M8 4a4 4 0 100 8 4 4 0 000-8zM2 8a6 6 0 1110.89 3.476l4.817 4.817a1 1 0 01-1.414 1.414l-4.816-4.816A6 6 0 012 8z" clip-rule="evenodd" />
                    </svg>
                </div>
            @endif
            
            @isset($actions)
                <div class="flex items-center gap-2">
                    {{ $actions }}
                </div>
            @endisset
        </div>
    @endif
    
    {{-- Table --}}
    <div class="overflow-x-auto">
        <table class="min-w-full divide-y divide-gray-200">
            <thead class="bg-gray-50">
                <tr>
                    @foreach ($columns as $column)
                        <th scope="col" 
                            class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                            {{ $column }}
                        </th>
                    @endforeach
                    
                    {{-- Actions column --}}
                    @if (isset($actionsColumn))
                        <th scope="col" class="relative px-6 py-3">
                            <span class="sr-only">Actions</span>
                        </th>
                    @endif
                </tr>
            </thead>
            <tbody class="bg-white divide-y divide-gray-200">
                @forelse ($rows as $row)
                    <tr class="hover:bg-gray-50 transition-colors">
                        {{-- Render row data using scoped slot --}}
                        @if (isset($rowContent))
                            {{ $rowContent($row) }}
                        @else
                            @foreach ($columns as $key => $label)
                                <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-900">
                                    {{ data_get($row, $key) }}
                                </td>
                            @endforeach
                        @endif
                        
                        {{-- Actions for each row --}}
                        @if (isset($actionsColumn))
                            <td class="px-6 py-4 whitespace-nowrap text-right text-sm font-medium">
                                {{ $actionsColumn($row) }}
                            </td>
                        @endif
                    </tr>
                @empty
                    <tr>
                        <td colspan="{{ count($columns) + (isset($actionsColumn) ? 1 : 0) }}" 
                            class="px-6 py-12 text-center text-gray-500">
                            {{ $emptyMessage }}
                            @isset($empty)
                                <div class="mt-2">
                                    {{ $empty }}
                                </div>
                            @endisset
                        </td>
                    </tr>
                @endforelse
            </tbody>
        </table>
    </div>
    
    {{-- Pagination --}}
    @if ($hasPaginator())
        <div class="mt-4">
            {{ $rows->links() }}
        </div>
    @endif
</div>
```

**4. Anonymous Modal Component with Alpine.js**
```blade
{{-- resources/views/components/modal.blade.php --}}
@props([
    'name' => 'modal',
    'maxWidth' => '2xl',
    'closeable' => true,
])

@php
    $maxWidthClasses = match($maxWidth) {
        'sm' => 'sm:max-w-sm',
        'md' => 'sm:max-w-md',
        'lg' => 'sm:max-w-lg',
        'xl' => 'sm:max-w-xl',
        '2xl' => 'sm:max-w-2xl',
        '3xl' => 'sm:max-w-3xl',
        '4xl' => 'sm:max-w-4xl',
        'full' => 'sm:max-w-full sm:mx-4',
        default => 'sm:max-w-lg',
    };
@endphp

<div x-data="{ 
        show: @entangle($attributes->wire('model')), 
        focusables() {
            let selector = 'a, button, input:not([type=\\'hidden\\']), textarea, select, details, [tabindex]:not([tabindex=\\'-1\\'])'
            return [...$el.querySelectorAll(selector)].filter(el => !el.hasAttribute('disabled'))
        },
        firstFocusable() { return this.focusables()[0] },
        lastFocusable() { return this.focusables().slice(-1)[0] },
        nextFocusable() { return this.focusables()[this.nextFocusableIndex()] || this.firstFocusable() },
        prevFocusable() { return this.focusables()[this.prevFocusableIndex()] || this.lastFocusable() },
        nextFocusableIndex() { return (this.focusables().indexOf(document.activeElement) + 1) % (this.focusables().length + 1) },
        prevFocusableIndex() { return Math.max(0, this.focusables().indexOf(document.activeElement)) - 1 },
    }" 
    x-init="$watch('show', value => {
        if (value) {
            document.body.classList.add('overflow-y-hidden');
            {{ $attributes->has('focusable') ? 'setTimeout(() => firstFocusable().focus(), 100)' : '' }}
        } else {
            document.body.classList.remove('overflow-y-hidden');
        }
    })"
    x-on:close.stop="show = false"
    x-on:keydown.escape.window="show = false"
    x-on:keydown.tab.prevent="$event.shiftKey || nextFocusable().focus()"
    x-on:keydown.shift.tab.prevent="prevFocusable().focus()"
    x-show="show" 
    class="fixed inset-0 overflow-y-auto px-4 py-6 sm:px-0 z-50" 
    style="display: none;">
    
    {{-- Backdrop --}}
    <div x-show="show" 
         class="fixed inset-0 transform transition-all" 
         x-on:click="show = false"
         x-transition:enter="ease-out duration-300"
         x-transition:enter-start="opacity-0"
         x-transition:enter-end="opacity-100"
         x-transition:leave="ease-in duration-200"
         x-transition:leave-start="opacity-100"
         x-transition:leave-end="opacity-0">
        <div class="absolute inset-0 bg-gray-500 opacity-75"></div>
    </div>
    
    {{-- Modal panel --}}
    <div x-show="show" 
         class="mb-6 bg-white rounded-lg overflow-hidden shadow-xl transform transition-all sm:w-full {{ $maxWidthClasses }} sm:mx-auto"
         x-transition:enter="ease-out duration-300"
         x-transition:enter-start="opacity-0 translate-y-4 sm:translate-y-0 sm:scale-95"
         x-transition:enter-end="opacity-100 translate-y-0 sm:scale-100"
         x-transition:leave="ease-in duration-200"
         x-transition:leave-start="opacity-100 translate-y-0 sm:scale-100"
         x-transition:leave-end="opacity-0 translate-y-4 sm:translate-y-0 sm:scale-95">
        
        {{-- Close button --}}
        @if ($closeable)
            <div class="absolute top-0 right-0 pt-4 pr-4">
                <button type="button" 
                        x-on:click="show = false"
                        class="text-gray-400 hover:text-gray-500 focus:outline-none">
                    <span class="sr-only">Close</span>
                    <svg class="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
                    </svg>
                </button>
            </div>
        @endif
        
        {{ $slot }}
    </div>
</div>
```

**Best Practices (Staff Engineer Tips)**

1. **Component Granularity**: Strike a balance between component reusability and complexity. A component handling three different display modes with fifteen props is too complex. Extract variants into separate components that share logic through a trait or base class. Follow the Single Responsibility Principle.

2. **Constructor Dependency Injection**: Class-based components support full dependency injection. Inject services directly into the constructor rather than using facades. This makes components testable in isolation and explicit about their dependencies.

3. **Attribute Forwarding Patterns**: Use `$attributes->merge()` on the root element for class merging with Tailwind. Use `$attributes->that()` to extract specific attributes for forwarding to a single element. Avoid spreading `$attributes` across multiple elements without filtering.

4. **Component Testing**: Test components using Laravel's `View::make()` or `Blade::render()` methods in unit tests. Verify rendered HTML contains expected content, classes, and attributes. Test edge cases: empty slots, missing props, null values. Components should degrade gracefully without errors.

**Common Pitfalls and Solutions**

1. **Component Name Conflicts**: Two packages providing `<x-modal>` components conflict silently—one overrides the other. Use vendor prefixes: `<x-app-modal>`, `<x-package-modal>`. Laravel's component resolution uses the last registered match.

2. **Props vs Constructor Parameters Mismatch**: In class-based components, constructor parameter names must match the attribute names in kebab-case. `$maxWidth` in the constructor corresponds to `max-width` in the Blade tag. Mismatched naming causes silent failures where data doesn't flow.

3. **Slot Scope Variable Shadowing**: Scoped slot variables can shadow parent scope variables. Use explicit variable names in scoped slots. Document which variables are provided to avoid confusion. Consider type-hinting slot parameters in component documentation.

4. **Heavy Computations in Component Render**: Components are instantiated on every page render. Complex database queries or API calls in component constructors degrade performance. Cache expensive operations. Use lazy loading patterns for data that might not be rendered.

---
