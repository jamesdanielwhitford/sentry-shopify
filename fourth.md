# Using Sentry session replay to debug ecommerce performance issues

When customers abandon their shopping carts or complain about checkout problems, traditional error logs leave you guessing about what actually happened. You might see a "payment timeout" error in your logs, but did the user wait patiently for 30 seconds, or did they rage-click the checkout button dozens of times before giving up? Understanding this difference is crucial for fixing the right problem.

Session replay technology transforms ecommerce debugging by showing exactly what users experienced during errors and performance issues. Unlike traditional analytics that show aggregate data, session replay captures and recreates individual user interactions, creating a video-like recording of what users see and do during critical moments like product searches, cart additions, and checkout processes.

This guide demonstrates how to implement comprehensive monitoring in a Shopify environment using Sentry's unified platform, where session replay, performance data, and error information are automatically linked. We'll create realistic ecommerce failure scenarios and show you how to connect user frustration with technical root causes, eliminating the traditional gap between customer reports and technical debugging.

## Setting up comprehensive monitoring in your Shopify environment

We'll demonstrate Sentry's unified monitoring capabilities using a Shopify store with realistic ecommerce functionality. This setup process shows you how to integrate [session replay](https://sentry.io/product/session-replay/) with performance monitoring and error tracking while capturing the context needed for effective debugging.

### Setting Up Sentry

To add Sentry monitoring to your Shopify store, follow these steps:

1. Sign up for a [Sentry account](https://sentry.io).

2. Create a new project by clicking **Create Project**.

3. Choose **Browser JavaScript** as your platform, and give your project a name like "Shopify Store Monitoring". Click **Create Project** and then **Configure SDK** when prompted.

![Create a Sentry JavaScript project through the UI](images/create-project.png)

4. After creating the project, Sentry will provide you with a data source name (DSN), a unique identifier that tells the Sentry SDK where to send events from your Shopify store.

![DSN for the Sentry project in the Sentry UI](images/dsn-value.png)

We will use this DSN in the next section when integrating the Sentry SDK into the Shopify theme to enable session replay, performance monitoring, and error tracking for your ecommerce application.

### Setting Up Shopify

To demonstrate Sentry's session replay capabilities in a realistic ecommerce environment, we will set up a Shopify development store.

1. **Create a Shopify Partner account** at [partners.shopify.com](https://partners.shopify.com) if you don't already have one. Partner accounts let you create unlimited development stores for testing without monthly fees.

2. **Create a new development store** by clicking "Stores" in your Partner dashboard, then "Add store". Choose "Development store" and select "Create a store to test and build" as your purpose.

3. **Create your store** after giving it a name, click **Create Development Store**.

![Creating a new Shopify development store through the Partner dashboard](images/create-shopify-store.png)

3. **Install your store theme.** Navigate to "Online Store" then "Themes" in your Shopify admin panel. Install the Horizon theme to follow along with this guide. Once you have added this theme, click **Publish**.

![Installing the Horizon theme in Shopify store](images/install-horizon-theme.png)

4. **Get the store password** by clicking **See store password**. The store requires a password while in development mode.

5. **Add sample products** with different price points to create realistic shopping scenarios. Navigate to "Products" then "Add product" and create a demo product, and click **Save**.

![Adding sample products to Shopify development store](images/add-sample-products.png)

### Integrate Sentry into your Shopify Theme

In your Shopify admin, navigate to "Online Store" then "Themes". Click the three dots on your active theme, then "Edit code". 

Open the `layout/theme.liquid` file and add the Sentry SDK to the `<head>` section, before `{{ content_for_header }}`.

```html
<script
  src="https://browser.sentry-cdn.com/9.40.0/bundle.tracing.replay.feedback.min.js"
  integrity="sha384-Pe41llaXfNg82Pkv5LMIFFis6s9XOSxijOH52r55t4AU9mzbm6ZzQ/I0Syp8hkk9"
  crossorigin="anonymous"
></script>

<script>
  Sentry.onLoad(function() {
    Sentry.init({
    dsn: "YOUR_DSN_HERE",
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
      
      tracesSampleRate: 1.0,
      replaysSessionSampleRate: 1.0,
      replaysOnErrorSampleRate: 1.0,

      tracePropagationTargets: ["localhost", /^https?:\/\//],
      
      integrations: [
        Sentry.feedbackIntegration({
          colorScheme: "system",
          enableScreenshot: true,
          showBranding: false,
          showName: true,
          showEmail: true,
          isRequiredEmail: true,
        }),
        Sentry.replayIntegration(),
        Sentry.browserTracingIntegration()
      ],
      
      beforeSend(event) {
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

Replace `YOUR_DSN_HERE` with your actual Sentry project DSN. This configuration sets up three essential monitoring capabilities for your ecommerce store:

**Session Replay**: The `replaysSessionSampleRate: 1.0` and `replaysOnErrorSampleRate: 1.0` settings capture 100% of user sessions so every error is accompanied by session replay data.

**Performance Monitoring**: The `tracesSampleRate: 1.0` setting enables performance monitoring for all transactions.

**Ecommerce Context**: The `beforeSend` function automatically adds cart information, customer details, and page-specific data to every event.

**User Feedback Widget**: The `feedbackIntegration` configuration creates a persistent feedback widget that appears as a floating button in your store. When users encounter problems during their shopping experience, they can click this widget to submit feedback, which automatically captures their current session replay along with screenshots of the exact moment they experienced the issue.

![Sentry user feedback widget appearing as a purple feedback button in the bottom-right corner of a Shopify store.](images/feedback-widget-setup.png)

## Testing everything with the feedback widget

Before implementing error scenarios, let's verify that our monitoring setup works correctly. Navigate to your Shopify store and interact with different pages. Add products to your cart, browse around, and notice the purple feedback widget in the bottom-right corner. Click the feedback widget to test the feedback submission process, then check your Sentry dashboard to confirm that session replays are being captured and user feedback is being recorded.

The feedback widget serves as your direct connection to customer experience issues. When users encounter problems during their shopping journey, they can immediately report issues while the context is fresh, and Sentry automatically associates their feedback with the technical data needed for debugging.

## Setting up the checkout error scenario

Now we'll create a realistic checkout failure that demonstrates how session replay connects user frustration with technical root causes. This scenario simulates a shipping rates API that takes too long to respond and then fails completely, creating the kind of checkout abandonment issues that directly impact ecommerce revenue.

We need to properly modify the existing `snippets/cart-summary.liquid` file without breaking its Liquid template structure. The key is to enhance the existing checkout button and add our error simulation logic after the existing stylesheet section.

### Step 1: Modify the Existing Button Structure

In your Shopify admin, navigate to "Online Store" then "Themes", click the three dots on your active theme, then "Edit code". Open `snippets/cart-summary.liquid` and find this existing button:

```html
<button
  type="submit"
  id="checkout"
  class="cart__checkout-button button"
  name="checkout"
  {% if cart == empty %}
    disabled
  {% endif %}
  form="cart-form"
>
  {{ 'content.checkout' | t }}
</button>
```

Replace it with:

```html
<button
  type="submit"
  id="checkout"
  class="cart__checkout-button button"
  name="checkout"
  {% if cart == empty %}
    disabled
  {% endif %}
  form="cart-form"
>
  <span id="checkout-button-text">{{ 'content.checkout' | t }}</span>
  <span id="checkout-spinner" style="display: none;">⏳</span>
</button>
```

### Step 2: Add JavaScript After the Existing Stylesheet

Tha file has a `{% stylesheet %}` section at the bottom. Add this JavaScript section right **after** the closing `{% endstylesheet %}` tag:

```html
<script>
document.addEventListener('DOMContentLoaded', function() {
  const checkoutBtn = document.getElementById('checkout');
  if (checkoutBtn && checkoutBtn.form && checkoutBtn.form.id === 'cart-form') {
    checkoutBtn.addEventListener('click', function(e) {
      // Only intercept if cart has items
      if ({{ cart.item_count }} === 0) return;
      
      e.preventDefault();
      
      const buttonText = document.getElementById('checkout-button-text');
      const spinner = document.getElementById('checkout-spinner');
      
      if (!buttonText || !spinner) return;
      
      // Show loading state
      checkoutBtn.disabled = true;
      buttonText.style.display = 'none';
      spinner.style.display = 'inline-block';
      
      // Simulate realistic shipping calculation that times out
      calculateShippingForCheckout()
        .then(() => {
          // Success - proceed to actual checkout
          window.location.href = '{{ routes.cart_url }}/checkout';
        })
        .catch(error => {
          // Handle shipping calculation failure
          handleShippingError(error, checkoutBtn, buttonText, spinner);
        });
    });
  }
});

async function calculateShippingForCheckout() {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), 8000);
  
  try {
    const response = await fetch('{{ routes.cart_url }}/shipping_rates.json', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Requested-With': 'XMLHttpRequest'
      },
      body: JSON.stringify({
        shipping_address: {
          country: 'US',
          province: 'CA',
          zip: '90210'
        }
      }),
      signal: controller.signal
    });
    
    clearTimeout(timeoutId);
    
    if (!response.ok) {
      throw new Error(`Shipping service unavailable (HTTP ${response.status})`);
    }
    
    return await response.json();
  } catch (error) {
    clearTimeout(timeoutId);
    
    if (error.name === 'AbortError') {
      throw new Error('Shipping calculation timed out after 8 seconds');
    }
    throw error;
  }
}

function handleShippingError(error, button, buttonText, spinner) {
  // Capture error for monitoring (this is a critical checkout flow)
  if (typeof Sentry !== 'undefined') {
    Sentry.captureException(error, {
      tags: {
        operation: 'checkout_flow',
        step: 'shipping_calculation'
      },
      extra: {
        cart_total: '{{ cart.total_price | money_without_currency }}',
        cart_currency: '{{ cart.currency.iso_code }}',
        item_count: {{ cart.item_count }},
        shipping_country: 'US'
      }
    });
  }
  
  // Reset button to error state
  button.disabled = false;
  spinner.style.display = 'none';
  buttonText.style.display = 'inline-block';
  buttonText.textContent = 'Try Again - Shipping Error';
  button.style.backgroundColor = '#dc3545';
  button.style.color = 'white';
  
  // Reset button after 3 seconds
  setTimeout(() => {
    buttonText.textContent = '{{ 'content.checkout' | t }}';
    button.style.backgroundColor = '';
    button.style.color = '';
  }, 3000);
}
</script>
```

This implementation creates a realistic shipping calculation failure that demonstrates how technical problems manifest as user experience issues. The API call attempts to calculate shipping rates but fails either due to timeout or server error, leaving users frustrated and unable to complete their purchase.

The code follows production patterns by using Shopify's actual routing system with `{{ routes.cart_url }}`, including real cart data in error reports using Liquid template variables, and manually capturing the handled error with `Sentry.captureException()` since this represents a critical business flow that needs monitoring even when gracefully handled.

## Testing the error with mobile device feedback

Add some products to your cart and navigate to the `/cart` page on both desktop and mobile devices. Click the **Checkout** button to trigger the shipping calculation error. On mobile devices, the extended loading time and eventual failure feel particularly frustrating because users expect mobile interactions to be immediate and seamless.

![Screenshot of Shopify cart page on mobile showing the checkout button with loading spinner during the failed shipping calculation](images/mobile-checkout-failed.png)

While experiencing this checkout failure on your mobile device, click the purple feedback widget to submit a report about the problem. The mobile experience makes user frustration more pronounced, and the feedback widget provides an immediate outlet for users to report checkout issues while they're actively experiencing them.

When submitting feedback through the mobile widget, describe the checkout problem as a real user would: "Checkout button got stuck calculating shipping and then showed an error. Very frustrating when trying to buy something quickly." This realistic feedback helps demonstrate how user reports connect directly with technical debugging data.

## Viewing user feedback in Sentry

Navigate to your Sentry project dashboard and click "User Feedback" in the sidebar. You'll see the feedback submission you just created, complete with the user's description of the checkout problem and their contact information if provided.

![Screenshot of Sentry User Feedback dashboard showing the checkout error report with user description and associated technical data](images/user-feedback-dashboard.png)

The feedback entry shows not just the user's description of the problem, but also links directly to the associated session replay and error data. This connection eliminates the traditional gap between customer support reports and technical investigation. Instead of support teams describing problems to engineering teams, you have direct access to both the user's experience and the technical root cause.

## Connecting feedback to session replay and technical data

Click on the feedback entry to see the complete context that Sentry automatically captured. The feedback detail view connects four critical pieces of information that traditionally require separate tools to correlate.

### Mobile Session Replay

The session replay shows exactly how the checkout failure appeared on the user's mobile device. You can watch the user add items to their cart, navigate to checkout, click the button, wait through the loading state, and see the eventual error message. Mobile session replay captures touch interactions, scroll behavior, and the specific timing of user actions that led to frustration.

![Screenshot of Sentry mobile session replay showing the complete checkout sequence from cart interaction through the shipping calculation failure](images/mobile-session-replay-checkout.png)

Mobile session replay reveals behavior patterns unique to mobile users. Desktop users might wait patiently during loading states, but mobile users often expect immediate responses and quickly abandon slow processes. The replay shows precisely when users become frustrated and how they react to checkout delays.

### Network Request Details

The network tab within the session replay shows the complete shipping rates API call, including request headers, payload, timing information, and the eventual failure response. You can see exactly how long the API took to respond and what error was returned.

![Screenshot of Sentry network details showing the shipping rates API request timing, headers, and failure response with HTTP status code](images/network-request-details.png)

Network details reveal whether checkout failures stem from slow API responses, server errors, or client-side timeouts. In this case, the shipping rates API either times out after 8 seconds or returns an HTTP error status, providing clear technical context for the user experience problem captured in the session replay.

### Performance Trace

The performance trace shows the complete transaction timeline from user interaction through API failure. Timing data reveals that shipping calculation requests should complete in under 2 seconds, but this particular request either times out or fails completely, creating an unacceptable user experience.

![Screenshot of Sentry performance trace showing the checkout transaction timeline with slow shipping calculation API highlighted](images/performance-trace-checkout.png)

Performance tracing connects user actions with backend operations, showing how API slowness cascades into user experience problems. The trace reveals not just that the shipping calculation failed, but how that failure fits into the broader context of checkout performance.

### Error Context and Stack Trace

The error details show the exact JavaScript exception captured when the shipping calculation fails, complete with stack trace information and the custom context added during error capture. Error context includes cart information, shipping address, and API endpoint details that help with debugging.

![Screenshot of Sentry error details showing the shipping calculation timeout error with full stack trace and ecommerce context](images/error-details-context.png)

Error context transforms generic timeout messages into actionable debugging information. Instead of just knowing that "something failed during checkout," you have specific details about cart contents, shipping calculations, and API endpoints that enable targeted fixes.

## The complete debugging picture

The checkout failure scenario demonstrated why session replay technology becomes exponentially more valuable when integrated with performance monitoring and error tracking in a unified platform. Traditional approaches force you to manually correlate data across different systems for user session replay software, performance analysis, and error investigation, often leading to incomplete understanding of user experience problems.

When a customer reports that "checkout doesn't work," separate monitoring tools require you to check session recording in one system, look up performance data in another platform, and search for related errors in a third tool. Sentry's integrated approach automatically links session replays with performance traces and error information, eliminating context switching and ensuring you never miss critical connections between user experience and technical issues.

The unified platform enables sophisticated analysis that wouldn't be possible with separate tools. You can identify patterns where specific performance issues consistently lead to user abandonment, or where certain error conditions correlate with increased support ticket volume. This helps you prioritize fixes based on actual business impact rather than just technical severity.

Start implementing session replay on your most critical user flows, then expand coverage based on the debugging value you discover. Focus on scenarios where traditional logging provides insufficient context about user experience problems, particularly during checkout processes and cart operations where user frustration directly impacts revenue. The sample rate for Sentry session replay can be optimized based on your specific needs, balancing comprehensive monitoring with data volume considerations.

When users report problems with your ecommerce application, you'll immediately see exactly what they experienced and identify the technical root cause, leading to faster fixes and better customer experiences.

## Further Reading

To learn more about implementing session replay in your ecommerce environment, explore these resources:

- [Session Replay Product Overview](https://sentry.io/product/session-replay/) - Complete guide to Sentry's session replay capabilities
- [Session Replay Documentation](https://docs.sentry.io/product/explore/session-replay/) - Technical documentation and configuration options
- [Getting Started with Session Replay](https://blog.sentry.io/getting-started-with-session-replay/) - Step-by-step implementation guide
- [Session Replay for Mobile Apps](https://blog.sentry.io/session-replay-for-mobile-is-now-generally-available-see-what-your-users-see/) - Mobile session replay features and React Native integration
- [User Feedback Widget for Mobile Apps](https://blog.sentry.io/user-feedback-widget-for-mobile-apps/) - Connecting user feedback with session replay data
- [Flutter SDK Integration](https://blog.sentry.io/introducing-sentrys-flutter-sdk-9-0/) - Session replay for Flutter applications
- [Tilled Customer Success Story](https://sentry.io/customers/tilled/) - Real-world implementation example from a payments platform