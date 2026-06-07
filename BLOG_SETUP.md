# Blog / Login / Admin Setup (Firebase)

The new `blog.html` page needs a backend for accounts, email verification, and
storing posts. Since GitHub Pages only serves static files, we use **Firebase**
(Google's free-tier backend-as-a-service) — same idea as the Cloudflare Worker
relay for chat, just a different provider. Free tier is plenty for this site.

## 1. Create a Firebase Project

1. Go to https://console.firebase.google.com and click **Add project**.
2. Name it whatever you want (e.g. `definitely-not-propaganda`). Google
   Analytics is optional — you can turn it off.
3. Once created, click the **Web** icon (`</>`) to register a web app. You
   don't need Firebase Hosting — just register the app.
4. Firebase will show you a `firebaseConfig` object that looks like:
   ```js
   const firebaseConfig = {
     apiKey: "AIza...",
     authDomain: "definitely-not-propaganda.firebaseapp.com",
     projectId: "definitely-not-propaganda",
     storageBucket: "definitely-not-propaganda.appspot.com",
     messagingSenderId: "123456789",
     appId: "1:123456789:web:abcdef",
   };
   ```
   Copy these six values.

## 2. Paste Your Config into `blog.html`

Open `blog.html` and find the `FIREBASE_CONFIG` block near the bottom
(search for `CONFIGURE ME`). Replace the placeholder values with your real
ones from step 1.

Also set `ADMIN_EMAIL` to the email address of the account that should be
allowed to publish blog posts (your editorial/admin account — see step 5).

> Note: this file is public (it's a static site), so your Firebase config
> values will be visible in the page source. That's normal and expected —
> Firebase web config is not a secret. Security is enforced by the Firestore
> rules in step 4, not by hiding the config.

## 3. Enable Email/Password Authentication

1. In the Firebase Console, go to **Build → Authentication → Get started**.
2. Under **Sign-in method**, enable **Email/Password**.
3. (Optional but recommended) Go to **Authentication → Templates** and
   customize the "Email address verification" template — you can change the
   sender name, subject, and message. This is the email users get when they
   sign up.

## 4. Enable Firestore and Set Security Rules

1. Go to **Build → Firestore Database → Create database**. Pick a region
   close to your audience and start in **production mode**.
2. Go to the **Rules** tab and replace the default rules with:

   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /posts/{postId} {
         allow read: if true;
         allow write: if request.auth != null
                      && request.auth.token.email == 'YOUR_ADMIN_EMAIL@example.com'
                      && request.auth.token.email_verified == true;
       }
     }
   }
   ```

   Replace `YOUR_ADMIN_EMAIL@example.com` with the **exact same** address you
   set as `ADMIN_EMAIL` in `blog.html`. This is what actually enforces "only
   the admin account can publish" — the JS check in the page is just for
   showing/hiding the composer UI; this rule is what Firebase enforces
   server-side no matter what someone does in their browser.
3. Click **Publish**.

## 5. Create Your Admin Account

There's no separate "admin signup" — the admin account is just a normal
account whose email matches `ADMIN_EMAIL`:

1. Open `blog.html` on the live site, click **Sign Up**, and create an
   account using the exact email address you set as `ADMIN_EMAIL`.
2. Check that inbox for the verification email Firebase sends and click the
   link.
3. Log back in on the blog page. You should now see an
   **"★ Editorial / Admin Account — verified"** badge and a **New Post**
   composer appear below the login box. Anyone else who signs up will just
   see "Verified reader" and won't be able to publish (enforced by the rule
   in step 4).

## 6. Verify It Works

- Sign up with a second, non-admin email — confirm you get a verification
  email and do **not** see the composer after verifying.
- Publish a test post from the admin account — confirm it appears in the
  list immediately (Firestore updates live, no refresh needed).
- Try deleting it with the **Delete** button that appears on posts when
  you're logged in as admin.

## Notes

- Posts are stored in the `posts` collection in Firestore: `title`, `body`,
  `author`, `authorName`, `createdAt`.
- Reading posts is public — no login required to browse the blog.
- Only one admin account is supported by these rules. If you want multiple
  editors later, change the rule to check a list of emails or a Firestore
  `roles` collection instead of a single hardcoded address.
- Firebase's free "Spark" plan covers Authentication and Firestore at this
  scale with no billing setup required.
