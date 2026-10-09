# How to Use Firebase — Loyalty ICT Solutions

This guide turns your demo into a real, multi-device business system.
Follow it top to bottom. Every code block is copy-paste ready.

**What Firebase gives you here:**
1. **Firestore** — one shared database, so a customer's order on their phone shows up on your admin phone instantly.
2. **Authentication** — real, secure staff logins. No more password sitting in the code.
3. **Storage** — a proper home for product photos, instead of squeezing them into the browser.

Time needed: about 1–2 hours the first time.

---

## Part 1 — Create your Firebase project

1. Go to **console.firebase.google.com** and sign in with a Google account.
2. Click **Add project**. Name it `loyalty-ict`. Turn off Google Analytics (you don't need it yet). Click **Create project**.
3. On the project home, click the **web icon** `</>` to add a web app. Nickname it `loyalty-web`. **Do not** tick "Firebase Hosting" (you're using Netlify). Click **Register app**.
4. Firebase shows you a **config block**. Copy it. It looks like this:

```js
const firebaseConfig = {
  apiKey: "AIza....",
  authDomain: "loyalty-ict.firebaseapp.com",
  projectId: "loyalty-ict",
  storageBucket: "loyalty-ict.appspot.com",
  messagingSenderId: "12345",
  appId: "1:12345:web:abcd"
};
```

Keep this safe — you'll paste it in Part 3.

> **Note on the apiKey:** this key is *meant* to be public. It only identifies your project. What actually protects your data is the **security rules** in Part 5 — not hiding this key. This is a common misunderstanding, and as a security person it's worth knowing.

---

## Part 2 — Turn on the three services

In the Firebase console left menu:

**Authentication**
1. Click **Authentication → Get started**.
2. Under **Sign-in method**, enable **Email/Password**. Save.
3. Go to the **Users** tab → **Add user**. Create your admin account, e.g. `admin@loyaltyict.co.ke` with a strong password. This replaces `loyalty2025`.

**Firestore Database**
1. Click **Firestore Database → Create database**.
2. Choose **Production mode** (we'll set proper rules in Part 5).
3. Pick a location close to Kenya — `europe-west1` is a reasonable low-latency choice. Click **Enable**.

**Storage**
1. Click **Storage → Get started**.
2. Accept the default, choose the same location. This holds product images.

---

## Part 3 — Add Firebase to your app

Open `index.html`. Find the closing `</script>` tag near the very bottom (right before `</body>`). **Directly above `</body>`**, add this new module script. Paste your real config from Part 1.

```html
<!-- ============ FIREBASE ============ -->
<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/11.0.2/firebase-app.js";
import { getFirestore, collection, getDocs, getDoc, addDoc, doc,
         updateDoc, deleteDoc, onSnapshot, query, orderBy, setDoc }
  from "https://www.gstatic.com/firebasejs/11.0.2/firebase-firestore.js";
import { getAuth, createUserWithEmailAndPassword, signInWithEmailAndPassword,
         signOut, onAuthStateChanged }
  from "https://www.gstatic.com/firebasejs/11.0.2/firebase-auth.js";
import { getStorage, ref as sRef, uploadBytes, getDownloadURL }
  from "https://www.gstatic.com/firebasejs/11.0.2/firebase-storage.js";

// >>> PASTE YOUR CONFIG HERE <<<
const firebaseConfig = {
  apiKey: "PASTE",
  authDomain: "PASTE",
  projectId: "PASTE",
  storageBucket: "PASTE",
  messagingSenderId: "PASTE",
  appId: "PASTE"
};

const app  = initializeApp(firebaseConfig);
const fdb  = getFirestore(app);
const auth = getAuth(app);
const storage = getStorage(app);

// expose to the rest of the (non-module) app
window.FB = { fdb, auth, storage, collection, getDocs, getDoc, addDoc, doc,
              updateDoc, deleteDoc, onSnapshot, query, orderBy, setDoc,
              createUserWithEmailAndPassword, signInWithEmailAndPassword,
              signOut, onAuthStateChanged,
              sRef, uploadBytes, getDownloadURL };
window.dispatchEvent(new Event('firebase-ready'));
</script>
```

Why `window.FB`? Your app's code is a normal script; Firebase v11 is a module. This line hands the Firebase tools to your existing code cleanly.

---

## Part 4 — Replace the data layer

This is the core change. Your app already routes everything through `DB.get()` and `DB.set()`, so you only edit **one small block** plus the boot lines.

### 4a. Swap the demo DB with a live one

Find this near the top of your main `<script>`:

```js
const DB = {
  get(key,fallback){ ... localStorage ... },
  set(key,val){ ... localStorage ... }
};
```

Replace the whole `DB` object with this. It keeps an **in-memory cache** so all your existing synchronous code still works, while Firebase syncs it live across devices:

```js
// Live data layer. cache[] mirrors Firestore in real time.
const cache = { products:[], orders:[], bookings:[] };
const DB = {
  get(key, fallback){ return cache[key] ?? fallback; },
  // set() writes the WHOLE collection is avoided — we write per-document
  // instead (see helpers below), which is how Firestore is meant to work.
  set(){ /* no-op: writes now go through the FB helpers below */ }
};

// Called once Firebase is ready. Streams each collection into cache
// and re-renders the screen whenever anything changes on ANY device.
function startFirestore(){
  const { fdb, collection, onSnapshot, query, orderBy } = window.FB;

  onSnapshot(collection(fdb,'products'), snap => {
    cache.products = snap.docs.map(d => ({ id:d.id, ...d.data() }));
    if(document.getElementById('productGrid')) renderProducts();
    if(!document.getElementById('admin-view').classList.contains('hidden')) renderAdmin();
  });
  onSnapshot(query(collection(fdb,'orders'), orderBy('date','desc')), snap => {
    cache.orders = snap.docs.map(d => ({ id:d.id, ...d.data() }));
    if(!document.getElementById('admin-view').classList.contains('hidden')) renderAdmin();
  });
  onSnapshot(query(collection(fdb,'bookings'), orderBy('created','desc')), snap => {
    cache.bookings = snap.docs.map(d => ({ id:d.id, ...d.data() }));
    if(!document.getElementById('admin-view').classList.contains('hidden')) renderAdmin();
  });
}

// ---- Firestore write helpers (use these instead of DB.set) ----
async function fbAdd(col, data){
  const { fdb, addDoc, collection } = window.FB;
  return (await addDoc(collection(fdb, col), data)).id;
}
async function fbUpdate(col, id, data){
  const { fdb, updateDoc, doc } = window.FB;
  await updateDoc(doc(fdb, col, id), data);
}
async function fbDelete(col, id){
  const { fdb, deleteDoc, doc } = window.FB;
  await deleteDoc(doc(fdb, col, id));
}
```

### 4b. Point the boot lines at Firebase

At the very bottom of your script you have:

```js
renderCats();renderProducts();updateCartUI();
if('serviceWorker' in navigator){navigator.serviceWorker.register('sw.js').catch(()=>{});}
```

Change it to:

```js
renderCats();updateCartUI();
window.addEventListener('firebase-ready', () => {
  startFirestore();
  seedIfEmpty();   // uploads the 20 sample products the first time only
});
if('serviceWorker' in navigator){navigator.serviceWorker.register('sw.js').catch(()=>{});}

// One-time seed so your store isn't empty on first launch.
async function seedIfEmpty(){
  const { fdb, getDocs, collection } = window.FB;
  const snap = await getDocs(collection(fdb,'products'));
  if(snap.empty){ for(const p of SEED_PRODUCTS){ const {id,...rest}=p; await fbAdd('products', rest); } }
}
```

### 4c. Rewire the four write actions

Replace the bodies of these functions so they write to Firestore. Each change is small.

**Save a product** (`saveProduct`):
```js
async function saveProduct(id){
  const name=val('pr-name'); if(!name){toast('Enter a name','bad');return;}
  const data={ name, cat:val('pr-cat'), price:Number(val('pr-price'))||0,
    stock:Number(val('pr-stock'))||0, desc:val('pr-desc'),
    img:document.getElementById('pr-img').value };
  if(id){ await fbUpdate('products', id, data); toast('Product updated','ok'); }
  else  { await fbAdd('products', data);        toast('Product added','ok'); }
  closeModal(); // the live listener re-renders automatically
}
```

**Delete a product** (`confirmDelete`):
```js
async function confirmDelete(id){ await fbDelete('products', id); closeModal(); toast('Product deleted','ok'); }
```

**Place an order** (inside `completeOrder`, replace the localStorage lines):
```js
async function completeOrder(name,phone,total){
  const order={ ref:'LY'+Math.floor(100000+Math.random()*899999),
    name, phone, area:val('ck-area'), notes:val('ck-notes'),
    items:JSON.parse(JSON.stringify(cart)), total,
    status:'pending', date:new Date().toISOString(),
    mpesaCode:'Q'+Math.random().toString(36).slice(2,10).toUpperCase() };
  await fbAdd('orders', order);
  // reduce stock per item
  for(const c of cart){
    const p=cache.products.find(x=>x.id===c.id);
    if(p) await fbUpdate('products', p.id, { stock:Math.max(0,p.stock-c.qty) });
  }
  cart=[]; localStorage.setItem('loyalty_cart','[]'); updateCartUI(); closeCart();
  order.id='new'; showReceipt(order);
}
```

**Submit a booking** (`submitBooking`), replace the save lines:
```js
async function submitBooking(){
  const name=val('bk-name'), phone=val('bk-phone');
  if(!name||!phone){toast('Enter name and phone','bad');return;}
  const booking={ ref:'BKG'+Math.floor(10000+Math.random()*89999),
    name, phone, company:val('bk-company'), service:val('bk-service'),
    date:val('bk-date'), msg:val('bk-msg'),
    status:'pending', created:new Date().toISOString() };
  await fbAdd('bookings', booking);
  openModal('Request Received', `... your existing success HTML ...`);
}
```

**Approve / reject** (`setOrderStatus`, `setBooking`):
```js
async function setOrderStatus(id,status){ await fbUpdate('orders', id, {status}); closeModal(); toast('Order '+status,'ok'); }
async function setBooking(id,status){ await fbUpdate('bookings', id, {status}); closeModal(); toast('Request '+status,'ok'); }
```

> The cart itself can stay in `localStorage` — it's personal to each shopper and doesn't need to be shared. Only products, orders, and bookings move to Firestore.

---

## Part 5 — Security rules with roles (your real protection)

This is the most important part, and it's your area. These rules read each
user's **role** from the `users` collection and enforce it on the server —
so nobody can promote themselves to admin by editing the browser.

Open **Firestore Database → Rules**, replace everything with this, and click **Publish**:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Look up the signed-in user's role from their users document.
    function role() {
      return get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role;
    }
    function isStaff() { return request.auth != null && (role() == 'staff' || role() == 'admin'); }
    function isAdmin() { return request.auth != null && role() == 'admin'; }

    // USERS: on sign-up you may create ONLY your own doc, and ONLY as a
    // customer — you cannot make yourself staff/admin. You may read your
    // own doc. Only an admin can read everyone or change anyone's role.
    match /users/{uid} {
      allow read:   if request.auth != null && (request.auth.uid == uid || isAdmin());
      allow create: if request.auth.uid == uid
                    && request.resource.data.role == 'customer';
      allow update: if isAdmin();     // only admins assign roles
      allow delete: if isAdmin();
    }

    // PRODUCTS: shop is public to read; only staff/admin may change.
    match /products/{id} {
      allow read:  if true;
      allow write: if isStaff();
    }

    // ORDERS: a customer may create their order at checkout.
    // Only staff/admin may read all orders or change their status.
    match /orders/{id} {
      allow create: if true;
      allow read, update, delete: if isStaff();
    }

    // BOOKINGS: same pattern as orders.
    match /bookings/{id} {
      allow create: if true;
      allow read, update, delete: if isStaff();
    }
  }
}
```

What this enforces:
1. A new user can only ever create themselves as a **customer**. Self-promotion to admin is impossible from the browser — the server rejects it.
2. Only an **admin** can change roles, and only via the Team tab.
3. Only **staff/admin** can see orders, approve things, or edit products.
4. Customers can browse and buy, but cannot read other people's orders or wipe your catalog.

For **Storage** rules (product image uploads), open **Storage → Rules**:
```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /products/{file} {
      allow read: if true;                 // images are public
      allow write: if request.auth != null; // only staff upload
    }
  }
}
```

---

## Part 6 — Register, login & roles (no passwords in your code)

The app already has the full role system: anyone registers, the first user
becomes admin, and admins assign roles in the **Team** tab. Here you connect
that same flow to real Firebase Auth. Passwords are handled by Firebase —
you never store or even see them.

The app keeps users in `AUTH` (localStorage) for the demo. In production,
identity lives in **Firebase Auth** and the role lives in a **`users`**
Firestore doc. Replace the `AUTH` helpers and the four auth functions:

### 6a. Register — creates the login AND the role doc

```js
async function registerUser(){
  const name=val('au-name'), email=val('au-email').toLowerCase(), pass=val('au-pass');
  if(!name||!email||!pass){toast('Fill in all fields','bad');return;}
  if(pass.length<6){toast('Password must be 6+ characters','bad');return;}
  const { auth, fdb, doc, setDoc } = window.FB;
  try{
    const cred = await window.FB.createUserWithEmailAndPassword(auth, email, pass);
    // New users are always 'customer'. The security rules ENFORCE this —
    // even if someone tampered with the code, the server would reject
    // any attempt to self-assign staff/admin.
    await setDoc(doc(fdb,'users',cred.user.uid), { name, email, role:'customer', created:new Date().toISOString() });
    toast('Account created — you are logged in.','ok');
  }catch(e){ toast(e.code==='auth/email-already-in-use'?'Email already registered':'Could not register','bad'); }
}
```
> Add `createUserWithEmailAndPassword` to the imports and the `window.FB` object in Part 3.

**Bootstrapping the first admin (one time):** because the rules force every
new signup to `customer`, you promote yourself once by hand:
1. Register your own account in the app.
2. Open **Firestore → users → (your document)**.
3. Change the `role` field from `customer` to `admin`. Save.

That's it — you're now admin, and from then on you assign all other roles
inside the app's Team tab. This is the secure way: the very first admin is
set by someone with database access, never by the browser.

### 6b. Login — then load the role

```js
async function loginUser(){
  const email=val('au-email').toLowerCase(), pass=val('au-pass');
  const { auth, fdb, doc } = window.FB;
  try{
    const cred = await window.FB.signInWithEmailAndPassword(auth, email, pass);
    const snap = await window.FB.getDoc(doc(fdb,'users',cred.user.uid));
    window.currentUser = { uid:cred.user.uid, ...snap.data() };
    updateAccountUI();
    toast('Welcome back','ok');
    if(window.currentUser.role==='admin' || window.currentUser.role==='staff') go('admin');
  }catch(e){ toast('Wrong email or password','bad'); }
}
async function logoutUser(){ await window.FB.signOut(window.FB.auth); window.currentUser=null; updateAccountUI(); go('home'); }
```
> Add `getDoc` to the imports/`window.FB` too. Replace the demo `AUTH.current()`
> calls with `window.currentUser` throughout (it holds `{uid, name, email, role}`).

### 6c. Keep the session on refresh

```js
window.addEventListener('firebase-ready', () => {
  window.FB.onAuthStateChanged(window.FB.auth, async user => {
    if(user){
      const snap = await window.FB.getDoc(window.FB.doc(window.FB.fdb,'users',user.uid));
      window.currentUser = { uid:user.uid, ...snap.data() };
    } else { window.currentUser = null; }
    updateAccountUI();
    if(!document.getElementById('admin-view').classList.contains('hidden')) renderAdmin();
  });
});
```

### 6d. The Team tab — assign roles

Your `teamHtml` and `setRole` already work. Point them at Firestore:

```js
// load users once when the Team tab opens
async function loadTeam(){
  const { fdb, getDocs, collection } = window.FB;
  const snap = await getDocs(collection(fdb,'users'));
  window.teamCache = snap.docs.map(d => ({ uid:d.id, ...d.data() }));
  renderAdmin();
}
async function setRole(uid, role){
  const admins = (window.teamCache||[]).filter(u=>u.role==='admin');
  const target = (window.teamCache||[]).find(u=>u.uid===uid);
  if(target.role==='admin' && role!=='admin' && admins.length<=1){ toast('You cannot remove the last admin','bad'); return; }
  await window.FB.updateDoc(window.FB.doc(window.FB.fdb,'users',uid), { role });  // rules allow only admins
  toast('Role updated','ok');
  loadTeam();
}
```
Have `teamHtml` render from `window.teamCache`, and call `loadTeam()` when the
admin opens the Team tab.

The result is exactly the demo behaviour, now enforced by the server: anyone
can register, the first person you promote by hand becomes admin, and every
other role is assigned inside the app — with no password anywhere in your code.

---

## Part 7 — Product images in Storage (optional but better)

Right now images are stored as base64 text inside the product record. That works but bloats the database. To store them properly, change `previewImg` to upload to Storage and save the returned URL:

```js
async function previewImg(e){
  const file=e.target.files[0]; if(!file) return;
  if(file.size>3000000){toast('Image too large (max 3MB)','bad');return;}
  toast('Uploading image...');
  const { storage, sRef, uploadBytes, getDownloadURL } = window.FB;
  const path = 'products/'+Date.now()+'_'+file.name.replace(/\s+/g,'_');
  const r = sRef(storage, path);
  await uploadBytes(r, file);
  const url = await getDownloadURL(r);
  document.getElementById('pr-img').value = url;
  document.getElementById('imgPrev').innerHTML =
    `<img src="${url}" style="width:100px;height:100px;object-fit:cover;border-radius:10px">`;
  toast('Image uploaded','ok');
}
```

---

## Part 8 — Test, then deploy

1. **Test locally the right way.** Firebase Auth and modules need a server, not `file://`. In the folder, run:
   ```
   python3 -m http.server 5500
   ```
   Then open `http://localhost:5500`. Log into admin with your real account, add a product, and confirm it appears. Open the same URL in a second browser window — the product should appear there too, live.

2. **Deploy.** Drag the folder onto **app.netlify.com/drop**.

3. **Authorise your Netlify domain.** In Firebase → **Authentication → Settings → Authorized domains**, add your `*.netlify.app` address. Without this, admin login fails on the live site. (This is the #1 thing people forget.)

---

## What's still not done after this

Firebase does **not** handle M-Pesa. Real payments still need Safaricom's Daraja API called from a **backend function** (Netlify Functions or Firebase Cloud Functions), because your Daraja secret must never sit in the browser. That's a separate step — you've already done STK Push in sandbox, so it's familiar. The checkout in this app stays a simulation until you wire that.

---

## Quick reference — collection shapes

**products**: `{ name, cat, price, stock, desc, img }`
**orders**: `{ ref, name, phone, area, notes, items[], total, status, date, mpesaCode }`
**bookings**: `{ ref, name, phone, company, service, date, msg, status, created }`

`status` values: `pending`, `approved`, `rejected`.
