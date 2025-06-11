import React, { useState, useEffect } from "react";
import { db, collection, addDoc } from "../firebase";
import { loadStripe } from "@stripe/stripe-js";

const stripePromise = loadStripe("YOUR_STRIPE_PUBLIC_KEY");

export default function App() {
  const [products, setProducts] = useState([]);
  const [cart, setCart] = useState([]);
  const [search, setSearch] = useState("");
  const [categoryFilter, setCategoryFilter] = useState("");
  const [selectedOptions, setSelectedOptions] = useState({});

  useEffect(() => {
    async function fetchProducts() {
      try {
        const res = await fetch("https://fakestoreapi.com/products");
        const data = await res.json();
        const formatted = data.map((item) => ({
          id: item.id,
          name: item.title,
          price: item.price,
          image: item.image,
          category: item.category,
          options: ["Default", "Premium"],
        }));
        setProducts(formatted);
      } catch (err) {
        console.error("Failed to fetch products", err);
      }
    }
    fetchProducts();
  }, []);

  const addToCart = async (product) => {
    const option = selectedOptions[product.id] || product.options[0];
    const newItem = { ...product, selectedOption: option };
    setCart((prev) => [...prev, newItem]);

    try {
      await addDoc(collection(db, "cartItems"), newItem);
      console.log("Cart item added to Firestore");
    } catch (error) {
      console.error("Firebase add failed", error);
    }
  };

  const handleOptionChange = (productId, value) => {
    setSelectedOptions((prev) => ({ ...prev, [productId]: value }));
  };

  const filteredProducts = products.filter((product) => {
    return (
      product.name.toLowerCase().includes(search.toLowerCase()) &&
      (categoryFilter ? product.category === categoryFilter : true)
    );
  });

  const handleCheckout = async () => {
    const stripe = await stripePromise;
    const response = await fetch("/create-checkout-session", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ items: cart }),
    });
    const session = await response.json();
    const result = await stripe.redirectToCheckout({ sessionId: session.id });
    if (result.error) {
      alert(result.error.message);
    }
  };

  const fetchShopifyGraphQLProducts = async () => {
    const query = `{
      products(first: 5) {
        edges {
          node {
            title
            description
            variants(first: 1) {
              edges {
                node {
                  price
                }
              }
            }
          }
        }
      }
    }`;
    try {
      const res = await fetch("https://your-shopify-store.myshopify.com/api/graphql", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "X-Shopify-Storefront-Access-Token": "your-access-token",
        },
        body: JSON.stringify({ query }),
      });
      const data = await res.json();
      console.log("Shopify GraphQL Data:", data);
    } catch (error) {
      console.error("Failed to fetch from Shopify GraphQL", error);
    }
  };

  return (
    <div className="p-6 bg-gray-100 min-h-screen">
      <h1 className="text-4xl font-bold mb-6">🛍️ Shopify Furniture Store Clone</h1>

      <div className="mb-6 flex gap-4 flex-wrap">
        <input
          type="text"
          placeholder="Search furniture..."
          className="p-2 border border-gray-300 rounded-xl"
          value={search}
          onChange={(e) => setSearch(e.target.value)}
        />

        <select
          className="p-2 border border-gray-300 rounded-xl"
          value={categoryFilter}
          onChange={(e) => setCategoryFilter(e.target.value)}
        >
          <option value="">All Categories</option>
          {[...new Set(products.map((p) => p.category))].map((cat) => (
            <option key={cat} value={cat}>{cat}</option>
          ))}
        </select>

        <button
          className="px-4 py-2 bg-green-500 text-white rounded-xl"
          onClick={fetchShopifyGraphQLProducts}
        >
          📦 Load Shopify Products
        </button>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
        {filteredProducts.map((product) => (
          <div
            key={product.id}
            className="bg-white rounded-2xl shadow-md p-4 hover:shadow-xl transition"
          >
            <img
              src={product.image}
              alt={product.name}
              className="w-full h-40 object-cover rounded-lg"
            />
            <h2 className="text-xl font-semibold mt-2">{product.name}</h2>
            <p className="text-gray-600">${product.price}</p>
            <select
              className="mt-2 w-full border border-gray-300 rounded-xl p-1"
              value={selectedOptions[product.id] || product.options[0]}
              onChange={(e) => handleOptionChange(product.id, e.target.value)}
            >
              {product.options.map((opt, i) => (
                <option key={i} value={opt}>{opt}</option>
              ))}
            </select>
            <button
              className="mt-3 px-4 py-2 bg-blue-600 text-white rounded-xl hover:bg-blue-700 w-full"
              onClick={() => addToCart(product)}
            >
              Add to Cart
            </button>
          </div>
        ))}
      </div>

      <div className="mt-10 bg-white p-6 rounded-2xl shadow">
        <h2 className="text-2xl font-bold mb-2">🛒 Cart ({cart.length})</h2>
        {cart.length === 0 ? (
          <p className="text-gray-500">Cart is empty</p>
        ) : (
          <ul className="space-y-1">
            {cart.map((item, i) => (
              <li key={i} className="text-gray-700">
                • {item.name} ({item.selectedOption}) — ${item.price}
              </li>
            ))}
          </ul>
        )}
        {cart.length > 0 && (
          <div className="mt-4 text-right font-semibold space-y-2">
            <div>Total: ${cart.reduce((total, item) => total + item.price, 0).toFixed(2)}</div>
            <button
              onClick={handleCheckout}
              className="mt-2 px-4 py-2 bg-purple-600 text-white rounded-xl hover:bg-purple-700"
            >
              💳 Proceed to Checkout
            </button>
          </div>
        )}
      </div>
    </div>
  );
}
