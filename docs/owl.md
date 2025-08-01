# OWL Component Implementation Guide for Odoo 17

This document outlines key findings and best practices for implementing OWL components in Odoo 17, with a specific focus on field widgets and options handling.

## Field Widget Implementation

### Component Structure

```javascript
export class MyFieldWidget extends Component {
    static template = "module_name.MyFieldWidget";
    static components = { /* Sub-components */ };

    // Define props with proper types
    static props = {
        ...standardFieldProps,
        // Add specific props for options
        customOption1: { type: String, optional: true },
        customOption2: { type: Boolean, optional: true },
        // For props that might be undefined, avoid specifying a type
        customOption3: { optional: true },
    };

    // Define default values for props
    static defaultProps = {
        customOption1: 'default',
        customOption2: false,
    };

    setup() {
        // Component initialization logic
    }

    // Component methods
}

// Register the widget in the fields registry
registry.category("fields").add("my_field_widget", {
    component: MyFieldWidget,
    supportedTypes: ["char", "text"], // Field types this widget supports

    // Extract options from the view and convert to props
    extractProps: ({ attrs, options }) => {
        // Create the result object with basic options
        const result = {
            customOption1: options.customOption1 || 'default',
            customOption2: options.customOption2 || false,
        };

        // Only add props if they're explicitly defined
        if (options.customOption3 !== undefined) {
            result.customOption3 = options.customOption3;
        }

        return result;
    },
});
```

### Template Structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
    <t t-name="module_name.MyFieldWidget">
        <!-- Access props directly -->
        <div t-att-class="props.customOption2 ? 'special-class' : ''">
            <span t-esc="props.customOption1"/>
            <!-- Other template content -->
        </div>
    </t>
</templates>
```

## Key Findings

### 1. Options Handling in Odoo 17

- **Options Access Pattern**:
  - Options are passed directly to the `extractProps` function as a parameter
  - The function signature should be `extractProps: ({ attrs, options }) => { ... }`
  - This pattern is consistent with Odoo 17's approach to component options

- **Empty attrs Object**:
  - In Odoo 17, the `attrs` object is consistently empty (`{}`) due to the deprecation of attrs in view definitions
  - Odoo 17 has moved away from using the `attrs` attribute in view definitions
  - The `attrs` parameter is kept for backward compatibility or internal framework use

- **Options Format**:
  - Options are passed as a JavaScript object, not as a string that needs parsing
  - No need to use `JSON.parse` or replace single quotes with double quotes

### 2. Props Structure

- **Individual Props vs. Options Object**:
  - Instead of having a single `options` prop, each option should be defined as a separate prop
  - This makes the component more explicit about what options it accepts
  - It also allows for better type checking and default values

- **Default Values**:
  - Use `static defaultProps` to define default values for options
  - This ensures that the component works even if options aren't provided

- **Type Validation**:
  - OWL's type validation is strict - props must match their defined types
  - For optional props that might be undefined, avoid specifying a type or use `{ optional: true }`
  - Only add props to the result if they have valid values

### 3. extractProps Function

- **Purpose**:
  - The `extractProps` function extracts options from the view and converts them to props
  - It's called when the field is being rendered

- **Implementation**:
  - Extract each option individually and return them as separate props
  - Provide default values for options that aren't specified
  - Only include props in the result if they're explicitly defined
  - This prevents passing undefined or null values that might cause validation errors

### 4. Template Binding

- **Direct Props Access**:
  - In the template, bind directly to `props.optionName` instead of using intermediate properties
  - This ensures that the component reacts properly to prop changes

## Common Pitfalls

1. **Type Validation Errors**:
   - Error: `Invalid props for component 'MyComponent': 'propName' is not a string`
   - Solution: Make sure the prop type matches the actual value, or make the prop truly optional

2. **Undefined Options**:
   - Issue: Options specified in the XML view aren't being passed to the component
   - Solution: Check that the `extractProps` function is correctly extracting options

3. **Props Not Updating**:
   - Issue: Component doesn't react to prop changes
   - Solution: Make sure you're binding directly to props in the template

4. **Missing Default Values**:
   - Issue: Component breaks when options aren't specified
   - Solution: Provide default values using `static defaultProps` or in the `extractProps` function

5. **State Synchronization Issues**:
   - Issue: Widget state doesn't sync when field value changes externally
   - Solution: Use `onWillUpdateProps` to keep internal state synchronized with external field changes
   - Example: Form resets, programmatic field updates, or other widgets modifying the same field

6. **Field Clearing After Save Operations**:
   - Issue: Dependent fields get cleared after saving the record, even when the related field hasn't changed
   - Solution: Distinguish between actual field changes and save operations by storing the old value and comparing
   - Use conditional logic to only clear fields when the related field actually changes

7. **Race Conditions in useEffect**:
   - Issue: Async operations in useEffect can cause race conditions or unexpected behavior
   - Solution: Properly handle async operations with `.then()` and ensure proper return values
   - Always return `undefined` explicitly from useEffect when not using cleanup functions

8. **Missing onWillUpdateProps**:
   - Issue: Widget appears to work initially but doesn't respond to external field changes
   - Solution: Implement `onWillUpdateProps` alongside `useEffect` for complete state management
   - This is especially important for widgets that maintain internal state

## XML View Usage

```xml
<!-- Basic usage -->
<field name="field_name" widget="my_field_widget"/>

<!-- With options -->
<field name="field_name" widget="my_field_widget"
       options="{'customOption1': 'value', 'customOption2': true}"/>
```

## Debugging Tips

1. **Console Logging**:
   - Log the props object in the setup method: `console.log("Props:", this.props);`
   - Log the options in the extractProps function: `console.log("Options:", options);`

2. **Component Lifecycle**:
   - Use `onWillStart`, `onMounted`, etc. to debug component lifecycle issues

3. **Props Validation**:
   - Check the browser console for props validation errors
   - These errors indicate that a prop doesn't match its defined type

## State Synchronization and Field Monitoring

### Using onWillUpdateProps for External State Changes

When building field widgets that maintain internal state, it's crucial to keep the widget's state synchronized with external changes to the record. The `onWillUpdateProps` hook is essential for this purpose.

```javascript
setup() {
    this.state = useState({
        fieldPath: this.props.record.data[this.props.name] || "",
        // other state properties
    });

    // Watch for changes to the record data to keep state in sync
    onWillUpdateProps((nextProps) => {
        const newValue = nextProps.record.data[nextProps.name];
        if (newValue !== this.state.fieldPath) {
            this.state.fieldPath = newValue || "";
        }
    });
}
```

**Key Points:**
- `onWillUpdateProps` ensures the widget's internal state stays synchronized with external field value changes
- This is particularly important when the field value can be modified by other parts of the application
- Always check if the value has actually changed before updating state to avoid unnecessary re-renders

### Using useEffect to Monitor Record Fields

The `useEffect` hook is a powerful tool for reacting to changes in component state or props. In Odoo 18, it can be used to monitor changes in record fields and trigger actions accordingly.

#### Basic Implementation Pattern

```javascript
useEffect(
    // First argument: the effect function
    () => {
        // Code to run when dependencies change
        this._someFunction();

        // Return value must be a function or undefined
        return undefined; // Explicitly return undefined for cleanup
    },
    // Second argument: the dependency array - returns the values to watch
    () => [this.props.record.data[this.props.fieldName]]
);
```

### Key Points

1. **Dependency Function**:
   - Unlike React's useEffect which takes an array directly, OWL's useEffect takes a function that returns an array
   - This function is called to determine the current values of dependencies
   - Format: `() => [dependency1, dependency2, ...]`

2. **Monitoring Record Fields**:
   - To watch a field in the record: `() => [this.props.record.data[this.props.fieldName]]`
   - The effect will run whenever the value of that field changes

3. **Return Value Options**:
   - The effect function can return a cleanup function, undefined, or nothing
   - There are four common patterns for return values:
     1. **Return a cleanup function** when resources need to be cleaned up:
        ```javascript
        return () => {
            // Cleanup code (remove event listeners, clear timers, etc.)
        };
        ```
     2. **Return undefined explicitly** for maximum safety:
        ```javascript
        return undefined;
        ```
     3. **Early return** for conditional execution:
        ```javascript
        if (!someCondition) {
            return; // Implicitly returns undefined
        }
        // Rest of effect code...
        ```
     4. **No return statement** (implicitly returns undefined):
        ```javascript
        // Effect code with no return statement
        ```
   - While options 2, 3, and 4 are functionally equivalent, explicit returns (option 2) can help prevent certain runtime errors in some OWL versions

4. **Common Use Cases**:
   - Fetching related data when a field changes
   - Updating component state based on record field changes
   - Triggering validations or calculations

### Advanced Pattern: Handling Model Changes with Field Clearing

When building widgets that depend on related models (like a field selector that depends on a model_id), you need to handle cases where the model changes and previously selected fields become invalid.

```javascript
// Use useEffect to monitor model field changes and fetch model name
useEffect(
    () => {
        const oldModel = this.state.resModel;
        this._fetchModelName().then(() => {
            // Clear field path if model actually changed (not just initial load)
            // This prevents invalid field paths when switching models
            if (oldModel && oldModel !== this.state.resModel && this.state.fieldPath) {
                this.state.fieldPath = "";
                this.props.record.update({ [this.props.name]: "" });
            }
        });

        return undefined;
    },
    () => [this.props.record.data[this.props.modelField]]
);

// Function triggered when model_id changes
async _fetchModelName() {
    const modelIdValue = this.props.record.data[this.props.modelField];
    if (!modelIdValue) {
        this.state.resModel = "";
        return;
    }

    const modelId = Array.isArray(modelIdValue) ? modelIdValue[0] : modelIdValue;

    try {
        const result = await this.orm.call(
            'ir.model',
            'search_read',
            [[['id', '=', modelId]]],
            { fields: ['model'] }
        );

        this.state.resModel = result.length ? result[0].model : "";
    } catch (error) {
        console.error("Failed to fetch model name:", error);
        this.state.resModel = "";
    }
}
```

**Key Implementation Details:**

1. **Detecting Actual Model Changes**:
   - Store the old model value before fetching the new one
   - Only clear dependent fields if the model actually changed (not during initial load)
   - This prevents clearing fields after save operations

2. **Conditional Field Clearing**:
   - Check if `oldModel` exists (not initial load)
   - Check if the model actually changed (`oldModel !== this.state.resModel`)
   - Check if there's a field path to clear (`this.state.fieldPath`)

3. **Async Handling**:
   - Use `.then()` to handle the async operation properly
   - Ensure field clearing happens after the model name is fetched

This pattern is particularly useful for widgets that need to react to changes in related fields while avoiding unwanted field clearing during save operations.

### Combining onWillUpdateProps and useEffect

For complex widgets that need both external state synchronization and reactive field monitoring, you should use both hooks together:

```javascript
setup() {
    this.state = useState({
        fieldPath: this.props.record.data[this.props.name] || "",
        resModel: "",
    });

    // Sync with external field changes
    onWillUpdateProps((nextProps) => {
        const newValue = nextProps.record.data[nextProps.name];
        if (newValue !== this.state.fieldPath) {
            this.state.fieldPath = newValue || "";
        }
    });

    // React to model field changes
    useEffect(
        () => {
            const oldModel = this.state.resModel;
            this._fetchModelName().then(() => {
                if (oldModel && oldModel !== this.state.resModel && this.state.fieldPath) {
                    this.state.fieldPath = "";
                    this.props.record.update({ [this.props.name]: "" });
                }
            });

            return undefined;
        },
        () => [this.props.record.data[this.props.modelField]]
    );
}
```

**Why Both Are Needed:**
- `onWillUpdateProps`: Handles external changes to the field value (e.g., programmatic updates, form resets)
- `useEffect`: Handles reactive changes to related fields (e.g., model_id changes affecting field selection)

### Potential Issues with Async Functions and useEffect

When calling async functions within useEffect, special care must be taken with return values:

```javascript
useEffect(
    () => {
        this._fetchModelName(); // This is an async function
        return undefined; // Explicitly return undefined for cleanup
    },
    () => [this.props.record.data[this.props.modelField]]
);

async _fetchModelName() {
    // ...
    if (!modelIdValue) {
        this.state.resModel = "";
        return; // Early return in async function
    }
    // Rest of async function...
}
```

**Important considerations:**

1. **Async functions and return values**:
   - Async functions always return a Promise, not undefined, even when using `return;`
   - This can cause issues if the async function is directly returned from useEffect

2. **Early returns in async functions**:
   - Using `return;` in an async function called by useEffect is fine
   - The issue occurs only if the async function itself is returned from useEffect

3. **Best practices**:
   - Call async functions within useEffect, but don't return them
   - Always explicitly return undefined or a cleanup function from useEffect
   - Handle early returns within the async function as needed

## Conclusion

Implementing field widgets in Odoo 17 requires understanding multiple aspects:

1. **Options Handling**: How options are passed from the view to the component through the `extractProps` function
2. **State Management**: Proper synchronization between internal widget state and external field changes
3. **Reactive Programming**: Using `useEffect` to monitor related field changes and trigger appropriate actions
4. **Lifecycle Management**: Combining `onWillUpdateProps` and `useEffect` for comprehensive state management

By following the patterns outlined in this document, you can create robust and reusable field widgets that:
- Properly handle options and provide a consistent user experience
- Maintain state synchronization with external changes
- React appropriately to related field changes without unwanted side effects
- Avoid common pitfalls like race conditions and improper field clearing

**Key Takeaways:**
- Always use `onWillUpdateProps` for widgets that maintain internal state
- Use `useEffect` for reactive programming and monitoring related fields
- Be careful with async operations and always return `undefined` from useEffect when not using cleanup
- Distinguish between actual field changes and save operations to prevent unwanted field clearing
- Remember that Odoo 17 has moved away from using attrs in views, and options are now passed directly to the extractProps function

This comprehensive approach ensures that your field widgets work reliably across different scenarios and provide a smooth user experience.
