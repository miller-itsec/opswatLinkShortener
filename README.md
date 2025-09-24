# OPSWAT Link Shortener

A secure, self-hosted link shortener designed for internal use within
OPSWAT.\
This application allows authorized administrators to create short,
shareable **opsw.at** links for URLs from approved domains
(**filescan.io**, **opswat.com**, **metadefender.com**).

The application is built as a single-page app using **vanilla
JavaScript** and **Tailwind CSS**, with **Google Firebase** providing
the backend for authentication and database services.

------------------------------------------------------------------------

## Features

-   **Domain Whitelisting**: Only allows URLs from `filescan.io`,
    `opswat.com`, and `metadefender.com` to be shortened.
-   **Secure Admin Access**: Link creation is restricted to authorized
    users who sign in with their Google accounts. An admin whitelist is
    managed in the Firestore database.
-   **Public Link Redirection**: Anyone with a shortlink can use it; the
    redirection logic is public.
-   **Modern UI**: A clean, dark-themed interface built with Tailwind
    CSS that matches the OPSWAT branding.
-   **Serverless Backend**: Powered entirely by Firebase (Authentication
    and Firestore), making it scalable and easy to manage.
-   **Self-Hosted**: Designed to be hosted on your own server using
    Nginx.

------------------------------------------------------------------------

## Technology Stack

-   **Frontend**: HTML, Tailwind CSS, Vanilla JavaScript\
-   **Backend**: Google Firebase\
-   **Authentication**: Google Sign-In\
-   **Database**: Cloud Firestore

------------------------------------------------------------------------

## Setup and Configuration Guide

Follow these steps to set up and deploy your own instance of the OPSWAT
Link Shortener.

### Step 1: Create a Firebase Project

1.  Go to the [Firebase Console](https://console.firebase.google.com/).
2.  Click **Create a project** and follow the on-screen instructions.
    Give it a name like `opswat-link-shortener`.
3.  Navigate to **Project Settings** (gear icon in the top-left).
4.  In the **Your apps** card, click the web icon (`</>`) to register a
    new web app.
5.  Give the app a nickname and click **Register app**.
6.  Firebase will provide you with a configuration object. Copy this
    object. This is your `firebaseConfig`.

``` javascript
// Your config will look like this
const firebaseConfig = {
  apiKey: "AIza...",
  authDomain: "your-project-id.firebaseapp.com",
  projectId: "your-project-id",
  storageBucket: "your-project-id.appspot.com",
  messagingSenderId: "...",
  appId: "...",
  measurementId: "..."
};
```

### Step 2: Configure Firebase Authentication

1.  In the Firebase Console, go to **Authentication** → **Get started**.
2.  Select the **Sign-in method** tab.
3.  Click on **Google**, enable it, select a project support email, and
    click **Save**.
4.  Scroll down to **Authorized domains**, click **Add domain**, and
    add:
    -   `localhost`
    -   `opsw.at` (or your final domain)

### Step 3: Configure Firestore Database

1.  In the Firebase Console, go to **Firestore Database** → **Create
    database**.
2.  Select **Start in production mode**, click **Next**.
3.  Choose a Cloud Firestore location (default is fine) and click
    **Enable**.

### Step 4: Set Up Security Rules

This step secures your application.

1.  In **Firestore Database**, click the **Rules** tab.
2.  Replace the existing rules with:

``` firestore
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Rule for the admins list
    match /admins/{userId} {
      allow read: if request.auth != null && request.auth.uid == userId;
    }

    // Rule for the links collection
    match /artifacts/{appId}/public/data/links/{linkId} {
      allow read: if true; // Anyone can read shortlinks
      allow write: if request.auth != null &&
        exists(/databases/$(database)/documents/admins/$(request.auth.uid));
    }
  }
}
```

Click **Publish**.

### Step 5: Add the First Admin User

1.  Open your deployed application and sign in once using the Google
    account to be the first administrator.\
    (You may see an "unauthorized" error --- that's expected.)
2.  Go to **Authentication** → **Users**, copy the **User UID** of that
    account.
3.  In **Firestore Database**, click **+ Start collection**:
    -   Collection ID: `admins`
    -   Document ID: *paste the User UID*
    -   Add a field (e.g., `email`) and click **Save**.

Repeat for additional administrators.

### Step 6: Configure the Application

1.  Open the `index.html` file.
2.  Find the `<script>` block at the bottom and replace
    `window.__firebase_config`:

``` html
<script>
  window.__firebase_config = {
    apiKey: "AIza...", // Your real config
    authDomain: "...",
    projectId: "...",
    storageBucket: "...",
    messagingSenderId: "...",
    appId: "...",
    measurementId: "..."
  };
  window.__app_id = 'opswat-link-shortener';
</script>
```

------------------------------------------------------------------------

## Deployment

Place the finalized `index.html` in your web server root (e.g.,
`/var/www/opswat`).\
Use this **Nginx configuration**:

``` nginx
# HTTP -> HTTPS redirect
server {
    listen 80;
    listen [::]:80;
    server_name opsw.at www.opsw.at;
    return 301 https://$host$request_uri;
}

# HTTPS server
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name opsw.at www.opsw.at;

    root /var/www/opswat;
    index index.html;

    ssl_certificate     /path/to/your/fullchain.pem;
    ssl_certificate_key /path/to/your/privkey.pem;

    # Handle SPA routing
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

------------------------------------------------------------------------

## Usage

-   **Administrators**: Visit `https://opsw.at`, sign in with Google,
    and create shortlinks.\
-   **Users**: Visiting a shortlink like `https://opsw.at/XyZ123` will
    redirect to the original URL.
