import React, { useEffect, useState } from "react";

/*
  XARO - Option B: Professional Full Store (Single-file React demo)
  - Local product DB (products.json) with optional Firebase switch
  - Cart system (add/remove/update)
  - Checkout that posts orders to Google Sheets via a Webhook (Google Apps Script)
  - Admin dashboard (protected by a simple passphrase) that fetches orders from the same webhook
  - Simple stock management (decrements stock on order) and order id generation
  - UPI / Google Pay intent + QR (static placeholder)

  HOW TO USE / DEPLOY:
  1) Create a Google Sheet and an Apps Script that accepts POST requests and writes rows.
    - Create a Web App (Deploy -> New deployment -> Web app) and allow Anyone, even anonymous to execute (or restrict as needed)
    - Copy the Web App URL and paste it into GOOGLE_SHEET_WEBHOOK_URL below.
    - You can use this Apps Script sample: https://developers.google.com/apps-script/guides/web

  2) (Optional) If you want product DB in Firebase, set USE_FIREBASE=true and add your Firebase config below.
     This demo includes a simple local fallback (products array) so you can run it immediately.

  3) Deploy to Vercel / Netlify or host locally with create-react-app.

  IMPORTANT: This file is a demo and does not include production security. Use proper auth for admin endpoints.
*/

// ---------- CONFIG ----------
const GOOGLE_SHEET_WEBHOOK_URL = "https://script.google.com/macros/s/AKfycbzU5v_sLJQlEr7ytiSvUuiflm311bxfwujf3Z-nOJLfxy6FUAbRqpXJ5IRpos3TF2IN/exec"; // <- Replace with your Apps Script WebApp URL
const ADMIN_SECRET = "xaro-admin"; // change this before deploying
const USE_FIREBASE = false; // set true if you add firebaseConfig

// If you enable Firebase, paste your config here and install firebase packages in your project.
const firebaseConfig = {
  apiKey: "",
  authDomain: "",
  projectId: "",
  storageBucket: "",
  messagingSenderId: "",
  appId: "",
};

// ---------- DEMO PRODUCTS (local fallback) ----------
const initialProducts = [
  { id: "shoe-01", title: "Campus Runner", type: "Shoe", price: 599, stock: 50, img: "https://images.unsplash.com/photo-1603791440384-56cd371ee9a7?w=800&q=80" },
  { id: "jersey-01", title: "Home Football Jersey", type: "Jersey", price: 499, stock: 80, img: "https://images.unsplash.com/photo-1519741490212-89f59f3b8b9c?w=800&q=80" },
  { id: "slide-01", title: "Street Slide", type: "Slide", price: 299, stock: 120, img: "https://images.unsplash.com/photo-1585238342028-4e9f6b0e0f6f?w=800&q=80" },
  { id: "jersey-02", title: "Basket Pro Jersey", type: "Jersey", price: 549, stock: 40, img: "https://images.unsplash.com/photo-1600400352001-3b4a7f6fd2a0?w=800&q=80" },
];

export default function XaroWebsite() {
  const [products, setProducts] = useState(initialProducts);
  const [cart, setCart] = useState([]);
  const [query, setQuery] = useState("");
  const [view, setView] = useState("shop"); // shop, cart, checkout, admin
  const [orders, setOrders] = useState([]);
  const [adminKey, setAdminKey] = useState("");
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    // If USE_FIREBASE is true you would fetch products from Firebase here.
    // For now the local initialProducts are used.
  }, []);

  // Cart helpers
  function addToCart(product, qty = 1) {
    setCart((prev) => {
      const existing = prev.find((p) => p.id === product.id);
      if (existing) return prev.map((p) => (p.id === product.id ? { ...p, qty: p.qty + qty } : p));
      return [...prev, { ...product, qty }];
    });
  }

  function updateQty(id, qty) {
    if (qty < 1) return;
    setCart((prev) => prev.map((p) => (p.id === id ? { ...p, qty } : p)));
  }

  function removeFromCart(id) {
    setCart((prev) => prev.filter((p) => p.id !== id));
  }

  function cartTotal() {
    return cart.reduce((s, p) => s + p.price * p.qty, 0);
  }

  function generateOrderId() {
    return `XARO-${Date.now().toString().slice(-6)}`;
  }

  // Checkout -> sends order to Google Sheets webhook
  async function handleCheckout(customer) {
    if (!GOOGLE_SHEET_WEBHOOK_URL || GOOGLE_SHEET_WEBHOOK_URL.includes("XXXXXXXX")) {
      alert("Please set your Google Sheet webhook URL in the code before checking out.");
      return;
    }
    if (cart.length === 0) {
      alert("Your cart is empty.");
      return;
    }

    setLoading(true);
    const order = {
      id: generateOrderId(),
      createdAt: new Date().toISOString(),
      customer: customer,
      items: cart.map((c) => ({ id: c.id, title: c.title, qty: c.qty, price: c.price })),
      total: cartTotal(),
      status: "Pending",
    };

    try {
      // Post to Google Apps Script (sheet webhook)
      const resp = await fetch(GOOGLE_SHEET_WEBHOOK_URL, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(order),
      });

      if (!resp.ok) throw new Error("Webhook request failed");

      // Decrement stock locally (in production, do this on server-side)
      setProducts((prev) => prev.map((p) => {
        const ordered = cart.find((c) => c.id === p.id);
        if (!ordered) return p;
        return { ...p, stock: Math.max(0, p.stock - ordered.qty) };
      }));

      // Clear cart
      setCart([]);
      alert(`Order placed! Order ID: ${order.id}`);
      setView("shop");
    } catch (err) {
      console.error(err);
      alert("Order failed. Check console and webhook URL.");
    } finally {
      setLoading(false);
    }
  }

  // Admin fetch orders (GET)
  async function fetchOrders() {
    if (!GOOGLE_SHEET_WEBHOOK_URL || GOOGLE_SHEET_WEBHOOK_URL.includes("XXXXXXXX")) {
      alert("Please set your Google Sheet webhook URL in the code before fetching orders.");
      return;
    }
    setLoading(true);
    try {
      // Many Apps Script setups support GET returning JSON. Replace with your admin endpoint if different.
      const resp = await fetch(`${GOOGLE_SHEET_WEBHOOK_URL}?admin=true`);
      if (!resp.ok) throw new Error("Failed to fetch orders");
      const data = await resp.json();
      setOrders(data.orders || []);
    } catch (err) {
      console.error(err);
      alert("Failed to fetch orders. Check Apps Script implementation.");
    } finally {
      setLoading(false);
    }
  }

  // ---------- Small UI components ----------
  function Header() {
    return (
      <header className="flex items-center justify-between px-6 py-4 border-b border-neutral-800 sticky top-0 bg-black/80 z-50 backdrop-blur">
        <div className="flex items-center gap-4">
          <div className="w-10 h-10 rounded-full bg-white text-black flex items-center justify-center font-bold">X</div>
          <div>
            <div className="text-xl font-bold">XARO</div>
            <div className="text-sm text-neutral-400">Budget Shoes & Jerseys</div>
          </div>
        </div>
        <nav className="flex items-center gap-4">
          <button onClick={() => setView("shop")} className="text-sm hover:underline">Shop</button>
          <button onClick={() => setView("cart")} className="text-sm hover:underline">Cart ({cart.length})</button>
          <button onClick={() => setView("checkout")} className="text-sm hover:underline">Checkout</button>
          <button onClick={() => setView("admin")} className="text-sm hover:underline">Admin</button>
        </nav>
      </header>
    );
  }

  function ShopView() {
    const filtered = products.filter(p => p.title.toLowerCase().includes(query.toLowerCase()) || p.type.toLowerCase().includes(query.toLowerCase()));
    return (
      <main className="p-6 max-w-6xl mx-auto">
        <section className="mb-8 text-center">
          <h1 className="text-4xl font-bold">XARO — Affordable Shoes & Jerseys</h1>
          <p className="text-neutral-400 mt-2">Products from ₹299 • COD & Pan-India delivery</p>
          <div className="mt-4 flex justify-center gap-3">
            <input value={query} onChange={(e) => setQuery(e.target.value)} placeholder="Search" className="px-3 py-2 rounded bg-neutral-900" />
            <button onClick={() => setQuery("")} className="px-3 py-2 rounded border">Clear</button>
          </div>
        </section>

        <section className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6">
          {filtered.map(p => (
            <div key={p.id} className="bg-neutral-900 rounded-xl overflow-hidden shadow">
              <img src={p.img} alt={p.title} className="w-full h-56 object-cover" />
              <div className="p-4">
                <div className="flex items-center justify-between">
                  <h3 className="font-semibold text-lg">{p.title}</h3>
                  <div className="text-sm text-neutral-400">Stock: {p.stock}</div>
                </div>
                <p className="text-neutral-400 mt-1">{p.type}</p>
                <div className="mt-3 flex items-center justify-between">
                  <div className="text-xl font-bold">₹{p.price}</div>
                  <div className="flex gap-2">
                    <button className="px-3 py-1 rounded border" onClick={() => addToCart(p)}>Add</button>
                    <button className="px-3 py-1 rounded bg-white text-black" onClick={() => addToCart(p)}>Buy</button>
                  </div>
                </div>
              </div>
            </div>
          ))}
        </section>
      </main>
    );
  }

  function CartView() {
    return (
      <main className="p-6 max-w-4xl mx-auto">
        <h2 className="text-2xl font-bold mb-4">Your Cart</h2>
        {cart.length === 0 ? (
          <div className="text-neutral-400">Cart is empty. Add items from Shop.</div>
        ) : (
          <div className="space-y-4">
            {cart.map(c => (
              <div key={c.id} className="flex items-center justify-between bg-neutral-900 p-3 rounded">
                <div className="flex items-center gap-3">
                  <img src={c.img} alt={c.title} className="w-16 h-16 object-cover rounded" />
                  <div>
                    <div className="font-semibold">{c.title}</div>
                    <div className="text-sm text-neutral-400">₹{c.price}</div>
                  </div>
                </div>
                <div className="flex items-center gap-2">
                  <input type="number" min={1} value={c.qty} onChange={(e) => updateQty(c.id, Number(e.target.value))} className="w-20 px-2 py-1 rounded bg-neutral-800" />
                  <div className="font-semibold">₹{c.price * c.qty}</div>
                  <button onClick={() => removeFromCart(c.id)} className="text-red-400">Remove</button>
                </div>
              </div>
            ))}

            <div className="flex items-center justify-between bg-neutral-800 p-4 rounded">
              <div className="font-semibold">Total</div>
              <div className="text-xl font-bold">₹{cartTotal()}</div>
            </div>

            <div className="flex gap-3">
              <button onClick={() => setView("checkout")} className="flex-1 py-2 bg-green-600 rounded">Proceed to Checkout</button>
              <button onClick={() => setCart([])} className="flex-1 py-2 border rounded">Clear Cart</button>
            </div>
          </div>
        )}
      </main>
    );
  }

  function CheckoutView() {
    const [name, setName] = useState("");
    const [phone, setPhone] = useState("");
    const [address, setAddress] = useState("");
    const [note, setNote] = useState("");

    return (
      <main className="p-6 max-w-3xl mx-auto">
        <h2 className="text-2xl font-bold mb-4">Checkout</h2>
        {cart.length === 0 ? (
          <div className="text-neutral-400">Your cart is empty.</div>
        ) : (
          <form onSubmit={(e) => { e.preventDefault(); handleCheckout({ name, phone, address, note }); }} className="space-y-4">
            <input value={name} onChange={(e) => setName(e.target.value)} required placeholder="Full name" className="w-full px-3 py-2 rounded bg-neutral-900" />
            <input value={phone} onChange={(e) => setPhone(e.target.value)} required placeholder="Phone" className="w-full px-3 py-2 rounded bg-neutral-900" />
            <textarea value={address} onChange={(e) => setAddress(e.target.value)} required placeholder="Delivery address" className="w-full px-3 py-2 rounded bg-neutral-900" />
            <input value={note} onChange={(e) => setNote(e.target.value)} placeholder="Order note (optional)" className="w-full px-3 py-2 rounded bg-neutral-900" />

            <div className="bg-neutral-900 p-4 rounded">
              <div className="flex items-center justify-between mb-2"><div>Subtotal</div><div>₹{cartTotal()}</div></div>
              <div className="flex items-center justify-between mb-2"><div>Shipping</div><div>₹50</div></div>
              <div className="flex items-center justify-between font-bold"><div>Total</div><div>₹{cartTotal() + 50}</div></div>
            </div>

            <div className="grid grid-cols-1 sm:grid-cols-2 gap-3">
              <button type="submit" disabled={loading} className="py-2 bg-white text-black rounded">Place Order (COD)</button>
              <button type="button" onClick={() => alert('Google Pay / UPI flow: open UPI intent or show QR. This is a demo.')} className="py-2 border rounded">Pay with UPI / GPay</button>
            </div>
          </form>
        )}
      </main>
    );
  }

  function AdminView() {
    const [secret, setSecret] = useState("");
    return (
      <main className="p-6 max-w-5xl mx-auto">
        <h2 className="text-2xl font-bold mb-4">Admin Dashboard</h2>
        {!adminKey || adminKey !== ADMIN_SECRET ? (
          <div className="bg-neutral-900 p-6 rounded">
            <p className="mb-3">Enter admin secret to view orders.</p>
            <input type="password" value={adminKey} onChange={(e) => setAdminKey(e.target.value)} className="px-3 py-2 rounded bg-neutral-800" />
            <div className="mt-3 flex gap-3">
              <button onClick={() => { if (adminKey === ADMIN_SECRET) { fetchOrders(); } else alert('Wrong secret'); }} className="py-2 px-4 bg-white text-black rounded">Unlock</button>
              <button onClick={() => { setAdminKey(ADMIN_SECRET); fetchOrders(); }} className="py-2 px-4 border rounded">Autofill (demo)</button>
            </div>
          </div>
        ) : (
          <div className="space-y-4">
            <div className="flex items-center justify-between">
              <div className="text-neutral-400">Fetched Orders: {orders.length}</div>
              <div>
                <button onClick={fetchOrders} className="py-1 px-3 border rounded">Refresh</button>
              </div>
            </div>

            <div className="bg-neutral-900 p-4 rounded">
              {loading ? <div>Loading…</div> : (
                orders.length === 0 ? <div className="text-neutral-400">No orders fetched yet.</div> : (
                  <div className="space-y-3">
                    {orders.map(o => (
                      <div key={o.id} className="p-3 bg-neutral-800 rounded">
                        <div className="flex items-center justify-between">
                          <div>
                            <div className="font-semibold">{o.id} — {o.customer?.name}</div>
                            <div className="text-sm text-neutral-400">{o.createdAt} • ₹{o.total}</div>
                          </div>
                          <div className="text-sm">{o.status}</div>
                        </div>
                        <div className="text-neutral-400 mt-2">
                          {o.items?.map(it => (<div key={it.id}>{it.title} x{it.qty} — ₹{it.price}</div>))}
                        </div>
                      </div>
                    ))}
                  </div>
                )
              )}
            </div>
          </div>
        )}
      </main>
    );
  }

  return (
    <div className="min-h-screen bg-black text-white">
      <Header />

      {view === "shop" && <ShopView />}
      {view === "cart" && <CartView />}
      {view === "checkout" && <CheckoutView />}
      {view === "admin" && <AdminView />}

      <footer className="py-6 text-center text-neutral-500">© {new Date().getFullYear()} XARO — Managed by Naithik • Aatmay • Karthik • Sreehari • Nikhil</footer>
    </div>
  );
}
