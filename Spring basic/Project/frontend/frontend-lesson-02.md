# 🖥️ Frontend Lesson 2: Product Catalog & Pagination

---

## 🎯 Goal
- ✅ Fetch paginated products from the Spring Boot API.
- ✅ Display products in a beautiful, responsive grid using DaisyUI Cards.
- ✅ Add "Previous" and "Next" buttons for pagination.

---

## 🧠 The Big Picture
Right now, our backend has hundreds of products. If we load them all at once, the browser will freeze. 
Instead, we will ask the backend for "Page 0, Size 8". The backend returns 8 products, plus metadata (like `totalPages`). We will display the 8 products in a grid, and use buttons to load the next page.

---

## 📖 Key Words

| Word | Simple Meaning |
|------|---------------|
| **Grid** | A 2D layout (rows and columns) to display items neatly. |
| **Card** | A UI component that groups related information (image, title, price, button) into a neat box. |
| **Pagination** | Splitting a large list of data into smaller "pages". |

---

## 🛠️ Step 1: Create the Product Card Component

### What we're doing:
Create a reusable "Card" component for a single product so we don't have to rewrite the HTML for every product.

### The Code:
Create file: `src/components/ProductCard.tsx`

```tsx
"use client";

import Image from 'next/image'; // Next.js built-in image optimization

interface Product {
  id: number;
  name: string;
  price: number;
  description: string;
  category: { name: string };
}

interface ProductCardProps {
  product: Product;
  onAddToCart: (product: Product) => void;
}

export default function ProductCard({ product, onAddToCart }: ProductCardProps) {
  return (
    <div className="card bg-base-100 shadow-xl image-full hover:scale-105 transition-transform duration-200">
      <figure>
        {/* Using a placeholder image since we don't have real images yet */}
        <img src="https://placehold.co/400x200?text=Product" alt={product.name} />
      </figure>
      <div className="card-body">
        <h2 className="card-title">{product.name}</h2>
        <p className="text-sm opacity-80">{product.category.name}</p>
        <p className="text-lg font-bold">${product.price.toFixed(2)}</p>
        
        <div className="card-actions justify-end mt-2">
          {/* daisyui btn-primary and btn-outline */}
          <button 
            className="btn btn-primary btn-sm"
            onClick={() => onAddToCart(product)}
          >
            Add to Cart
          </button>
        </div>
      </div>
    </div>
  );
}
```

### 📝 After the Code - What Just Happened?
- `interface Product`: Defines the exact shape of the JSON we get from the backend.
- `image-full`: A DaisyUI class that makes the image take up the whole top of the card.
- `hover:scale-105`: A Tailwind class that makes the card grow slightly when the user hovers over it.
- `onAddToCart`: We pass a function down from the parent page so the card can tell the parent "Hey, add this product to the cart!"

---

## 🛠️ Step 2: Build the Home Page with Pagination

### What we're doing:
Fetch the products from the backend, map them to `ProductCard` components, and add pagination buttons.

### The Code:
Update file: `src/app/page.tsx` (This is the homepage)

```tsx
"use client";

import { useState, useEffect } from 'react';
import apiClient from '@/lib/api';
import ProductCard from '@/components/ProductCard';

interface Category { id: number; name: string; }
interface Product { id: number; name: string; price: number; description: string; category: Category; }

export default function HomePage() {
  const [products, setProducts] = useState<Product[]>([]);
  const [currentPage, setCurrentPage] = useState(0);
  const [totalPages, setTotalPages] = useState(0);
  const [loading, setLoading] = useState(true);

  // Fetch products whenever currentPage changes
  useEffect(() => {
    const fetchProducts = async () => {
      setLoading(true);
      try {
        // Ask backend for 8 items per page
        const response = await apiClient.get(`/api/v1/products/paged?page=${currentPage}&size=8`);
        
        // Extract data from our HttpBodyResponse wrapper
        const pageData = response.data.data; 
        setProducts(pageData.items);
        setTotalPages(pageData.totalPages);
      } catch (error) {
        console.error("Failed to fetch products", error);
      } finally {
        setLoading(false);
      }
    };

    fetchProducts();
  }, [currentPage]); // Re-run this effect when currentPage changes

  const handleAddToCart = (product: Product) => {
    alert(`Added ${product.name} to cart! (We will build the real cart in Lesson 3)`);
  };

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-4xl font-bold text-center mb-8">Welcome to ShopCore</h1>

      {loading ? (
        <div className="flex justify-center my-20">
          <span className="loading loading-spinner loading-lg text-primary"></span>
        </div>
      ) : (
        <>
          {/* Grid Layout: 1 col on mobile, 2 on md, 4 on xl */}
          <div className="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-4 gap-6">
            {products.map((product) => (
              <ProductCard 
                key={product.id} 
                product={product} 
                onAddToCart={handleAddToCart} 
              />
            ))}
          </div>

          {/* Pagination Controls */}
          <div className="flex justify-center mt-10 gap-4">
            <button 
              className="btn btn-outline"
              disabled={currentPage === 0}
              onClick={() => setCurrentPage(prev => prev - 1)}
            >
              Previous
            </button>
            
            <span className="btn btn-ghost no-animation">
              Page {currentPage + 1} of {totalPages}
            </span>

            <button 
              className="btn btn-outline"
              disabled={currentPage >= totalPages - 1}
              onClick={() => setCurrentPage(prev => prev + 1)}
            >
              Next
            </button>
          </div>
        </>
      )}
    </div>
  );
}
```

### 📝 After the Code - What Just Happened?
- `useEffect`: A React hook that runs code when the component loads, or when a specific variable (`currentPage`) changes.
- `grid grid-cols-1 md:grid-cols-2 xl:grid-cols-4`: Tailwind magic. On a phone, it shows 1 column. On a tablet, 2. On a big screen, 4.
- `loading loading-spinner`: DaisyUI's built-in loading spinner.

---

## 🧪 Run & Test

1. Ensure Spring Boot is running and has at least 10 products in the database.
2. Go to **http://localhost:3000**.
3. You should see 8 products in a grid.
4. Click "Next". The spinner appears, and the next page of products loads!

---

## ✏️ Exercise
1. Change the `size=8` parameter to `size=12`.
2. Add a search bar at the top of the page that filters the products by name (Hint: update the API URL to `/api/v1/products/search?name=...`).

---

## 🛑 STOP
Reply with your exercise confirmation before moving to Lesson 3.