# Pod Booking System — Setup Guide

A drag-and-drop tent booking system for campsites. Campers get assigned to tents across pods. Works as a single HTML file — deploy on GitHub Pages, Netlify, or any static host.

---

## Quick Start (Demo Mode)

1. Open `index.html` in your browser
2. Click **"Try demo mode"** on the login screen
3. Drag campers from the sidebar onto tent slots
4. Use the Admin panel to add/remove campers

No backend or accounts needed for demo mode. Data only lives in your browser tab.

---

## Full Setup — Firebase (Free Tier)

Firebase gives you passwordless email login and real-time data sync. The free "Spark" plan supports up to 50,000 reads/day and 20,000 writes/day — more than enough for a campsite.

### Step 1: Create a Firebase Project

1. Go to [Firebase Console](https://console.firebase.google.com/)
2. Click **Add project** → name it (e.g. "pod-booking") → click through the steps → create
3. When it asks about Google Analytics, you can skip it (not needed)

### Step 2: Register a Web App

1. In your new project, go to **Project Settings** (gear icon) → **General**
2. Scroll down to **Your apps** → click the web icon `</>`
3. Give it a nickname (e.g. "pod-booking-web") → click **Register app**
4. You'll see a config object like this — copy it:

```js
const firebaseConfig = {
  apiKey:            "AIzaSy...",
  authDomain:        "your-project.firebaseapp.com",
  projectId:         "your-project",
  storageBucket:     "your-project.appspot.com",
  messagingSenderId: "123456789",
  appId:             "1:123456789:web:abc123"
};
```

### Step 3: Paste Your Config

Open `index.html` and find the `firebaseConfig` block (around line 622). Replace the placeholder values with the ones you copied:

```js
const firebaseConfig = {
  apiKey:            "AIzaSy...",        // your real key
  authDomain:        "your-project.firebaseapp.com",
  projectId:         "your-project",
  storageBucket:     "your-project.appspot.com",
  messagingSenderId: "123456789",
  appId:             "1:123456789:web:abc123"
};
```

> **Is the API key safe to have in the HTML?** Yes — Firebase API keys are public identifiers, not secrets. They just tell Firebase which project to connect to. Security comes from the rules you set up in Steps 5 and 6 below.

### Step 4: Enable Email Link Sign-in

1. In Firebase Console → **Authentication** (left sidebar) → **Get started**
2. Go to the **Sign-in method** tab
3. Click **Email/Password** → enable it
4. Toggle on **Email link (passwordless sign-in)** → save
5. Go to the **Settings** tab → **Authorized domains**
6. Add your domain (e.g. `yourname.github.io`) — localhost is already there for testing

### Step 5: Set Up Firestore with Security Rules

This is the most important security step. Firestore rules control who can read and write data — without them, anyone could modify your bookings.

#### Create the database

1. In Firebase Console → **Firestore Database** (left sidebar) → **Create database**
2. Choose **Start in production mode** (we'll set proper rules next)
3. Pick a region close to your users (e.g. `europe-west2` for UK, `us-central1` for US)

#### Set the security rules

1. Go to **Firestore Database** → **Rules** tab
2. Delete the default rules and paste this:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Pod booking site data
    match /sites/{siteId} {

      // Anyone signed in can read the site map
      allow read: if request.auth != null;

      // Only the admin can write (add campers, assign tents, clear all)
      allow write: if request.auth != null
                   && request.auth.token.email == 'YOUR_ADMIN_EMAIL@gmail.com';
    }

    // Block everything else by default
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

3. **Replace `YOUR_ADMIN_EMAIL@gmail.com`** with the actual email address you want as admin
4. Click **Publish**

#### What these rules do

| Rule | Effect |
|------|--------|
| `allow read: if request.auth != null` | Any signed-in user can see the tent map |
| `allow write: if ... email == 'admin@...'` | Only the admin email can make changes |
| `match /{document=**} allow ... if false` | Everything else is blocked |

#### If you want campers to self-assign

If you want campers (not just the admin) to be able to drag themselves onto tents, use these rules instead:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /sites/{siteId} {
      // Anyone signed in can read
      allow read: if request.auth != null;

      // Anyone signed in can update (assign/unassign themselves)
      allow update: if request.auth != null;

      // Only admin can create the initial document or delete it
      allow create, delete: if request.auth != null
                            && request.auth.token.email == 'YOUR_ADMIN_EMAIL@gmail.com';
    }

    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

> **Note:** Firestore rules can't see _which fields_ changed in an update (without complex diffing), so `allow update` for all authenticated users means a signed-in camper could technically reassign anyone. For a small trusted group this is fine. For stricter control, keep the admin-only write rules.

### Step 6: Set Your Admin Email in the Code

In `index.html`, find `ADMIN_EMAIL` (around line 641) and change it to match the email in your Firestore rules:

```js
const ADMIN_EMAIL = "your-email@gmail.com";
```

This must be the **same email** you put in the Firestore rules. The code uses it to show/hide the admin panel in the UI. The Firestore rules enforce it server-side.

### Step 7: Restrict Your API Key (Optional but Recommended)

This limits your API key so it only works from your website's domain:

1. Go to [Google Cloud Console → Credentials](https://console.cloud.google.com/apis/credentials)
2. Make sure you're in the same project as your Firebase project (check the project dropdown at the top)
3. Find the **Browser key** (auto created by Firebase) → click it
4. Under **Application restrictions**, select **HTTP referrers (websites)**
5. Add your domain:
   - `https://yourname.github.io/*` (GitHub Pages)
   - `https://your-site.netlify.app/*` (Netlify)
   - `http://localhost:*` (for local testing — remove this before going live)
6. Click **Save**

This means even if someone copies your API key, it won't work from their domain.

### Step 8: Deploy

**GitHub Pages (free):**
1. Push your code to a GitHub repository
2. Go to repo **Settings** → **Pages**
3. Under **Source**, select your branch (e.g. `main`) and click **Save**
4. Your site will be live at `https://yourname.github.io/repo-name/`
5. Make sure this domain is in your Firebase **Authorized domains** (Step 4)

**Netlify (free):**
1. Go to [netlify.com](https://www.netlify.com/) → sign in
2. Drag your project folder onto the deploy area — done
3. Add your Netlify domain to Firebase **Authorized domains**

**Any static host:**
It's a single HTML file with no build step. Upload it anywhere that serves static files.

---

## Customising the Layout

In `index.html`, find these constants (around line 646):

```js
const PODS = 6;           // number of pods
const TENTS_PER_POD = 12; // tents in each pod
```

Change these to match your campsite layout. The grid auto-adjusts.

---

## How It Works

| Feature | Details |
|---------|---------|
| **Layout** | Pods × tents (default 6×12, configurable) |
| **Tent types** | Solo (one person, full tent) or Sharing (up to 2 people) |
| **Booking** | Drag a camper from the sidebar onto a tent slot |
| **Admin** | Can add/remove campers, reassign tents, clear all |
| **Camper** | Can see the map (editing is controlled by Firestore rules) |
| **Sync** | Real-time via Firestore — all users see changes instantly |
| **Auth** | Passwordless email magic links via Firebase |

---

## Troubleshooting

**"Failed to send link"** — Check that:
- Your `firebaseConfig` values are correct
- Email link sign-in is enabled in Firebase Console
- Your domain is in the Authorized domains list

**"Failed to save"** — Check that:
- Firestore is created and rules are published
- The admin email in the rules matches the one you're signed in with
- You're not still in the default "deny all" production rules

**Changes don't sync** — Make sure:
- You're not in demo mode (demo mode is local-only)
- Firestore is in the same project as your auth
- Check the browser console (F12) for error messages

**Login link doesn't arrive** — Check:
- Your spam/junk folder
- That the email is spelled correctly
- Firebase free tier allows 10,000 auth emails/month — unlikely to hit this, but check if testing a lot

---

## Security Checklist

Before sharing the site with real users, confirm:

- [ ] Firebase config values are pasted in `index.html`
- [ ] Email link sign-in is enabled
- [ ] Firestore rules are published (not still on test/default rules)
- [ ] Admin email in Firestore rules matches `ADMIN_EMAIL` in the code
- [ ] Your deploy domain is in Firebase Authorized domains
- [ ] API key is restricted to your domain (Step 7)
- [ ] You're deploying over HTTPS (GitHub Pages and Netlify do this automatically)
