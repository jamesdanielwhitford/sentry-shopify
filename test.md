# Sentry Shopify Integration - Complete Implementation Guide

## Overview
This document details the complete Sentry integration for a Shopify theme, implementing 4 realistic ecommerce error scenarios to demonstrate session replay, performance monitoring, and error tracking capabilities.

## Sentry Configuration

### Initial Setup
**File:** `layout/theme.liquid`
**Location:** In the `<head>` section, after existing scripts

```html
<script
  src="https://js.sentry-cdn.com/e2ac19e8552de63877c49774a0d915f6.min.js"
  crossorigin="anonymous"
></script>

<script>
  Sentry.onLoad(function() {
    Sentry.init({
      // Set user context for ecommerce tracking
      initialScope: {
        tags: {
          store: "{{ shop.name }}",
          page_type: "{{ template.name }}"
        },
        user: {
          {% if customer %}
            id: "{{ customer.id }}",
            email: "{{ customer.email }}",
          {% endif %}
        },
        contexts: {
          page: {
            template: "{{ template.name }}",
            url: "{{ canonical_url }}"
          }
        }
      },
      
      // Performance & Session Replay
      tracesSampleRate: 1.0, // 100% for testing - lower in production
      replaysSessionSampleRate: 1.0, // 100% for testing
      replaysOnErrorSampleRate: 1.0, // Always capture on errors
      
      // Ecommerce-specific configuration
      beforeSend(event) {
        // Add cart info to all events
        if (typeof window !== 'undefined' && window.cart) {
          event.contexts = event.contexts || {};
          event.contexts.cart = {
            item_count: window.cart.item_count || 0,
            total_price: window.cart.total_price || 0,
            currency: "{{ cart.currency.iso_code }}"
          };
        }
        return event;
      }
    });
    
    // Track ecommerce events
    document.addEventListener('DOMContentLoaded', function() {
      Sentry.setTag('cart_items', {{ cart.item_count }});
      Sentry.setContext('store_info', {
        currency: "{{ cart.currency.iso_code }}",
        locale: "{{ request.locale.iso_code }}"
      });
    });
  });
</script>
```

---

## Scenario 1: Checkout Button Stays Greyed Out (Shipping API Timeout)

### File Modified
**File:** `sections/main-cart-footer.liquid`

### Implementation
**Find:** The existing checkout button:
```html
<button
  type="submit"
  id="checkout"
  class="cart__checkout-button button"
  name="checkout"
  {% if cart == empty %}
    disabled
  {% endif %}
  form="cart"
>
  {{ 'sections.cart.checkout' | t }}
</button>
```

**Replace with:**
```html
<button
  type="submit"
  id="checkout"
  class="cart__checkout-button button"
  name="checkout"
  {% if cart == empty %}
    disabled
  {% endif %}
  form="cart"
>
  <span id="checkout-button-text">{{ 'sections.cart.checkout' | t }}</span>
  <span id="checkout-spinner" class="loading__spinner" style="display: none;"></span>
</button>

<script>
document.addEventListener('DOMContentLoaded', function() {
  const checkoutBtn = document.getElementById('checkout');
  if (checkoutBtn) {
    checkoutBtn.addEventListener('click', function(e) {
      // Simulate the shipping rates API getting stuck
      e.preventDefault();
      
      const buttonText = document.getElementById('checkout-button-text');
      const spinner = document.getElementById('checkout-spinner');
      
      checkoutBtn.disabled = true;
      buttonText.style.display = 'none';
      spinner.style.display = 'inline-block';
      buttonText.textContent = 'Calculating shipping...';
      
      // This will hang for 12+ seconds (the problematic scenario)
      setTimeout(() => {
        // Just capture the error with Sentry (simpler approach)
        if (typeof Sentry !== 'undefined') {
          Sentry.captureException(new Error('Shipping rates API timeout after 12 seconds'));
        }
        
        // Show error state
        checkoutBtn.disabled = false;
        spinner.style.display = 'none';
        buttonText.style.display = 'inline-block';
        buttonText.textContent = 'Try Again - Shipping Error';
        checkoutBtn.style.backgroundColor = '#dc3545';
        
      }, 12000); // 12 second delay
    });
  }
});
</script>
```

### Expected Sentry Data
- **Error:** "Shipping rates API timeout after 12 seconds"
- **Session Replay:** Shows user clicking checkout, waiting 12 seconds, button turning red
- **User Context:** Cart items, customer info, page template
- **Performance:** Long task duration flagged

---

## Scenario 2: Search Autocomplete Gets Stuck (Slow Search API)

### File Modified
**File:** `assets/predictive-search.js`

### Implementation
**Find:** The `getSearchResults` method and **modify the fetch call section**:

```javascript
getSearchResults(searchTerm) {
  const queryKey = searchTerm.replace(' ', '-').toLowerCase();
  this.setLiveRegionLoadingState();

  if (this.cachedResults[queryKey]) {
    this.renderSearchResults(this.cachedResults[queryKey]);
    return;
  }

  // Add this at the beginning of the search function that makes API calls
  console.log('Search initiated for:', searchTerm);

  // Add artificial delay before the existing fetch call
  setTimeout(() => {
    fetch(`${routes.predictive_search_url}?q=${encodeURIComponent(searchTerm)}&section_id=predictive-search`, {
      signal: this.abortController.signal,
    })
      .then((response) => {
        if (!response.ok) {
          var error = new Error(response.status);
          this.close();
          throw error;
        }

        return response.text();
      })
      .then((text) => {
        const resultsMarkup = new DOMParser()
          .parseFromString(text, 'text/html')
          .querySelector('#shopify-section-predictive-search').innerHTML;
        // Save bandwidth keeping the cache in all instances synced
        this.allPredictiveSearchInstances.forEach((predictiveSearchInstance) => {
          predictiveSearchInstance.cachedResults[queryKey] = resultsMarkup;
        });
        this.renderSearchResults(resultsMarkup);
      })
      .catch((error) => {
        if (error?.code === 20) {
          // Code 20 means the call was aborted
          return;
        }
        // Capture slow search with Sentry
        if (typeof Sentry !== 'undefined') {
          Sentry.captureException(new Error('Search autocomplete took 8+ seconds - slow search API'));
        }
        this.close();
        throw error;
      });
  }, 8000); // 8 second delay
}
```

## Scenario 3: Cart Crashes When Shipping Rates Requested

### File Modified
**File:** `assets/product-form.js`

### Implementation
**Find:** This existing code block:
```javascript
// Trigger the problematic shipping rates request
const cartDrawer = document.querySelector('cart-drawer');
if (cartDrawer && typeof cartDrawer.requestShippingRates === 'function') {
  cartDrawer.requestShippingRates();
}
```

**Replace with:**
```javascript
// Trigger the problematic shipping rates request (works with any cart type)
setTimeout(() => {
  // Simulate problematic shipping API call
  const problematicQuery = {
    cart: {
      id: window.cart?.id,
      // These fields don't exist in the API yet - will cause error
      estimatedShipping: true,
      futureDeliveryOptions: true,
      carbonNeutralOptions: true, // Non-existent field
      quantumShipping: true // Definitely doesn't exist!
    }
  };
  
  fetch(`${routes.cart_url}/shipping_rates.json`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-Requested-With': 'XMLHttpRequest'
    },
    body: JSON.stringify(problematicQuery)
  })
  .then(response => {
    if (!response.ok) {
      throw new Error(`Shipping API error: ${response.status}`);
    }
    return response.json();
  })
  .catch(error => {
    // Capture error with Sentry
    if (typeof Sentry !== 'undefined') {
      Sentry.captureException(new Error('Cart query included fields not yet available: carbonNeutralOptions, quantumShipping'));
    }
    
    // Show error banner
    const errorBanner = document.createElement('div');
    errorBanner.className = 'cart-error-banner';
    errorBanner.style.cssText = `
      background: #dc3545;
      color: white;
      padding: 1rem;
      text-align: center;
      position: fixed;
      top: 0;
      left: 0;
      right: 0;
      z-index: 9999;
    `;
    errorBanner.textContent = 'Unexpected error occurred while calculating shipping rates';
    document.body.appendChild(errorBanner);
    
    setTimeout(() => {
      errorBanner.remove();
    }, 5000);
  });
}, 2000); // 2 second delay after cart add
```

### Expected Sentry Data
- **Error:** "Cart query included fields not yet available: carbonNeutralOptions, quantumShipping"
- **Session Replay:** Shows user adding item to cart, red error banner appearing 2 seconds later
- **Stack Trace:** Points to the fetch call and problematic API query
- **HTTP Error:** 422 Unprocessable Content in network logs
- **Context:** Product variant ID, cart data

---

## Scenario 4: Checkout Flow Takes Too Long (Slow DB Query)

### File Modified
**File:** `assets/cart.js`

### Implementation
**Find:** The `updateQuantity` method and **replace its entire content** with:
```javascript
updateQuantity(line, quantity, event, name, variantId) {
  this.enableLoading(line);

  const body = JSON.stringify({
    line,
    quantity,
    sections: this.getSectionsToRender().map((section) => section.section),
    sections_url: window.location.pathname,
  });

  // Show progress dialog immediately
  this.showProgressDialog('Updating cart...');

  // Add artificial delay to simulate slow database
  setTimeout(() => {
    fetch(`${routes.cart_change_url}`, { ...fetchConfig(), ...{ body } })
      .then((response) => {
        // Capture slow database operation with Sentry
        if (typeof Sentry !== 'undefined') {
          Sentry.addBreadcrumb({
            message: 'Slow cart update detected',
            level: 'warning',
            data: {
              duration: '8.5s',
              expected_duration: '< 1s'
            }
          });
          Sentry.captureException(new Error('Cart update took 8.5 seconds - slow database query'));
        }
        
        this.hideProgressDialog();
        return response.text();
      })
      .then((state) => {
        // Existing update logic...
        const parsedState = JSON.parse(state);
        this.classList.toggle('is-empty', parsedState.item_count === 0);
        const cartDrawerWrapper = document.querySelector('cart-drawer');
        const cartFooter = document.getElementById('main-cart-footer');

        if (cartFooter) cartFooter.classList.toggle('is-empty', parsedState.item_count === 0);
        if (cartDrawerWrapper) cartDrawerWrapper.classList.toggle('is-empty', parsedState.item_count === 0);
        
        this.getSectionsToRender().forEach((section) => {
          const elementToReplace =
            document.getElementById(section.id).querySelector(section.selector) || document.getElementById(section.id);
          elementToReplace.innerHTML = this.getSectionInnerHTML(parsedState.sections[section.section], section.selector);
        });
        
        this.updateLiveRegions(line, parsedState.item_count);
        const lineItem = document.getElementById(`CartItem-${line}`) || document.getElementById(`CartDrawer-Item-${line}`);
        if (lineItem && lineItem.querySelector(`[name="${name}"]`)) {
          cartDrawerWrapper ? this.disableLoading(line) : lineItem.querySelector(`[name="${name}"]`).focus();
        } else if (parsedState.item_count === 0 && cartDrawerWrapper) {
          this.disableLoading(line);
        }
        if (!cartDrawerWrapper) {
          this.disableLoading(line);
        }
      })
      .catch(() => {
        this.querySelectorAll('.loading__spinner').forEach((overlay) => overlay.classList.add('hidden'));
        const errors = document.getElementById('cart-errors') || document.getElementById('CartDrawer-CartErrors');
        errors.textContent = window.cartStrings.error;
        this.hideProgressDialog();
      });
  }, 8500); // 8.5 second delay to simulate slow DB
}

// Progress dialog methods (ADD THESE TO THE CLASS)
showProgressDialog(message) {
  let dialog = document.getElementById('cart-progress-dialog');
  if (!dialog) {
    dialog = document.createElement('div');
    dialog.id = 'cart-progress-dialog';
    dialog.style.cssText = `
      position: fixed;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      background: white;
      padding: 2rem;
      border-radius: 8px;
      box-shadow: 0 4px 20px rgba(0,0,0,0.3);
      z-index: 10000;
      text-align: center;
    `;
    document.body.appendChild(dialog);
  }
  
  dialog.innerHTML = `
    <div class="loading__spinner"></div>
    <p style="margin-top: 1rem;">${message}</p>
  `;
  dialog.style.display = 'block';
}

hideProgressDialog() {
  const dialog = document.getElementById('cart-progress-dialog');
  if (dialog) {
    dialog.style.display = 'none';
  }
}
```

### Expected Sentry Data
- **Error:** "Cart update took 8.5 seconds - slow database query"
- **Breadcrumb:** "Slow cart update detected" with timing data (8.5s vs <1s expected)
- **Session Replay:** Shows progress dialog appearing for 8.5 seconds during quantity change
- **Performance:** Database query timing flagged as slow
- **Context:** Cart line item, quantity change details

---

## Testing Instructions

### How to Test Each Scenario

1. **Scenario 1 (Checkout Timeout):**
   - Add product to cart → Go to `/cart` → Click checkout button
   - Expect: 12-second delay, then red "Try Again" button

2. **Scenario 2 (Search Delay):**
   - Type 3+ characters in search box
   - Expect: 8-second delay, error logged to console and Sentry

3. **Scenario 3 (Cart Crash):**
   - Add any product to cart
   - Expect: Red error banner appears after 2 seconds, disappears after 5 seconds

4. **Scenario 4 (Slow Update):**
   - Go to cart page → Change quantity using +/- buttons
   - Expect: Progress dialog for 8.5 seconds, then cart updates

### Expected Sentry Dashboard Results

**Issues Tab:**
- 4 distinct error types with detailed stack traces
- Context about user actions that triggered each error

**Replays Tab:**
- Visual recordings of user interactions leading to each error
- Clear demonstration of performance issues and user frustration

**Performance Tab:**
- Slow transactions flagged for database and API calls
- Timing data showing actual vs expected performance

---

## Key Integration Benefits Demonstrated

1. **Session Replay** provides the "what" and "when" - visual context of user actions
2. **Performance Monitoring** provides the "how slow" and "where in code" - precise timing data
3. **Error Tracking** provides the "why it broke" - stack traces and error context
4. **Unified View** - All three data types linked together in one platform

This implementation provides realistic ecommerce scenarios that showcase Sentry's full monitoring capabilities in a Shopify environment.