# 🖥️ Frontend Lesson 3: Shopping Cart & Global State

---

## 🎯 Goal
- ✅ Create a global "Cart" using React Context.
- ✅ Allow users to add/remove items from the cart from any page.
- ✅ Build a Cart page to view the items.

---

## 🧠 The Big Picture
Right now, if we add an item to the cart on the Home page, the Cart page doesn't know about it. 
We need a **Global State**. Think of React Context like a public bulletin board. The Home page pins a note: "Added 1 Laptop". The Cart page looks at the board and sees the note.

---

## 🛠️ Step 1: Create the Cart Context

### What we're doing:
Create the "bulletin board" that holds the cart items and the functions to modify them.

### The Code:
Create file: `src/context/CartContext.tsx`

```tsx
"use client";

import { createContext, useState, useContext, ReactNode } from 'react';

interface CartItem {
  id: number;
  name: string;
  price: number;
  quantity: number;
}

interface CartContextType {
  cartItems: CartItem[];
  addToCart: (product: any) => void;
  removeFromCart: (productId: number) => void;
  clearCart: () => void;
  totalAmount: number;
}

// 1. Create the Context
const CartContext = createContext<CartContextType | undefined>(undefined);

// 2. Create the Provider (The actual bulletin board)
export function CartProvider({ children }: { children: ReactNode }) {
  const [cartItems, setCartItems] = useState<CartItem[]>([]);

  const addToCart = (product: any) => {
    setCartItems((prevItems) => {
      const existingItem = prevItems.find((item) => item.id === product.id);
      if (existingItem) {
        // If exists, increase quantity
        return prevItems.map((item) =>
          item.id === product.id ? { ...item, quantity: item.quantity + 1 } : item
        );
      }
      // If new, add to array
      return [...prevItems, { id: product.id, name: product.name, price: product.price, quantity: 1 }];
    });
  };

  const removeFromCart = (productId: number) => {
    setCartItems((prevItems) => prevItems.filter((item) => item.id !== productId));
  };

  const clearCart = () => setCartItems([]);

  const totalAmount = cartItems.reduce((sum, item) => sum + item.price * item.quantity, 0);

  return (
    <CartContext.Provider value={{ cartItems, addToCart, removeFromCart, clearCart, totalAmount }}>
      {children}
    </CartContext.Provider>
  );
}

// 3. Create a custom hook to easily use the cart anywhere
export function useCart() {
  const context = useContext(CartContext);
  if (!context) {
    throw new Error('useCart must be used within a CartProvider');
  }
  return context;
}
```

### 📝 After the Code - What Just Happened?
- `createContext`: Creates the empty bulletin board.
- `CartProvider`: The component that actually holds the state (`cartItems`) and the functions (`addToCart`).
- `useCart`: A custom hook. Instead of writing `useContext(CartContext)` everywhere, we just write `const { addToCart } = useCart()`.

---

## 🛠️ Step 2: Wrap the App in the Provider

### What we're doing:
Make the Cart available to the entire application.

### The Code:
Update file: `src/app/layout.tsx`

```tsx
import type { Metadata } from "next";
import { Inter } from "next/font/google";
import "./globals.css";
import { CartProvider } from "@/context/CartContext"; // Import it

const inter = Inter({ subsets: ["latin"] });

export const metadata: Metadata = {
  title: "ShopCore",
  description: "Enterprise E-Commerce",
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body className={inter.className}>
        {/* Wrap the children with CartProvider */}
        <CartProvider>
          {children}
        </CartProvider>
      </body>
    </html>
  );
}
```

---

## 🛠️ Step 3: Connect the Cart to the Product Card

### What we're doing:
Update the Home page to use the real `addToCart` function instead of the `alert`.

### The Code:
Update `src/app/page.tsx`:

```tsx
// Add this import at the top:
import { useCart } from '@/context/CartContext';

// Inside the HomePage component:
export default function HomePage() {
  const { addToCart } = useCart(); // Get the function from the Context

  // ... (keep all the existing state and useEffect)

  // Update the handleAddToCart function:
  const handleAddToCart = (product: Product) => {
    addToCart(product); // This updates the global state!
  };

  // ... (rest of the code remains the same)
}
```

---

## 🛠️ Step 4: Build the Cart Page

### What we're doing:
Create a page that reads from the Cart Context and displays the items.

### The Code:
Create file: `src/app/cart/page.tsx`

```tsx
"use client";

import { useCart } from '@/context/CartContext';
import Link from 'next/link';

export default function CartPage() {
  const { cartItems, removeFromCart, totalAmount } = useCart();

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-6">Your Shopping Cart</h1>

      {cartItems.length === 0 ? (
        <div className="text-center py-20">
          <p className="text-xl mb-4">Your cart is empty.</p>
          <Link href="/" className="btn btn-primary">Continue Shopping</Link>
        </div>
      ) : (
        <div className="overflow-x-auto">
          <table className="table table-zebra w-full">
            <thead>
              <tr>
                <th>Product</th>
                <th>Price</th>
                <th>Quantity</th>
                <th>Total</th>
                <th>Action</th>
              </tr>
            </thead>
            <tbody>
              {cartItems.map((item) => (
                <tr key={item.id}>
                  <td>{item.name}</td>
                  <td>${item.price.toFixed(2)}</td>
                  <td>{item.quantity}</td>
                  <td>${(item.price * item.quantity).toFixed(2)}</td>
                  <td>
                    <button 
                      className="btn btn-error btn-xs"
                      onClick={() => removeFromCart(item.id)}
                    >
                      Remove
                    </button>
                  </td>
                </tr>
              ))}
            </tbody>
          </table>

          <div className="flex justify-end mt-6">
            <div className="text-xl font-bold">
              Grand Total: <span className="text-primary">${totalAmount.toFixed(2)}</span>
            </div>
          </div>
          
          <div className="flex justify-end mt-4">
            <Link href="/checkout" className="btn btn-primary btn-lg">
              Proceed to Checkout
            </Link>
          </div>
        </div>
      )}
    </div>
  );
}
```

---

## 🧪 Run & Test

1. Go to the Home page. Click "Add to Cart" on a few products.
2. Go to **http://localhost:3000/cart**.
3. You should see the items in a beautiful table!
4. Click "Remove" and watch the item disappear and the total update instantly.

---

## 🛑 STOP
Reply with your confirmation before moving to Lesson 4.