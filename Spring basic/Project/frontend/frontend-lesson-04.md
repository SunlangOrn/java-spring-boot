# 🖥️ Frontend Lesson 4: Protected Routes & Checkout

---

## 🎯 Goal
- ✅ Update Axios to automatically attach the JWT token to every request.
- ✅ Protect frontend routes (redirect to login if the user isn't logged in).
- ✅ Build the Checkout page to call the Spring Boot Order API.

---

## 🧠 The Big Picture
In Lesson 1, we saved the JWT token in `localStorage`. But right now, our Axios client doesn't send it. 
We need to add an **Interceptor** to Axios. An interceptor is like a security guard that checks every outgoing request and says, "Wait, let me attach the user's ID badge (JWT) before you leave."

---

## 🛠️ Step 1: Add JWT Interceptor to Axios

### What we're doing:
Update `apiClient` so it automatically reads the token from `localStorage` and adds it to the `Authorization` header.

### The Code:
Update file: `src/lib/api.ts`

```typescript
import axios from 'axios';

const apiClient = axios.create({
  baseURL: 'http://localhost:8081',
  headers: {
    'Content-Type': 'application/json',
  },
});

// --- THE MAGIC INTERCEPTOR ---
apiClient.interceptors.request.use(
  (config) => {
    // Check if we are in the browser (localStorage doesn't exist on the server)
    if (typeof window !== 'undefined') {
      const token = localStorage.getItem('accessToken');
      
      // If token exists, attach it to the header
      if (token) {
        config.headers.Authorization = `Bearer ${token}`;
      }
    }
    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);

export default apiClient;
```

### 📝 After the Code - What Just Happened?
- `interceptors.request.use`: This runs *before* every single API call.
- `typeof window !== 'undefined'`: Next.js runs code on the server first. `localStorage` only exists in the browser. This check prevents the app from crashing during server-side rendering.
- Now, whenever we call `apiClient.post('/api/v1/orders/checkout')`, Axios automatically adds `Authorization: Bearer eyJhbG...` to the request!

---

## 🛠️ Step 2: Build the Checkout Page

### What we're doing:
Create a page that calls the backend checkout endpoint, clears the cart, and shows a success message.

### The Code:
Create file: `src/app/checkout/page.tsx`

```tsx
"use client";

import { useState } from 'react';
import { useCart } from '@/context/CartContext';
import apiClient from '@/lib/api';
import Link from 'next/link';

export default function CheckoutPage() {
  const { cartItems, totalAmount, clearCart } = useCart();
  const [loading, setLoading] = useState(false);
  const [success, setSuccess] = useState(false);
  const [error, setError] = useState('');

  const handleCheckout = async () => {
    setLoading(true);
    setError('');
    
    try {
      // Call the Spring Boot checkout API
      // Axios interceptor will automatically attach the JWT!
      await apiClient.post('/api/v1/orders/checkout');
      
      // If successful, clear the frontend cart
      clearCart();
      setSuccess(true);
    } catch (err: any) {
      setError(err.response?.data?.message || 'Checkout failed. Please try again.');
    } finally {
      setLoading(false);
    }
  };

  if (success) {
    return (
      <div className="hero min-h-screen bg-base-200">
        <div className="hero-content text-center">
          <div className="max-w-md">
            <h1 className="text-5xl font-bold text-success">Order Placed!</h1>
            <p className="py-6">Thank you for your purchase. Your order is being processed.</p>
            <Link href="/" className="btn btn-primary">Continue Shopping</Link>
          </div>
        </div>
      </div>
    );
  }

  return (
    <div className="container mx-auto px-4 py-8 max-w-2xl">
      <h1 className="text-3xl font-bold mb-6">Checkout</h1>

      <div className="card bg-base-100 shadow-xl">
        <div className="card-body">
          <h2 className="card-title">Order Summary</h2>
          <p>You have <span className="font-bold">{cartItems.length}</span> items in your cart.</p>
          <p className="text-2xl font-bold text-primary mt-4">Total: ${totalAmount.toFixed(2)}</p>

          {error && <div className="alert alert-error mt-4">{error}</div>}

          <div className="card-actions justify-end mt-6">
            <Link href="/cart" className="btn btn-ghost">Back to Cart</Link>
            <button 
              className="btn btn-primary"
              onClick={handleCheckout}
              disabled={loading || cartItems.length === 0}
            >
              {loading ? <span className="loading loading-spinner"></span> : 'Place Order'}
            </button>
          </div>
        </div>
      </div>
    </div>
  );
}
```

---

## 🛠️ Step 3: Protect the Cart and Checkout Routes

### What we're doing:
If a user goes to `/cart` or `/checkout` but isn't logged in, we should redirect them to `/login`. We do this using Next.js Middleware.

### The Code:
Create file: `src/middleware.ts` (Must be in the `src` root folder!)

```typescript
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  // Define protected routes
  const protectedPaths = ['/cart', '/checkout'];
  const { pathname } = request.nextUrl;

  // Check if the user is trying to access a protected route
  const isProtected = protectedPaths.some(path => pathname.startsWith(path));

  if (isProtected) {
    // Check if they have a token in their cookies or localStorage
    // Note: Middleware runs on the server, so we check cookies, not localStorage.
    // For simplicity in this lesson, we will just check if a specific cookie exists.
    // (In a real app, you'd use httpOnly cookies for the token).
    
    const token = request.cookies.get('accessToken')?.value;
    
    // If no token, redirect to login
    if (!token) {
      // Fallback: Since we used localStorage, we can't easily check it in middleware.
      // We will handle this via a simple client-side check in the next step.
    }
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/cart', '/checkout'],
};
```

*Wait, Next.js Middleware runs on the server, so it can't read `localStorage`!* 
Let's do a simpler **Client-Side Protection** instead.

### Alternative: Client-Side Route Protection
Update `src/app/cart/page.tsx` and `src/app/checkout/page.tsx` to add this at the very top of the component:

```tsx
import { useEffect } from 'react';
import { useRouter } from 'next/navigation';

export default function CartPage() {
  const router = useRouter();

  useEffect(() => {
    // Check if token exists in localStorage
    const token = localStorage.getItem('accessToken');
    if (!token) {
      // If no token, kick them to the login page
      router.push('/login');
    }
  }, [router]);

  // ... rest of the component
}
```

### 📝 After the Code - What Just Happened?
- `useRouter`: Next.js's way to programmatically change pages.
- `useEffect`: Runs once when the page loads. If there is no token, it instantly redirects the user to `/login`.

---

## 🧪 Run & Test

1. **Test Protected Route:** Open an Incognito/Private window. Go to `http://localhost:3000/cart`. You should instantly be redirected to `/login`!
2. **Test Checkout:** 
   - Log in in your normal window.
   - Add items to the cart.
   - Go to Checkout and click "Place Order".
   - Check pgAdmin! The `orders` table should have a new row, the `order_items` table should have the products, and the `products` table should have reduced stock!

---

## 🎉 FRONTEND INTEGRATION COMPLETE! 🎉

You have successfully connected your Next.js frontend to your Spring Boot backend!

**You now have a fully functioning E-Commerce App:**
✅ Beautiful UI with DaisyUI.
✅ Secure Login/Register with JWT.
✅ Paginated Product Catalog.
✅ Global Shopping Cart.
✅ Transactional Checkout.

---

## 🛑 STOP
Reply with **"Frontend Complete"** when you have successfully placed an order and seen the stock reduce in pgAdmin! 

Let me know what you want to conquer next: **Dockerizing the whole app**, **Deploying to the cloud**, or **Microservices**! 🚀