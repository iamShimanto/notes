# Data_Layer_Next_JS

### 1) Add GTM in app/layout.js
#### Create / edit: app/layout.js

```js
import Script from "next/script";

const GTM_ID = process.env.NEXT_PUBLIC_GTM_ID;

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <head>
        {GTM_ID ? (
          <Script
            id="gtm-base"
            strategy="afterInteractive"
            dangerouslySetInnerHTML={{
              __html: `
                (function(w,d,s,l,i){w[l]=w[l]||[];w[l].push({'gtm.start':
                new Date().getTime(),event:'gtm.js'});var f=d.getElementsByTagName(s)[0],
                j=d.createElement(s),dl=l!='dataLayer'?'&l='+l:'';j.async=true;j.src=
                'https://www.googletagmanager.com/gtm.js?id='+i+dl;f.parentNode.insertBefore(j,f);
                })(window,document,'script','dataLayer','${GTM_ID}');
              `,
            }}
          />
        ) : null}
      </head>

      <body>
        {GTM_ID ? (
          <noscript>
            <iframe
              src={`https://www.googletagmanager.com/ns.html?id=${GTM_ID}`}
              height="0"
              width="0"
              style={{ display: "none", visibility: "hidden" }}
            />
          </noscript>
        ) : null}

        {children}
      </body>
    </html>
  );
}
```
### 2) Create a DataLayer helper (App Router safe)
#### Create: src/lib/datalayer.js (or lib/datalayer.js if you don’t use src/)
```js
export function pushToDataLayer(payload) {
  if (typeof window === "undefined") return;
  window.dataLayer = window.dataLayer || [];
  window.dataLayer.push(payload);
}

export function trackEcomEvent(event, ecommerce, extra = {}) {
  // recommended: clear previous ecommerce object so GTM doesn't reuse old items
  pushToDataLayer({ ecommerce: null });

  pushToDataLayer({
    event,
    ecommerce,
    ...extra,
  });
}

```
### Fire events from Client Components
#### A) view_item on Product Details Page
##### Your product page component must be a client component:
```js
"use client";

import { useEffect } from "react";
import { trackEcomEvent } from "@/lib/datalayer";

export default function ProductDetails({ product }) {
  useEffect(() => {
    if (!product?._id) return;

    trackEcomEvent("view_item", {
      currency: "BDT",
      value: product.price,
      items: [
        {
          item_id: product._id,
          item_name: product.name,
          item_category: product.category || "Uncategorized",
          price: product.price,
        },
      ],
    });
  }, [product]);

  return <div>{product?.name}</div>;
}
```
#### B) add_to_cart on button click
```js
"use client";

import { trackEcomEvent } from "@/lib/datalayer";

export default function AddToCartButton({ product }) {
  const onAdd = () => {
    // your existing cart logic here...

    trackEcomEvent("add_to_cart", {
      currency: "BDT",
      value: product.price,
      items: [
        {
          item_id: product._id,
          item_name: product.name,
          item_category: product.category || "Uncategorized",
          price: product.price,
          quantity: 1,
        },
      ],
    });
  };

  return <button onClick={onAdd}>Add to cart</button>;
}

```
#### C) begin_checkout when user proceeds to checkout
##### Put this on the Checkout button (from cart page/drawer):
```js
"use client";

import { trackEcomEvent } from "@/lib/datalayer";

export default function CheckoutButton({ cartItems }) {
  const onCheckout = () => {
    const totalValue = cartItems.reduce((sum, i) => sum + i.price * i.qty, 0);
    const totalQty = cartItems.reduce((sum, i) => sum + i.qty, 0);

    trackEcomEvent("begin_checkout", {
      currency: "BDT",
      value: totalValue,
      total_quantity: totalQty,
      items: cartItems.map((i) => ({
        item_id: i._id,
        item_name: i.name,
        item_category: i.category || "Uncategorized",
        price: i.price,
        quantity: i.qty,
      })),
    });

    // then route to checkout
    // router.push("/checkout");
  };

  return <button onClick={onCheckout}>Proceed to Checkout</button>;
}

```
#### D) purchase on success / thank you page
##### Important: fire purchase only after the order is confirmed (success page or after API confirms payment)
##### Example: app/checkout/success/page.js (client)
```js
"use client";

import { useEffect } from "react";
import { trackEcomEvent } from "@/lib/datalayer";

export default function SuccessPage({ searchParams }) {
  // Example: you can pass orderId in query params or fetch by sessionId
  // In App Router, you may fetch order details on server and pass as prop to a client component too.

  useEffect(() => {
    // Replace this with actual order data from your backend
    const order = {
      transaction_id: "ORD1767530045177533",
      value: 769,
      shipping: 59,
      discount: 0,
      currency: "BDT",
      total_quantity: 1,
      items: [
        {
          item_id: "6873f804b38361db35b4bb97",
          item_name: "KAO 8x4 Sarasara Switch Deodorant Spray (Rose & Verbena – Pink)",
          item_category: "Uncategorized",
          price: 710,
          quantity: 1,
        },
      ],
      customer: {
        name: "hashed_name_here",
        email: "hashed_email_here",
        phone: "hashed_phone_here",
        address: "hashed_address_here",
      },
    };

    trackEcomEvent(
      "purchase",
      {
        transaction_id: order.transaction_id,
        value: order.value,
        shipping: order.shipping,
        discount: order.discount,
        currency: order.currency,
        total_quantity: order.total_quantity,
        items: order.items,
      },
      { customer: order.customer }
    );
  }, []);

  return <div>Payment Success ✅</div>;
}

```