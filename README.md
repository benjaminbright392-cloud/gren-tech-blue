# Green Tech Empire website

Plain HTML, CSS and JavaScript. No build step, no frameworks.

## Files
- index.html: single-page shop (hero, catalog, verify payment, about, contact)
- admin.html: owner-only page to list pending payments and mark them verified
- css/style.css: dark green theme with yellow-green accent
- js/config.js: Firebase config and ADMIN_EMAIL (placeholders)
- js/auth.js: email/password signup, login, logout
- js/orders.js: payment verification form checks, SHA-256 hash, Firestore writes
- js/main.js: product catalog and the verify-payment form
- assets/: product photos

## 1. Firebase setup (free tier)
1. Go to https://console.firebase.google.com and create a project.
2. Add a Web app (the </> icon) and copy the firebaseConfig values into js/config.js.
3. Build > Authentication > Get started > enable Email/Password.
4. Build > Firestore Database > Create database (production mode).
5. Set your owner email in js/config.js (ADMIN_EMAIL). Sign up with that email on the site first.
6. Paste the Firestore rules below in Firestore > Rules, replacing owner@example.com with the same email as ADMIN_EMAIL, then Publish.

## 2. Firestore rules (only the signed-in user reads their own orders; only the admin verifies)

    rules_version = '2';
    service cloud.firestore {
      match /databases/{database}/documents {
        function isAdmin() {
          return request.auth != null
            && request.auth.token.email == "owner@example.com"
            && request.auth.token.email_verified == true;
        }
        match /orders/{orderId} {
          // Users create orders only for themselves, always starting as pending.
          allow create: if request.auth != null
            && request.resource.data.uid == request.auth.uid
            && request.resource.data.status == "pending"
            && request.resource.data.keys().hasOnly(["uid","name","email","productId","productName","amount","reference","hash","status","createdAt"]);
          // Users read only their own orders; admin reads all (for the pending list).
          allow read: if (request.auth != null && resource.data.uid == request.auth.uid) || isAdmin();
          // Only the admin can change an order, and only its status fields.
          allow update: if isAdmin()
            && request.resource.data.diff(resource.data).affectedKeys().hasOnly(["status","verifiedAt","verifiedBy"])
            && request.resource.data.status == "verified";
          allow delete: if false;
        }
      }
    }

## 3. Payment flow
1. Visitor signs up or logs in, picks a product and sends a bank transfer to the shop account (details come from the owner, not the website).
2. On the site, the visitor enters name, email, product, amount in naira and the transfer reference or trace ID.
3. The site checks formats (name, email, whole-number amount matching the product price, reference 8 to 30 letters or digits), computes a SHA-256 hash of the submission and saves the order as pending under the visitor's account.
4. The visitor sees the order as Pending in "Your orders".
5. The owner checks the transfer in their bank app, opens admin.html, and clicks Mark verified. The order becomes Verified for the visitor.

Client-side checks only catch typos and bad entries. They cannot prove a payment happened. The owner's bank confirmation is the real check. No card numbers or payment card data are collected anywhere on the site.

## 4. Set prices
Each product in js/main.js has `price: null`. Replace null with the price in naira (for example 450000). Products with no price cannot be submitted for verification.

## 5. Run locally
Open index.html through a local server (for example `python3 -m http.server` in this folder), because ES modules do not load from file://.

## 6. Hosting
Upload the folder to any static host (Firebase Hosting, GitHub Pages, Netlify).
