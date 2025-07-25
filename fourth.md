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

## Setting up the complex checkout flow scenario

Now we'll create a comprehensive checkout flow that demonstrates how session replay connects complex performance issues with user experience problems. This scenario simulates a realistic multi-step checkout process with multiple potential failure points, performance bottlenecks, and proper instrumentation to show how technical issues cascade into user frustration.

Modern ecommerce checkout flows involve multiple services working together: inventory validation, tax calculation, shipping rates, discount application, payment processing, and order creation. When any of these steps fails or performs slowly, users experience checkout abandonment without understanding why. Our implementation creates a realistic scenario where multiple services can fail independently, helping demonstrate how Sentry's unified monitoring reveals the complete picture.

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

### Step 2: Add Comprehensive Checkout Flow After the Existing Stylesheet

The file has a `{% stylesheet %}` section at the bottom. Add this JavaScript section right **after** the closing `{% endstylesheet %}` tag:

```html
<script>
document.addEventListener('DOMContentLoaded', function() {
  const checkoutBtn = document.getElementById('checkout');
  if (checkoutBtn && checkoutBtn.form && checkoutBtn.form.id === 'cart-form') {
    checkoutBtn.addEventListener('click', function(e) {
      if ({{ cart.item_count }} === 0) return;
      
      e.preventDefault();
      
      const buttonText = document.getElementById('checkout-button-text');
      const spinner = document.getElementById('checkout-spinner');
      
      if (!buttonText || !spinner) return;
      
      // Show loading state
      checkoutBtn.disabled = true;
      buttonText.style.display = 'none';
      spinner.style.display = 'inline-block';
      
      // Start the instrumented checkout flow
      processComplexCheckout()
        .then(() => {
          window.location.href = '{{ routes.cart_url }}/checkout';
        })
        .catch(error => {
          handleCheckoutError(error, checkoutBtn, buttonText, spinner);
        });
    });
  }
});

async function processComplexCheckout() {
  // Check if Sentry is available
  if (typeof Sentry === 'undefined') {
    console.warn('Sentry not available, falling back to basic flow');
    return basicCheckoutFlow();
  }

  return await Sentry.startSpan(
    { 
      name: "Complete Checkout Flow", 
      op: "checkout.process",
      attributes: {
        "checkout.cart_total": {{ cart.total_price | divided_by: 100.0 }},
        "checkout.item_count": {{ cart.item_count }},
        "checkout.currency": "{{ cart.currency.iso_code }}"
      }
    },
    async (checkoutSpan) => {
      const checkoutData = {
        cartTotal: {{ cart.total_price | divided_by: 100.0 }},
        itemCount: {{ cart.item_count }},
        currency: '{{ cart.currency.iso_code }}',
        startTime: Date.now(),
        items: [
          {% for item in cart.items %}
          {
            id: {{ item.product_id }},
            variantId: {{ item.variant_id }},
            quantity: {{ item.quantity }},
            price: {{ item.price | divided_by: 100.0 }}
          }{% unless forloop.last %},{% endunless %}
          {% endfor %}
        ]
      };

      try {
        // Step 1: Validate cart items and inventory
        const validationResult = await Sentry.startSpan(
          { 
            name: "Validate Cart Items", 
            op: "checkout.validate",
            attributes: {
              "validation.item_count": checkoutData.itemCount
            }
          },
          () => validateCartItems(checkoutData)
        );
        
        // Step 2: Calculate taxes (depends on validation)
        const taxResult = await Sentry.startSpan(
          { 
            name: "Calculate Taxes", 
            op: "checkout.tax",
            attributes: {
              "tax.cart_total": checkoutData.cartTotal,
              "tax.calculation_type": "api"
            }
          },
          () => calculateTaxes(checkoutData, validationResult)
        );
        
        // Step 3: Parallel operations (shipping + discounts)
        const [shippingResult, discountResult] = await Promise.all([
          Sentry.startSpan(
            { 
              name: "Calculate Shipping Rates", 
              op: "checkout.shipping",
              attributes: {
                "shipping.providers_count": 3,
                "shipping.destination": "US"
              }
            },
            () => calculateShippingRates(checkoutData)
          ),
          Sentry.startSpan(
            { 
              name: "Apply Discounts", 
              op: "checkout.discounts",
              attributes: {
                "discount.cart_total": checkoutData.cartTotal
              }
            },
            () => applyDiscounts(checkoutData)
          )
        ]);
        
        // Step 4: Payment authorization
        const finalTotal = checkoutData.cartTotal + taxResult.taxAmount + shippingResult.selected.cost - discountResult.discountAmount;
        const paymentResult = await Sentry.startSpan(
          { 
            name: "Authorize Payment", 
            op: "checkout.payment",
            attributes: {
              "payment.amount": finalTotal,
              "payment.currency": checkoutData.currency,
              "payment.method": "credit_card"
            }
          },
          () => authorizePayment({
            ...checkoutData,
            taxes: taxResult,
            shipping: shippingResult,
            discounts: discountResult
          })
        );
        
        // Step 5: Create order record
        const orderResult = await Sentry.startSpan(
          { 
            name: "Create Order Record", 
            op: "checkout.order",
            attributes: {
              "order.final_total": finalTotal,
              "order.processing_time_ms": Date.now() - checkoutData.startTime
            }
          },
          () => createOrderRecord({
            ...checkoutData,
            taxes: taxResult,
            shipping: shippingResult,
            discounts: discountResult,
            payment: paymentResult
          })
        );
        
        // Add success attributes to parent span
        checkoutSpan.setAttributes({
          "checkout.success": true,
          "checkout.total_time_ms": Date.now() - checkoutData.startTime,
          "checkout.final_amount": finalTotal
        });
        
        return orderResult;
        
      } catch (error) {
        // Mark span as failed and add error context
        checkoutSpan.setStatus({ code: 2, message: error.message });
        checkoutSpan.setAttributes({
          "checkout.success": false,
          "checkout.error_step": error.step || "unknown",
          "checkout.error_message": error.message
        });
        
        // Capture detailed error
        Sentry.captureException(error, {
          tags: {
            operation: 'complex_checkout_flow',
            checkout_step: error.step || 'unknown',
            business_impact: 'high'
          },
          extra: {
            checkout_data: checkoutData,
            failed_at: error.step,
            total_value: checkoutData.cartTotal,
            processing_time: Date.now() - checkoutData.startTime
          }
        });
        
        throw error;
      }
    }
  );
}

// Fallback for when Sentry isn't available
async function basicCheckoutFlow() {
  // Simple delay to simulate processing
  await new Promise(resolve => setTimeout(resolve, 1500));
  if (Math.random() < 0.1) {
    throw new Error('Basic checkout failed');
  }
  return { success: true };
}

// Step 1: Cart validation with inventory check
async function validateCartItems(checkoutData) {
  return new Promise((resolve, reject) => {
    setTimeout(async () => {
      try {
        // Simulate checking each item's availability (N operations)
        for (let i = 0; i < checkoutData.items.length; i++) {
          await new Promise(resolve => setTimeout(resolve, 80));
        }
        
        // 10% chance of inventory issues
        if (Math.random() < 0.1) {
          const error = new Error('Item no longer available in inventory');
          error.step = 'inventory_validation';
          reject(error);
          return;
        }
        
        resolve({ 
          validated: true, 
          timestamp: Date.now(),
          itemsChecked: checkoutData.items.length
        });
      } catch (error) {
        error.step = 'cart_validation';
        reject(error);
      }
    }, 150);
  });
}

// Step 2: Tax calculation
async function calculateTaxes(checkoutData, validationResult) {
  return new Promise((resolve, reject) => {
    setTimeout(async () => {
      try {
        // Simulate tax service API call (will 404 but that's fine for demo)
        const response = await fetch('/cart/taxes.json', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            items: checkoutData.items,
            address: { country: 'US', state: 'CA', zip: '90210' }
          })
        }).catch(() => {
          // Expected to fail, use fallback calculation
          return { ok: true };
        });
        
        // 5% chance of tax calculation failure
        if (Math.random() < 0.05) {
          const error = new Error('Tax service temporarily unavailable');
          error.step = 'tax_calculation';
          reject(error);
          return;
        }
        
        const taxRate = 0.0875; // California tax rate
        resolve({ 
          taxRate: taxRate,
          taxAmount: checkoutData.cartTotal * taxRate,
          timestamp: Date.now(),
          service: 'tax_api'
        });
      } catch (error) {
        error.step = 'tax_calculation';
        reject(error);
      }
    }, 280);
  });
}

// Step 3: Shipping calculation (the intentional bottleneck)
async function calculateShippingRates(checkoutData) {
  return new Promise((resolve, reject) => {
    setTimeout(async () => {
      try {
        const providers = ['UPS', 'FedEx', 'USPS'];
        const rates = [];
        
        // Simulate API call to each shipping provider
        for (const provider of providers) {
          await new Promise(resolve => setTimeout(resolve, 400)); // Each takes 400ms
          rates.push({
            provider,
            cost: 5.99 + Math.random() * 10,
            days: Math.floor(Math.random() * 5) + 1
          });
        }
        
        // 15% chance of shipping service failure
        if (Math.random() < 0.15) {
          const error = new Error('Shipping service temporarily unavailable');
          error.step = 'shipping_calculation';
          reject(error);
          return;
        }
        
        resolve({
          rates,
          selected: rates[0], // Select cheapest option
          timestamp: Date.now(),
          providersChecked: providers.length
        });
      } catch (error) {
        error.step = 'shipping_rates';
        reject(error);
      }
    }, 1200); // This is the bottleneck - 1.2 seconds base + provider calls
  });
}

// Step 3b: Discount application (parallel with shipping)
async function applyDiscounts(checkoutData) {
  return new Promise((resolve) => {
    setTimeout(() => {
      const discountAmount = checkoutData.cartTotal > 50 ? 5.00 : 0;
      resolve({
        discountAmount,
        discountCode: discountAmount > 0 ? 'AUTO5' : null,
        timestamp: Date.now(),
        eligibleForDiscount: checkoutData.cartTotal > 50
      });
    }, 200);
  });
}

// Step 4: Payment authorization
async function authorizePayment(fullCheckoutData) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      // 5% chance of payment failure
      if (Math.random() < 0.05) {
        const error = new Error('Payment authorization declined');
        error.step = 'payment_authorization';
        reject(error);
        return;
      }
      
      const finalAmount = fullCheckoutData.cartTotal + 
                         fullCheckoutData.taxes.taxAmount + 
                         fullCheckoutData.shipping.selected.cost - 
                         fullCheckoutData.discounts.discountAmount;
      
      resolve({
        authorizationId: 'auth_' + Math.random().toString(36).substr(2, 9),
        amount: finalAmount,
        timestamp: Date.now(),
        gateway: 'stripe'
      });
    }, 350);
  });
}

// Step 5: Order creation
async function createOrderRecord(fullOrderData) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      // 2% chance of database failure
      if (Math.random() < 0.02) {
        const error = new Error('Order creation failed - database timeout');
        error.step = 'order_creation';
        reject(error);
        return;
      }
      
      resolve({
        orderId: 'order_' + Math.random().toString(36).substr(2, 9),
        timestamp: Date.now(),
        totalProcessingTime: Date.now() - fullOrderData.startTime
      });
    }, 180);
  });
}

function handleCheckoutError(error, button, buttonText, spinner) {
  button.disabled = false;
  spinner.style.display = 'none';
  buttonText.style.display = 'inline-block';
  
  const errorMessages = {
    'inventory_validation': 'Item Unavailable',
    'cart_validation': 'Cart Error',
    'tax_calculation': 'Tax Service Error',
    'shipping_calculation': 'Shipping Error', 
    'shipping_rates': 'Shipping Unavailable',
    'payment_authorization': 'Payment Declined',
    'order_creation': 'Order Failed'
  };
  
  buttonText.textContent = errorMessages[error.step] || 'Checkout Error';
  button.style.backgroundColor = '#dc3545';
  button.style.color = 'white';
  
  // Reset after 4 seconds
  setTimeout(() => {
    buttonText.textContent = '{{ 'content.checkout' | t }}';
    button.style.backgroundColor = '';
    button.style.color = '';
  }, 4000);
}
</script>
```

This comprehensive implementation demonstrates realistic ecommerce challenges by creating a multi-step checkout process where each stage can fail independently. The code uses Sentry's performance monitoring capabilities with nested spans to track each operation's timing and success rate.

**Key features of this implementation:**

- **Sequential Dependencies**: Cart validation must complete before tax calculation, mirroring real checkout flows
- **Parallel Operations**: Shipping and discount calculations run simultaneously for efficiency
- **Performance Bottlenecks**: Shipping calculation intentionally takes 1.2+ seconds, simulating slow third-party APIs
- **Multiple Failure Points**: Each step has realistic failure rates (inventory 10%, shipping 15%, payment 5%)
- **Detailed Instrumentation**: Every operation is wrapped in Sentry spans with relevant attributes and timing data

The checkout flow spans demonstrate how performance issues cascade through dependent operations, while session replay captures the user experience during these delays and failures.

## Testing the complex checkout flow with mobile device feedback

Add several products to your cart to create a realistic checkout scenario, then navigate to the `/cart` page on both desktop and mobile devices. Click the **Checkout** button to trigger the complex checkout process. You'll notice the loading state persists longer than typical ecommerce flows, especially during the shipping calculation phase which acts as the primary bottleneck.

The checkout process may succeed after several seconds, or it may fail at any step with specific error messages like "Shipping Error" or "Payment Declined". Each failure represents a different technical issue that would require different debugging approaches in production.

![Screenshot of Shopify cart page on mobile showing the checkout button with loading spinner during the complex multi-step checkout process](images/mobile-checkout-complex.png)

Test the checkout flow multiple times to experience different failure scenarios. The randomized failure rates mean you'll encounter various error conditions, each providing different performance traces and error context in Sentry. While experiencing checkout failures on your mobile device, click the purple feedback widget to submit reports about the specific issues you encounter.

When submitting feedback through the mobile widget, describe the problems as a real user would: "Checkout took forever to load and then failed with shipping error" or "Payment was declined even though my card works fine." This realistic feedback helps demonstrate how user reports connect directly with the detailed performance data and error traces captured by Sentry.

## Viewing user feedback in Sentry

Navigate to your Sentry project dashboard and click "User Feedback" in the sidebar. You'll see the feedback submissions you created, each describing different aspects of the checkout experience problems. The feedback entries now connect to much more sophisticated technical data than simple API timeouts.

![Screenshot of Sentry User Feedback dashboard showing the complex checkout error reports with user descriptions and associated technical performance data](images/user-feedback-complex-dashboard.png)

Each feedback entry links to comprehensive performance traces showing exactly which step in the checkout process failed, how long each operation took, and what the user experienced during the entire flow. This connection eliminates guesswork about which technical issues correlate with user frustration and checkout abandonment.

## Connecting feedback to session replay and comprehensive performance data

Click on any feedback entry to see the complete context that Sentry automatically captured for the complex checkout flow. The feedback detail view now connects multiple layers of technical information that would traditionally require separate tools and manual correlation.

### Mobile Session Replay with Multi-Step Process

The session replay shows the complete user journey through the extended checkout process. You can watch users interact with the cart, click checkout, observe the prolonged loading state, and see their reaction to different failure scenarios. Mobile session replay captures the frustration that builds during the shipping calculation delay, particularly noticeable when users expect immediate responses.

![Screenshot of Sentry mobile session replay showing the complete multi-step checkout sequence with extended loading states and various failure points](images/mobile-session-replay-complex.png)

The replay reveals user behavior patterns during extended loading periods. Mobile users often switch between apps or scroll around the page while waiting, and session replay captures these context switches that indicate growing impatience with checkout performance.

### Comprehensive Performance Traces

The performance trace now shows a waterfall of all checkout operations with nested span relationships. You can see how cart validation leads to tax calculation, how shipping and discount operations run in parallel, and where the process bottlenecks or fails completely.

![Screenshot of Sentry performance trace showing the complete checkout flow waterfall with nested spans for validation, tax calculation, shipping, discounts, payment, and order creation](images/performance-trace-complex.png)

Performance traces reveal the cascade effect of slow operations. When shipping calculation takes 2+ seconds, the entire checkout experience feels sluggish even if other operations complete quickly. The trace shows exactly which operations consume the most time and how parallel processing helps mitigate some delays.

### Detailed Error Context with Business Impact

Error details now include rich context about which step failed, what the cart contents were, how long the process had been running, and what the business impact represents. Each error includes the complete checkout state, making it possible to reproduce issues or understand why specific combinations of cart contents trigger failures.

![Screenshot of Sentry error details showing complex checkout failure with complete context including cart data, processing time, failed step, and business impact assessment](images/error-details-complex.png)

Error context transforms generic failure messages into actionable debugging information. Instead of knowing that "checkout failed," you have specific details about cart value, processing time, which integration failed, and how many customers might be affected by similar issues.

### Network Requests and Third-Party Integration Monitoring

The network tab shows all API calls made during the checkout process, including the intentional tax service call that demonstrates how to monitor third-party integrations. You can see request timing, headers, payloads, and responses for each step in the checkout flow.

![Screenshot of Sentry network monitoring showing API calls during checkout including tax service, shipping providers, and payment gateway requests with timing and response data](images/network-requests-complex.png)

Network monitoring reveals how third-party service performance affects user experience. Slow shipping provider APIs or unreliable tax calculation services directly impact checkout completion rates, and Sentry's integration shows exactly where these dependencies create user experience problems.

## The complete debugging picture

The complex checkout flow scenario demonstrates why unified monitoring becomes essential for modern ecommerce applications. Traditional approaches require manually correlating data across separate systems for session recording, performance analysis, error tracking, and business intelligence, often leading to incomplete understanding of how technical issues impact business metrics.

When a customer reports that "checkout doesn't work," separate monitoring tools force you to check user session replay software in one system, look up performance data in another platform, search for related errors in a third tool, and manually piece together the complete story. Sentry's integrated approach automatically links session replays with performance traces and error information, ensuring you see the complete technical and user experience context.

The unified platform enables sophisticated analysis that separate tools cannot provide. You can identify patterns where specific performance bottlenecks consistently lead to user abandonment, correlate error conditions with support ticket volume, and prioritize fixes based on actual business impact rather than just technical severity. This holistic view helps you understand not just what broke, but how technical issues affect revenue and customer satisfaction.

Performance optimization becomes data-driven when you can see exactly how checkout delays correlate with abandonment rates. The sample rate for Sentry session replay can be optimized based on your traffic volume and debugging needs, balancing comprehensive monitoring with data storage costs while ensuring critical business flows are always monitored.

Start implementing comprehensive monitoring on your most critical user flows, then expand coverage based on the debugging value you discover. Focus particularly on checkout processes, cart operations, and payment flows where technical issues directly impact revenue. When users report problems with your ecommerce application, you'll immediately see their complete experience and identify the technical root cause, leading to faster fixes and better customer experiences.

## Further Reading

To learn more about implementing session replay in your ecommerce environment, explore these resources:

- [Session Replay Product Overview](https://sentry.io/product/session-replay/) - Complete guide to Sentry's session replay capabilities
- [Session Replay Documentation](https://docs.sentry.io/product/explore/session-replay/) - Technical documentation and configuration options
- [Getting Started with Session Replay](https://blog.sentry.io/getting-started-with-session-replay/) - Step-by-step implementation guide
- [Session Replay for Mobile Apps](https://blog.sentry.io/session-replay-for-mobile-is-now-generally-available-see-what-your-users-see/) - Mobile session replay features and React Native integration
- [User Feedback Widget for Mobile Apps](https://blog.sentry.io/user-feedback-widget-for-mobile-apps/) - Connecting user feedback with session replay data
- [Flutter SDK Integration](https://blog.sentry.io/introducing-sentrys-flutter-sdk-9-0/) - Session replay for Flutter applications
- [Tilled Customer Success Story](https://sentry.io/customers/tilled/) - Real-world implementation example from a payments platform